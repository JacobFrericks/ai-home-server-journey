# AI Home Server Journey

Welcome to my documentation of building an AI-powered home server! This repository chronicles my adventure into self-hosting AI models and services at home.

## 📝 What's This About?

I'm documenting my journey of setting up a home server capable of running AI workloads. From hardware selection to software configuration, I'll share everything I learn along the way.

## 🗂️ Content

### Blog Posts

- [Welcome to My AI Home Server Journey](posts/2024-12-05-welcome/README.md) - Introduction to this project and what to expect
- [Desktop Build Parts List](posts/2024-12-05-desktop-build/README.md) - The hardware components chosen for this build and why
- [Installing Ollama and OpenWebUI](posts/2024-12-21-ollama-openwebui-setup/README.md) - Setting up local LLM inference with Ollama and a web interface
- [Locking Down the Server: SSH, Firewall, and Secrets](posts/2026-07-10-securing-the-server/README.md) - A security audit covering SSH hardening, a router-reserved IP, a UFW firewall, and fixing an empty Open WebUI secret key
- [Consolidating the AI Stack with Docker Compose](posts/2026-07-10-docker-compose-migration/README.md) - Migrating Ollama and Open WebUI from a native service and a hand-run container into one declarative Compose stack
- [Automating OS Updates (and What I Chose NOT to Automate)](posts/2026-07-10-automating-updates/README.md) - Setting up unattended-upgrades for security patches while deliberately holding the GPU stack and container updates

### Topics Covered

- **Home Server Setup**: Hardware selection and configuration
- **AI & Machine Learning**: Running local LLMs, image generation, and other AI applications
- **Self-Hosted Solutions**: Privacy-focused alternatives and taking control of your data

## 🎯 Goals

- Build a capable home server for AI workloads
- Run local LLMs (like Llama, Mistral) without cloud dependencies
- Self-host various AI-powered services
- Document the entire process for others to learn from

## 📁 Repository Structure

```
.
├── README.md           # This file - main introduction
└── posts/              # Blog posts, each in their own directory
    └── YYYY-MM-DD-title/
        ├── README.md   # The post content
        └── images/     # Images for that post
```

## 📄 License

This project is open source and available under the MIT License.
