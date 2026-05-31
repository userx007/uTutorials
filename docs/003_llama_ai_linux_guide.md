# Running Llama AI on Linux Desktop

A complete guide to installing, configuring, and using Meta's Llama language models on a Linux desktop.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation Methods](#installation-methods)
   - [Method 1: Ollama (Recommended)](#method-1-ollama-recommended)
   - [Method 2: llama.cpp (Advanced)](#method-2-llamacpp-advanced)
   - [Method 3: LM Studio (GUI)](#method-3-lm-studio-gui)
4. [Downloading Llama Models](#downloading-llama-models)
5. [Running Llama](#running-llama)
6. [GPU Acceleration](#gpu-acceleration)
7. [Web UI Frontends](#web-ui-frontends)
8. [API Usage](#api-usage)
9. [Performance Tips](#performance-tips)
10. [Troubleshooting](#troubleshooting)

---

## Overview

Llama (Large Language Model Meta AI) is an open-weight family of large language models released by Meta. You can run Llama models **fully locally** on your Linux machine — no cloud subscription required, with complete privacy.

**Current model families (as of mid-2025):**

| Model | Parameters | Use Case |
|-------|-----------|----------|
| Llama 3.2 | 1B, 3B | Edge / low-RAM devices |
| Llama 3.1 | 8B, 70B, 405B | General use / high-end hardware |
| Llama 3.3 | 70B | Latest general purpose |
| Code Llama | 7B–70B | Code generation |

---

## Prerequisites

### Hardware Requirements

| Model Size | Minimum RAM | Recommended RAM | GPU VRAM |
|-----------|-------------|-----------------|----------|
| 1B–3B | 4 GB | 8 GB | 4 GB |
| 7B–8B | 8 GB | 16 GB | 6–8 GB |
| 13B | 16 GB | 32 GB | 12 GB |
| 70B | 48 GB | 64 GB | 40+ GB |

> **No GPU?** No problem. Llama can run on CPU-only hardware, though it will be significantly slower.

### Software Requirements

- Linux (Ubuntu 20.04+, Fedora 38+, Debian 11+, Arch, or similar)
- `curl` or `wget`
- Python 3.8+ (for some methods)
- Git
- Optional: NVIDIA GPU with CUDA 11.8+ or AMD GPU with ROCm 5.6+

### Check your system

```bash
# Check RAM
free -h

# Check CPU cores
nproc

# Check GPU (NVIDIA)
nvidia-smi

# Check GPU (AMD)
rocm-smi

# Check available disk space (models are large!)
df -h ~
```

---

## Installation Methods

### Method 1: Ollama (Recommended)

[Ollama](https://ollama.com) is the easiest way to run Llama on Linux. It handles model downloads, quantization, and serving automatically.

#### Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This script installs the `ollama` binary and sets up a systemd service.

#### Verify installation

```bash
ollama --version
systemctl status ollama
```

#### Start the Ollama service (if not already running)

```bash
sudo systemctl start ollama
sudo systemctl enable ollama  # Start automatically on boot
```

---

### Method 2: llama.cpp (Advanced)

[llama.cpp](https://github.com/ggerganov/llama.cpp) is a high-performance C++ inference engine. It offers the most control and best raw performance, especially for custom quantization.

#### Install dependencies

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y build-essential cmake git

# Fedora
sudo dnf install -y gcc-c++ cmake git

# Arch
sudo pacman -S base-devel cmake git
```

#### Clone and build

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

# CPU-only build
cmake -B build
cmake --build build --config Release -j$(nproc)

# NVIDIA GPU build (CUDA)
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)

# AMD GPU build (ROCm)
cmake -B build -DGGML_HIPBLAS=ON
cmake --build build --config Release -j$(nproc)
```

The compiled binaries will be in `./build/bin/`.

---

### Method 3: LM Studio (GUI)

LM Studio provides a graphical interface for running Llama models. It's ideal for users who prefer a desktop application over the command line.

#### Download

```bash
# Download the AppImage from lmstudio.ai
wget https://lmstudio.ai/download/linux
chmod +x LM_Studio-*.AppImage
./LM_Studio-*.AppImage
```

LM Studio includes a model browser, chat interface, and a local API server — all in one app.

---

## Downloading Llama Models

### Via Ollama (easiest)

```bash
# Pull Llama 3.2 (3B) — good starting point
ollama pull llama3.2

# Pull Llama 3.1 (8B) — better quality, needs more RAM
ollama pull llama3.1:8b

# Pull Llama 3.3 (70B) — best quality, needs high-end hardware
ollama pull llama3.3:70b

# Pull Code Llama for coding tasks
ollama pull codellama

# List downloaded models
ollama list
```

### Via Hugging Face (for llama.cpp)

Models in GGUF format are available on Hugging Face and work directly with llama.cpp.

```bash
pip install huggingface-hub

# Download a quantized GGUF model (Q4_K_M is a good balance)
huggingface-cli download \
  bartowski/Meta-Llama-3.1-8B-Instruct-GGUF \
  Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  --local-dir ./models
```

#### Understanding quantization levels

| Quantization | Size (8B model) | Quality | Speed |
|-------------|-----------------|---------|-------|
| Q2_K | ~3 GB | Low | Fastest |
| Q4_K_M | ~5 GB | Good | Fast |
| Q5_K_M | ~6 GB | Better | Moderate |
| Q8_0 | ~9 GB | Best | Slower |
| F16 | ~16 GB | Full | Slowest |

> **Recommendation:** Start with `Q4_K_M` — it offers the best trade-off between quality and performance.

---

## Running Llama

### With Ollama

#### Interactive chat (terminal)

```bash
ollama run llama3.2
```

Type your messages and press Enter. Type `/bye` to exit.

#### Single prompt (non-interactive)

```bash
ollama run llama3.2 "Explain quantum computing in simple terms"
```

#### System prompts

```bash
ollama run llama3.1:8b "What is Rust?" --system "You are a senior software engineer. Be concise."
```

### With llama.cpp

#### Interactive chat

```bash
./build/bin/llama-cli \
  -m ./models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -c 4096 \
  --interactive \
  --chat-template llama3
```

#### Single generation

```bash
./build/bin/llama-cli \
  -m ./models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -p "The capital of Germany is" \
  -n 100
```

#### Key flags for llama-cli

| Flag | Description |
|------|-------------|
| `-m` | Path to the model file |
| `-c` | Context size (tokens) |
| `-n` | Max tokens to generate |
| `--threads` | CPU threads to use |
| `--n-gpu-layers` | Layers to offload to GPU |
| `--temp` | Temperature (creativity, 0.0–2.0) |

---

## GPU Acceleration

### NVIDIA (CUDA)

```bash
# Verify CUDA is available
nvidia-smi

# With Ollama — GPU is used automatically if drivers are installed

# With llama.cpp — offload layers to GPU
./build/bin/llama-cli \
  -m ./models/model.gguf \
  --n-gpu-layers 35   # Increase to offload more layers; use 999 for all layers
```

### AMD (ROCm)

```bash
# Install ROCm (Ubuntu)
sudo apt install rocm-hip-sdk

# Set environment variable
export HSA_OVERRIDE_GFX_VERSION=10.3.0  # Adjust for your GPU

# Ollama detects AMD GPUs automatically after ROCm install
ollama run llama3.2
```

### Check if GPU is being used

```bash
# Monitor GPU usage while running a model
watch -n 1 nvidia-smi      # NVIDIA
watch -n 1 rocm-smi        # AMD
```

---

## Web UI Frontends

Running a local web interface makes Llama much more user-friendly.

### Open WebUI (formerly Ollama WebUI)

The most popular browser-based chat UI for Ollama.

#### Install with Docker

```bash
# Install Docker if not present
sudo apt install docker.io docker-compose

# Run Open WebUI
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Then open your browser at `http://localhost:3000`.

#### Install without Docker (pip)

```bash
pip install open-webui
open-webui serve
```

### text-generation-webui

A powerful alternative with more advanced features including LoRA loading and model fine-tuning.

```bash
git clone https://github.com/oobabooga/text-generation-webui
cd text-generation-webui
pip install -r requirements.txt
python server.py --model ./models/model.gguf
```

Access at `http://localhost:7860`.

---

## API Usage

Both Ollama and llama.cpp expose a REST API, making it easy to integrate Llama into your own applications.

### Ollama API

Ollama provides an OpenAI-compatible API at `http://localhost:11434`.

#### Basic request

```bash
curl http://localhost:11434/api/generate \
  -d '{
    "model": "llama3.2",
    "prompt": "Why is the sky blue?",
    "stream": false
  }'
```

#### Chat completions (OpenAI-compatible)

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is the speed of light?"}
    ]
  }'
```

#### Using the Python client

```bash
pip install ollama
```

```python
import ollama

response = ollama.chat(
    model='llama3.2',
    messages=[{'role': 'user', 'content': 'Explain black holes simply.'}]
)
print(response['message']['content'])
```

#### Using the OpenAI Python SDK with Ollama

```python
from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:11434/v1',
    api_key='ollama'  # Required but unused
)

response = client.chat.completions.create(
    model='llama3.2',
    messages=[{'role': 'user', 'content': 'Hello!'}]
)
print(response.choices[0].message.content)
```

### llama.cpp Server API

```bash
# Start the server
./build/bin/llama-server \
  -m ./models/model.gguf \
  --host 0.0.0.0 \
  --port 8080

# Query it
curl http://localhost:8080/completion \
  -d '{"prompt": "Once upon a time", "n_predict": 128}'
```

---

## Performance Tips

### 1. Choose the right quantization
Use `Q4_K_M` for most cases. Use `Q5_K_M` or `Q8_0` only if you have RAM to spare and need higher quality.

### 2. Maximize GPU offloading
If you have a GPU, offload as many layers as your VRAM allows:
```bash
# llama.cpp: try increasing --n-gpu-layers until you run out of VRAM
--n-gpu-layers 999   # Attempt to offload all layers
```

### 3. Tune thread count
```bash
# Use physical cores, not hyperthreaded logical cores
--threads $(nproc --all)
```

### 4. Increase context only when needed
Larger context windows use more memory and slow down generation. Use `-c 2048` for simple tasks and `-c 8192` only when needed.

### 5. Use mmap for large models
llama.cpp uses memory mapping by default, which is efficient for large models. Don't disable it unless you have a specific reason.

### 6. Persistent model loading with Ollama
Ollama keeps the model in memory between requests for up to 5 minutes by default. Set a longer keep-alive for interactive use:
```bash
OLLAMA_KEEP_ALIVE=1h ollama serve
```

---

## Troubleshooting

### Model is very slow

- Check if GPU is being used: `nvidia-smi` or `rocm-smi`
- Try a smaller or more aggressively quantized model
- Reduce context size with `-c 2048`
- Close memory-hungry applications

### "Out of memory" errors

- Use a smaller model or lower quantization (e.g., Q2_K)
- Reduce the number of GPU layers (`--n-gpu-layers 20`)
- Set `OLLAMA_MAX_LOADED_MODELS=1` to prevent multiple models in memory

### Ollama service not starting

```bash
# Check logs
journalctl -u ollama -n 50

# Restart the service
sudo systemctl restart ollama
```

### CUDA not detected

```bash
# Verify CUDA installation
nvcc --version
nvidia-smi

# Reinstall Ollama after CUDA drivers are properly installed
curl -fsSL https://ollama.com/install.sh | sh
```

### Permission denied on AppImage (LM Studio)

```bash
chmod +x LM_Studio-*.AppImage
./LM_Studio-*.AppImage --no-sandbox
```

### Model download stuck or corrupted

```bash
# With Ollama — remove and re-pull
ollama rm llama3.2
ollama pull llama3.2

# With Hugging Face CLI — resume interrupted download
huggingface-cli download <repo> <file> --local-dir ./models --resume-download
```

---

## Quick Reference

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Download and run a model in one command
ollama run llama3.2

# List all models
ollama list

# Remove a model
ollama rm llama3.2

# Check Ollama service status
systemctl status ollama

# Start Open WebUI (Docker)
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

---

*Guide current as of May 2026. Check [ollama.com](https://ollama.com) and [github.com/ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) for the latest updates.*
