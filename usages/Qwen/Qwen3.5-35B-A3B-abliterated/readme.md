## Docker-Based Model Serving
- Option 1: Pre-download and operate a Docker image within our system.
- Option 2: Pull and operate the Docker image directly from nvidia-ai-iot.
  - To pull and run the image directly, Docker must be installed, and the environment must include an NVIDIA GPU and NVIDIA Container Toolkit.
  - This allows you to use the latest Docker image version and makes image management easier. However, an internet connection is required, and the initial image download may take some time.
  - If the image has already been downloaded in advance, specify the corresponding image path in the IMAGE variable in the code below.
  - If you continue using the pull-based approach without manually pre-downloading the image, Docker will cache the image locally. Therefore, the image only needs to be downloaded once, and subsequent runs will start much faster.
  - Once cached, Docker uses the local image, so an internet connection is no longer required for running the container.
- NVIDIA official GitHub repository for edge devices: https://github.com/NVIDIA-AI-IOT
- NVIDIA vLLM repository: https://github.com/orgs/NVIDIA-AI-IOT/packages/container/package/vllm
- NVIDIA llama.cpp repository: https://github.com/orgs/NVIDIA-AI-IOT/packages/container/package/llama_cpp
- NVIDIA ollama repository: https://github.com/orgs/NVIDIA-AI-IOT/packages/container/package/ollama

## Resources
- Model Source : huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated

## 1. (Local hosting) Test Serving
- Server (Jetson Thor)
```
IMAGE=ghcr.io/nvidia-ai-iot/vllm:0.16.0-g15d76f74e-r38.2-arm64-sbsa-cu130-24.04
MODEL=huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated
SERVED_MODEL_NAME=Huihui-Qwen3.5-35B-A3B-abliterated
PORT=12222

mkdir -p "$HOME/thor-hf-cache"

docker run --rm -it \
  --runtime nvidia --gpus all \
  --ipc=host \
  --network host \
  -e HF_HOME=/data/models/huggingface \
  -e HF_HUB_CACHE=/data/models/huggingface/hub \
  -e TRANSFORMERS_CACHE=/data/models/huggingface/hub \
  -v "$HOME/thor-hf-cache:/data/models/huggingface" \
  "$IMAGE" \
  vllm serve "$MODEL" \
    --host 127.0.0.1 \
    --port "$PORT" \
    --download-dir /data/models/huggingface/hub \
    --served-model-name "$SERVED_MODEL_NAME" \
    --tensor-parallel-size 1 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.85 \
    --max-num-seqs 1
```

- Client (Jetons Thor)
```
curl http://127.0.0.1:12222/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Huihui-Qwen3.5-35B-A3B-abliterated",
    "messages": [
      {
        "role": "user",
        "content": "Hi, how are you?"
      }
    ],
    "max_tokens": 512,
    "temperature": 0.7
  }'
```

## 2. (Local hosting) Recommended Serving
- Server
```
IMAGE=ghcr.io/nvidia-ai-iot/vllm:0.16.0-g15d76f74e-r38.2-arm64-sbsa-cu130-24.04
MODEL=huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated
SERVED_MODEL_NAME=Huihui-Qwen3.5-35B-A3B-abliterated
PORT=12222

mkdir -p "$HOME/thor-hf-cache"
mkdir -p "$HOME/thor-vllm-cache"
mkdir -p "$HOME/thor-torch-cache"

docker run --rm -it \
  --runtime nvidia --gpus all \
  --ipc=host \
  --network host \
  -e HF_HOME=/data/models/huggingface \
  -e HF_HUB_CACHE=/data/models/huggingface/hub \
  -e TRANSFORMERS_CACHE=/data/models/huggingface/hub \
  -e VLLM_LOGGING_LEVEL=INFO \
  -v "$HOME/thor-hf-cache:/data/models/huggingface" \
  -v "$HOME/thor-vllm-cache:/root/.cache/vllm" \
  -v "$HOME/thor-torch-cache:/root/.cache/torch" \
  "$IMAGE" \
  vllm serve "$MODEL" \
    --host 127.0.0.1 \
    --port "$PORT" \
    --download-dir /data/models/huggingface/hub \
    --served-model-name "$SERVED_MODEL_NAME" \
    --tensor-parallel-size 1 \
    --max-model-len 65536 \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --speculative-config '{"method":"qwen3_next_mtp","num_speculative_tokens":2}' \
    --gpu-memory-utilization 0.9 \
    --kv-cache-dtype fp8 \
    --max-num-seqs 1
```
- Client (Jetson Thor)
```
curl http://127.0.0.1:12222/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Huihui-Qwen3.5-35B-A3B-abliterated",
    "messages": [
      {
        "role": "user",
        "content": "Explain what vLLM is in one paragraph."
      }
    ],
    "max_tokens": 1024,
    "temperature": 0.7
  }'
```


## 3. (Internal/Public hosting) Recommended Serving 🔥
- Server (Jetson Thor)
```
IMAGE=ghcr.io/nvidia-ai-iot/vllm:0.16.0-g15d76f74e-r38.2-arm64-sbsa-cu130-24.04
MODEL=huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated
SERVED_MODEL_NAME=Huihui-Qwen3.5-35B-A3B-abliterated
HOST=0.0.0.0
PORT=12222

mkdir -p "$HOME/thor-hf-cache"
mkdir -p "$HOME/thor-vllm-cache"
mkdir -p "$HOME/thor-torch-cache"

docker run --rm -it \
  --runtime nvidia --gpus all \
  --ipc=host \
  --network host \
  -e HF_HOME=/data/models/huggingface \
  -e HF_HUB_CACHE=/data/models/huggingface/hub \
  -e TRANSFORMERS_CACHE=/data/models/huggingface/hub \
  -e VLLM_LOGGING_LEVEL=INFO \
  -v "$HOME/thor-hf-cache:/data/models/huggingface" \
  -v "$HOME/thor-vllm-cache:/root/.cache/vllm" \
  -v "$HOME/thor-torch-cache:/root/.cache/torch" \
  "$IMAGE" \
  vllm serve "$MODEL" \
    --host "$HOST" \
    --port "$PORT" \
    --download-dir /data/models/huggingface/hub \
    --served-model-name "$SERVED_MODEL_NAME" \
    --tensor-parallel-size 1 \
    --max-model-len 65536 \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --speculative-config '{"method":"qwen3_next_mtp","num_speculative_tokens":2}' \
    --gpu-memory-utilization 0.9 \
    --kv-cache-dtype fp8 \
    --max-num-seqs 1
```

- Client (Other End-point)
```
curl http://SERVER_IP:12222/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-secure-api-key" \
  -d '{
    "model": "Huihui-Qwen3.5-35B-A3B-abliterated",
    "messages": [
      {
        "role": "user",
        "content": "Explain what vLLM is in one paragraph."
      }
    ],
    "max_tokens": 1024,
    "temperature": 0.7
  }'
```
