IMAGE=ghcr.io/nvidia-ai-iot/vllm:0.16.0-g15d76f74e-r38.2-arm64-sbsa-cu130-24.04 && \

docker run --rm -it \
  --runtime nvidia --gpus all \
  --ipc=host --network host \
  -e HF_HOME=/data/models/huggingface \
  -e HF_HUB_CACHE=/data/models/huggingface/hub \
  -e TRANSFORMERS_CACHE=/data/models/huggingface/hub \
  -v "$HOME/thor-hf-cache:/data/models/huggingface" \
  -v "$HOME/thor-vllm-cache:/root/.cache/vllm" \
  -v "$HOME/thor-torch-cache:/root/.cache/torch" \
  "$IMAGE" \
  vllm serve huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated \
    --host 0.0.0.0 \
    --port 12222 \
    --download-dir /data/models/huggingface/hub \
    --served-model-name Huihui-Qwen3.5-35B-A3B-abliterated \
    --tensor-parallel-size 1 \
    --max-model-len 65536 \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --speculative-config '{"method":"qwen3_next_mtp","num_speculative_tokens":2}' \
    --gpu-memory-utilization 0.9 \
    --kv-cache-dtype fp8 \
    --max-num-seqs 1
