# LLM-for-JetsonThor

LLM-for-JetsonThor is a practical repository for running, testing, and benchmarking Large Language Models (including multi-modal) on NVIDIA Jetson Thor.
This project focuses on deploying local LLM inference servers using Docker-based environments such as vLLM and Ollama, with support for OpenAI-compatible API testing.

<!-- UPDATE_TIME_START -->
Last updated: 2026-04-30 11:00:03 KST
<!-- UPDATE_TIME_END -->

## Overview
This repository is designed for developers who want to run LLMs directly on Jetson Thor devices.

Main goals:

- Run LLM inference on Jetson Thor
- Serve models with OpenAI-compatible APIs
- Test vLLM, Llama.cpp, and Ollama deployment workflows
- Manage Hugging Face model cache and Docker volumes
- Benchmark latency, throughput, and memory usage
- Document practical troubleshooting steps for Jetson-based LLM serving

## Listup

- Qwen series
  - [Qwen 3.5] huihui-ai/Huihui-Qwen3.5-35B-A3B-abliterated

## Repository Structure (On-going)

```text
LLM-for-JetsonThor/
├── docker/
├── models/
│   ├── Qwen/
│   │   ├── vLLM/
│   │   ├── Llama.cpp/
│   │   ├── Ollama/
│   │   ├── HuggingFace Serve/
│   │   └── openai_inference /
│   ├── SmolVLM/
│   │   ├── vLLM/
│   │   ├── Llama.cpp/
│   │   ├── Ollama/
│   │   ├── HuggingFace Serve/
│   │   └── openai_inference /
│   └── Others .../
├── docs/
│   ├── setup.md
│   ├── troubleshooting.md
│   └── benchmark.md
└── README.md
```

## Citation
Thanks for @pastoriomarco for the great work
- https://github.com/pastoriomarco/thor_llm
