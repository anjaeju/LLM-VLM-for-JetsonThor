## Docker-Based Model Serving for Dfloat WAN 2.2 (Image-2-Video)
- Option 1: Pre-download and operate a Docker image within our system.
- Option 2: Pull and operate the Docker image directly from nvidia-ai-iot.
- NVIDIA official GitHub repository for edge devices: https://github.com/NVIDIA-AI-IOT
- NVIDIA vLLM repository: https://github.com/orgs/NVIDIA-AI-IOT/packages/container/package/vllm

## Resources
- Model Source : DFloat11/Wan2.2-I2V-A14B-DF11
  - Model	Model Size	Peak GPU Memory
  - Wan-AI/Wan2.2-I2V-A14B (BFloat16)	~56 GB
  - Wan-AI/Wan2.2-I2V-A14B (DFloat11)	~29.12 GB
  - Wan-AI/Wan2.2-I2V-A14B (DFloat11 + CPU Offloading) 19.47 + 19.44 GB = ~22 GB
- Usage:
  - bash Run-Dfloat-wan22-i2v.sh
  - WORKSPACE=/path/to/workspace bash Run-Dfloat-wan22-i2v.sh
  - CONTAINER_NAME=Dfloat-wan22
- Notes:
  - WORKSPACE defaults to the directory where this script is launched.
  - The workspace is mounted at /workspace inside the container.
  - Hugging Face, vLLM, and Torch caches are kept under $HOME/thor-*.
  - The container is named and not removed, so local changes survive restarts.
  - Exiting the shell leaves the container available for docker stop/start reuse.
  - Required Python packages are installed before opening the container shell.
  - To change image, workspace, cache mounts, or GPU settings, remove the old container first.
  - Override MODEL, SERVED_MODEL_NAME, HOST, PORT, WORKSPACE, CONTAINER_NAME, or INSTALL_PACKAGES as needed.

- Server
```
IMAGE=ghcr.io/nvidia-ai-iot/vllm:0.16.0-g15d76f74e-r38.2-arm64-sbsa-cu130-24.04
HOST="${HOST:-0.0.0.0}"
PORT="${PORT:-12232}"
WORKSPACE="${WORKSPACE:-$(pwd)}"
CONTAINER_NAME="${CONTAINER_NAME:-wan22-i2v-vllm}"
INSTALL_PACKAGES="${INSTALL_PACKAGES:-1}"
PYTHON_PACKAGES=(
  "git+https://github.com/huggingface/diffusers"
  "dfloat11[cuda12]"
  "transformers"
  "accelerate>=1.1.1"
  "safetensors"
  "sentencepiece"
  "pillow"
  "imageio[ffmpeg]"
  "tqdm"
  "easydict"
  "ftfy"
  "dashscope"
)


open_shell() {
  if [ "$INSTALL_PACKAGES" = "1" ]; then
    # IMPORTANT:
    # Package names like accelerate>=1.1.1, imageio[ffmpeg], dfloat11[cuda12]
    # must be shell-escaped before passing into bash -lc.
    local packages
    packages="$(printf '%q ' "${PYTHON_PACKAGES[@]}")"

    docker exec -it "$CONTAINER_NAME" bash -lc "pip install -U $packages && bash"
  else
    docker exec -it "$CONTAINER_NAME" bash
  fi
}

mkdir -p "$HOME/thor-hf-cache"
mkdir -p "$HOME/thor-vllm-cache"
mkdir -p "$HOME/thor-torch-cache"

if docker container inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
  if [ "$(docker inspect -f '{{.State.Running}}' "$CONTAINER_NAME")" != "true" ]; then
    docker start "$CONTAINER_NAME" >/dev/null
  fi

  open_shell
  exit 0
fi

docker run -d \
  --name "$CONTAINER_NAME" \
  --runtime nvidia --gpus all \
  --ipc=host --network host \
  -e HF_HOME=/data/models/huggingface \
  -e HF_HUB_CACHE=/data/models/huggingface/hub \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  -e HOST="$HOST" \
  -e PORT="$PORT" \
  -v "$WORKSPACE:/workspace" \
  -v "$HOME/thor-hf-cache:/data/models/huggingface" \
  -v "$HOME/thor-vllm-cache:/root/.cache/vllm" \
  -v "$HOME/thor-torch-cache:/root/.cache/torch" \
  -w /workspace \
  "$IMAGE" \
  sleep infinity >/dev/null

open_shell
```

- Generation Scripts
```
import time
import torch
import numpy as np
import argparse
from diffusers import WanImageToVideoPipeline
from diffusers.utils import export_to_video, load_image
from dfloat11 import DFloat11Model

parser = argparse.ArgumentParser(description='Image to Video generation using Wan2.2-I2V model')
parser.add_argument('--cpu_offload', action='store_true', help='Enable CPU offloading')
parser.add_argument('--image_path', type=str, default="https://huggingface.co/datasets/YiYiXu/testing-images/resolve/main/wan_i2v_input.JPG", help='Path or URL to the input image')
parser.add_argument('--width', type=int, default=640, help='Output video width')
parser.add_argument('--height', type=int, default=480, help='Output video height')
parser.add_argument('--prompt', type=str, default="Summer beach vacation style, a white cat wearing sunglasses sits on a surfboard. The fluffy-furred feline gazes directly at the camera with a relaxed expression. Blurred beach scenery forms the background featuring crystal-clear waters, distant green hills, and a blue sky dotted with white clouds. The cat assumes a naturally relaxed posture, as if savoring the sea breeze and warm sunlight. A close-up shot highlights the feline's intricate details and the refreshing atmosphere of the seaside.", help='Prompt for video generation')
parser.add_argument('--negative_prompt', type=str, default="色调艳丽，过曝，静态，细节模糊不清，字幕，风格，作品，画作，画面，静止，整体发灰，最差质量，低质量，JPEG压缩残留，丑陋的，残缺的，多余的手指，画得不好的手部，画得不好的脸部，畸形的，毁容的，形态畸形的肢体，手指融合，静止不动的画面，杂乱的背景，三条腿，背景人很多，倒着走", help='Negative prompt for video generation')
parser.add_argument('--num_frames', type=int, default=41, help='Number of frames to generate')
parser.add_argument('--guidance_scale', type=float, default=3.5, help='Guidance scale for generation')
parser.add_argument('--num_inference_steps', type=int, default=8, help='Number of inference steps')
parser.add_argument('--seed', type=int, default=42, help='Random seed for generation')
parser.add_argument('--output', type=str, default='i2v_output.mp4', help='Output video path')
parser.add_argument('--fps', type=int, default=16, help='FPS of output video')

args = parser.parse_args()

image = load_image(args.image_path)

pipe = WanImageToVideoPipeline.from_pretrained("Wan-AI/Wan2.2-I2V-A14B-Diffusers", torch_dtype=torch.bfloat16)

DFloat11Model.from_pretrained(
    "DFloat11/Wan2.2-I2V-A14B-DF11",
    device="cpu",
    cpu_offload=args.cpu_offload,
    bfloat16_model=pipe.transformer,
)
DFloat11Model.from_pretrained(
    "DFloat11/Wan2.2-I2V-A14B-2-DF11",
    device="cpu",
    cpu_offload=args.cpu_offload,
    bfloat16_model=pipe.transformer_2,
)

pipe.enable_model_cpu_offload()

max_area = args.width * args.height
aspect_ratio = image.height / image.width
mod_value = pipe.vae_scale_factor_spatial * pipe.transformer.config.patch_size[1]
height = round(np.sqrt(max_area * aspect_ratio)) // mod_value * mod_value
width = round(np.sqrt(max_area / aspect_ratio)) // mod_value * mod_value
image = image.resize((width, height))

generator = torch.Generator(device="cuda").manual_seed(args.seed)

start_time = time.time()
output = pipe(
    image=image,
    prompt=args.prompt,
    negative_prompt=args.negative_prompt,
    height=height,
    width=width,
    num_frames=args.num_frames,
    guidance_scale=args.guidance_scale,
    num_inference_steps=args.num_inference_steps,
    generator=generator,
).frames[0]
print(f"Time taken: {time.time() - start_time:.2f} seconds")

export_to_video(output, args.output, fps=args.fps)

max_memory = torch.cuda.max_memory_allocated()
print(f"Max memory: {max_memory / (1000 ** 3):.2f} GB")
```

- the results:
