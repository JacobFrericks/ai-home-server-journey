# Installing Ollama and OpenWebUI

**Date:** December 21, 2024  
**Author:** Jacob Frericks  
**Tags:** homelab, ai, ollama, openwebui, llm

---

With the hardware built and Debian installed, it's time to get the AI stack running! This post covers installing Ollama for running local LLMs and OpenWebUI as a ChatGPT-like interface.

## Use Case

This server is used by my entire family: myself, my wife, and my two young kids. My wife and I are power users who will leverage the full capabilities of the system, while the kids are entry-level users who will use it for simpler tasks and learning.

---

## Installing NVIDIA Drivers

Since I'm running an NVIDIA 3090, I needed to install the proprietary NVIDIA drivers to take advantage of GPU acceleration for AI workloads.

I followed the official Debian wiki instructions: [https://wiki.debian.org/NvidiaGraphicsDrivers](https://wiki.debian.org/NvidiaGraphicsDrivers)

The NVIDIA drivers are essential for running models efficiently on the GPU, dramatically improving inference speeds compared to CPU-only operation.

---

## Installing Ollama

Ollama is a fantastic tool for running large language models locally. It handles model downloads, manages the model runtime, and provides a simple API for interacting with models.

Installation was straightforward using the official install script:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This script automatically detects your system configuration and installs Ollama with all necessary dependencies. After installation, Ollama runs as a service and listens on port 11434.

> **Note**: This installation method downloads and executes a script directly from the internet. While convenient, always ensure you trust the source. For added security, you can download the script first, review it, and then execute it locally.

---

## Installing OpenWebUI

OpenWebUI provides a ChatGPT-like web interface for interacting with local LLMs. It's perfect for making AI accessible to the whole family without needing to use command-line tools.

I installed OpenWebUI using Docker to keep it isolated and easy to manage:

```bash
docker run -d --network=host -v open-webui:/app/backend/data -e OLLAMA_BASE_URL=http://127.0.0.1:11434 --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

Breaking down this command:
- `--network=host`: Uses the host network, allowing OpenWebUI to easily communicate with Ollama
- `-v open-webui:/app/backend/data`: Persists user data and configurations
- `-e OLLAMA_BASE_URL=http://127.0.0.1:11434`: Configures OpenWebUI to connect to the local Ollama instance
- `--restart always`: Ensures OpenWebUI automatically starts after system reboots
- `ghcr.io/open-webui/open-webui:main`: Uses the latest main branch image

> **Note**: This setup uses `--network=host` for simplicity in a home environment. For production deployments, consider using explicit port mappings (e.g., `-p 3000:8080`) and pinning to a specific version tag instead of `:main` for better security and stability.

---

## Configuring OpenWebUI

After OpenWebUI was running, I accessed it through the web interface and performed initial configuration:

- **User Setup**: Created one admin account (for me) and one regular user account for family use
- **System Prompts**: I'm holding off on customizing system prompts and other advanced settings until the system becomes more stable and I better understand our usage patterns
- **Future Plans**: As we use the system more, I'll fine-tune permissions, create additional user accounts for the kids, and customize the experience for different user levels

---

## Model Selection

Choosing the right model was crucial. I needed something that:
- Fits within 24GB of VRAM on my single 3090
- Provides good quality responses for general use
- Works well with other integrations I plan to add (like Home Assistant)

After research, I selected **Gemma 3 27B** (`gemma3:27b`). Here's why:

1. **Quantized for Efficiency**: The model is quantized, allowing it to fit comfortably on my 3090's 24GB VRAM
2. **Easy to Install**: Available directly through Ollama with a simple pull command
3. **Community Proven**: Others in the home server community have reported success using it, particularly for Home Assistant integrations
4. **Good Balance**: 27 billion parameters provide a nice balance between capability and resource usage

Installing the model was simple:

```bash
ollama pull gemma3:27b
```

---

## Next Steps

With Ollama and OpenWebUI running, the family can now interact with a powerful local LLM! Future posts will cover:
- Integrating with Home Assistant for smart home automation
- Fine-tuning system prompts for different family members
- Performance optimization and monitoring
- Adding additional models for specialized tasks

---

[← Back to Home](../../README.md)
