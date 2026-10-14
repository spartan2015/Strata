# Image generation (Qwen-Image-2.1-Turbo)

Strata's server can also make images: `POST /v1/images/generations`, OpenAI's images API. The model is
[Qwen-Image-2.1-Turbo](https://huggingface.co/Qwen/Qwen-Image-2.1-Turbo) (8 steps, CFG 1) in
[Unsloth's GGUF quants](https://huggingface.co/unsloth/Qwen-Image-2.1-Turbo-GGUF), run by
[stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp) (`sd-cli`), which setup compiles for your GPU.
It is optional and off unless you ask for it.

The model is under the **Qwen Research License**, not Apache 2.0: read
[its license](https://huggingface.co/Qwen/Qwen-Image-2.1-Turbo/blob/main/LICENSE) before you use the images
commercially.

## Set it up

    ./setup.sh --setup --images qwen-image          (Linux)
    START-HERE.bat --setup --images qwen-image      (Windows)

or, for an install that is already set up, without running the whole setup again:

    .venv/bin/python tools/image/install.py --config strata-<model>.json

Setup compiles `sd-cli` (stable-diffusion.cpp at a pinned commit) into `engine-image/`, downloads three files into
the data folder's `models/qwen-image-2.1-turbo/` and checks each against its SHA-256, and writes an `"images"` section
into the model's run config.

| piece | file | size |
|---|---|---|
| denoiser (by GPU memory) | `qwen-image-2.1-turbo-Q4_K_M.gguf` up to 8 GB, `Q6_K` up to 12 GB, `Q8_0` above | 4.2 / 6.3 / 7.6 GB |
| text encoder | `Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf` | 5.1 GB |
| VAE | `qwen_image_2.1_vae_bf16.safetensors` | 0.7 GB |

Options: `--image-quant Q4_K_M|Q6_K|Q8_0` (a bigger one than recommended is kept, with a warning),
`--image-backend cuda|hip|vulkan|cpu`, `--image-gpu-policy auto|keep|shrink|unload`, `--images no` (takes the section
out of the config).

Backends: CUDA on NVIDIA (needs the CUDA Toolkit), HIP on AMD cards with their own memory (Linux, the same ROCm as the
engine), Vulkan on AMD APUs/iGPUs (Radeon 780M, Strix Point/Halo), Intel Arc and AMD under Windows. A Linux PC without
the Vulkan development files gets the LunarG Vulkan SDK unpacked into `third_party/` (no sudo); on Windows install the
Vulkan SDK first. On Vulkan setup adds `--vae-tiling --vae-tile-overlap 0.25`: an untiled 1024x1024 VAE decode on a Radeon 780M is one
GPU job long enough to trip the kernel driver's job timeout (no image), and a quarter's overlap decodes in half the
time of sd.cpp's default half overlap.

## Use it

    curl http://127.0.0.1:8080/v1/images/generations -H "Content-Type: application/json" \
      -d '{"prompt": "a cartoon sloth mascot waving, flat vector illustration", "size": "1024x1024", "seed": 42}'

| field | |
|---|---|
| `prompt` | the description (required) |
| `size` | `WIDTHxHEIGHT`, each side a multiple of 32 from 256 to 2048 (`max_side`); default and `auto`: `1024x1024` |
| `n` | images, 1 to 4 (`max_n`); image *i* uses seed + *i* |
| `response_format` | `b64_json` (default) or `url` (`/v1/images/files/<random name>.png`; the server keeps the newest 64) |
| `seed` | Strata's: a whole number for a repeatable image (default: random; the answer says which) |
| `steps` | Strata's: 8 (Turbo's schedule). Another count uses stable-diffusion.cpp's own schedule, which Turbo was not made for |
| `image` | a PNG to build on instead of starting from noise: the name in a `url` answer (`a1b2….png`), that answer's whole link, or the PNG itself as base64 (a `data:image/png;base64,` link works). 32 MiB decoded at most |
| `strength` | with `image`: how much of it to re-draw, `0` (keep it) to `1` (ignore it). Default `0.75`, sd-cli's own. Below `1` the picture keeps its composition, so the same prompt lands in the same place |

The answer is OpenAI's shape (`created`, `data[].b64_json` or `data[].url`, `revised_prompt` = the prompt) plus
`data[].seed` and a `strata` object with the seconds, steps and backend. The settings are the model card's for
stable-diffusion.cpp: 8 steps, CFG 1, euler, Turbo's own sigmas, `--diffusion-fa`. An image from the API is the PNG
plain `sd-cli` writes with the same arguments, byte for byte.

## Build on an image you already have

Ask for `response_format: "url"`, keep the link, and send it back as `image` with the next prompt. The server keeps
the PNG under the config's `images.out_dir` (the newest `keep_files`, 64 by default; set `out_dir` to a folder of your
own and they survive a restart — without it they go with the temporary directory).

    curl http://127.0.0.1:8080/v1/images/generations -H "Content-Type: application/json" \
      -d '{"prompt": "a red fox cub asleep in ferns", "seed": 7, "response_format": "url"}'
    # -> data[0].url = /v1/images/files/55e220c69b788814e9a613fab8839a4f.png

    curl http://127.0.0.1:8080/v1/images/generations -H "Content-Type: application/json" \
      -d '{"prompt": "the same fox cub, now with a red kite in its paws", "seed": 7, "strength": 0.4, \
           "image": "/v1/images/files/55e220c69b788814e9a613fab8839a4f.png"}'

The second call re-draws 40% of the first, so the cub, the ferns and the light stay where they were and the kite
appears. `strata.strength` says what it used. Any PNG you already own works the same way as base64, so an image from
a previous session, or one you painted over by hand, can be the base.

Measured on the APU at 512x512, 8 steps: `generate_image` took 15.01 s for a text-to-image pass and 12.53 s for an
img2img pass at `--strength 0.5` — re-drawing part of an image is *faster* than drawing all of it, because strength
skips the steps whose noise it keeps. Building on an image costs nothing over drawing again. The same run checked the
command the server builds: with no `image` field its output was byte-identical to plain `sd-cli` (same md5), and an
img2img pass that kept the server's own `--sigmas` still came out different from the same prompt with no init image,
so `--strength` is honoured on top of Turbo's schedule rather than overridden by it.

## Sharing the GPU with the text model

One job at a time: an image job takes the same turn a chat request takes, so a chat request waits for the image and
the other way round. What the text model does meanwhile is the config's `images.gpu_policy`:

| policy | the text model during an image job |
|---|---|
| `auto` (default) | `keep` when the image job fits next to it (an APU always), else `shrink` when the engine was started with `--vram-elastic`, else `unload` |
| `keep` | stays loaded and idle |
| `shrink` | gives VRAM from its expert cache (as `POST /v1/vram`) and takes it back afterwards; the conversation stays |
| `unload` | its conversation is saved to a file, the engine is stopped, and the next start restores the conversation (`"reload": "eager"` starts it again right after the image; default `lazy`: with the next chat request) |

Each image runs its own `sd-cli` process, so no image memory stays on the GPU between images. Loading the three
files takes a few seconds per image from the page cache.

## Speed

Seconds per image through the API, Q8_0, 8 steps (one `sd-cli` process per image, loading included):

| GPU | backend | 1024x1024 | 512x512 |
|---|---|---|---|
| RTX A4000 16 GB | CUDA | 27 | 8.8 |
| Radeon 780M (iGPU) | HIP | 131 | 29 |
| Radeon 780M (iGPU) | Vulkan | 179 | 43 |

On the 780M HIP is faster but runs through the ROCm path whose queue evictions reset this GPU on current firmware
(a driver/firmware defect, not this feature; no reset was seen in these image runs); Vulkan is setup's default there, `--image-backend hip` picks HIP.
