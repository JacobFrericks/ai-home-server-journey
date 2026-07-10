# Consolidating the AI Stack with Docker Compose

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, docker, ollama, openwebui, compose

---

Since the original setup post, Ollama had been running as a native systemd service directly on Debian, while Open WebUI was a hand-run `docker run` command I'd typed once and never touched again. Two completely different management styles for two halves of the same stack — one gets updated with `apt`/the install script, the other with a manual `docker pull` and re-run. That's exactly the kind of asymmetry that turns into a maintenance headache, so the goal was to fold both into a single, declarative `docker compose` stack.

## NVIDIA Container Toolkit

Getting Ollama into a container meant it needed GPU access from inside Docker, which doesn't work out of the box. The NVIDIA Container Toolkit is what bridges that gap — it exposes the host's GPU and drivers to containers so they can talk to the 3090 as if they were running natively.

```bash
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

That last step registers an `nvidia` runtime with the Docker daemon, which is what lets a `deploy.resources.reservations.devices` block in a compose file request GPU access.

## Preserving Data (the Part I Was Most Careful About)

This was the part I couldn't afford to get wrong: I did **not** want to re-download the model library, and I definitely didn't want to lose Open WebUI's existing chat history and user accounts.

- For Ollama, I bind-mounted the real, existing data directory (`/usr/share/ollama/.ollama`) straight into the container, instead of letting it create a fresh one. (Worth noting: there was also a stale `/home/jacob/.ollama` on the box that was *not* the live store — the running service's models actually lived under the `ollama` user's home, so confirming the authoritative path first mattered.)
- For Open WebUI, I reused its existing named Docker volume by declaring it `external: true` in the compose file, so compose adopts the volume that's already there rather than creating a new empty one.

Both came up with everything intact — same models, same chats, same accounts, zero re-downloading.

## Pinned Versions

The original Open WebUI container was running `:main` — convenient for grabbing the newest features, bad for reproducibility, since the exact same command could pull a different image tomorrow. I pinned both images to specific version tags in the compose file so a `docker compose up` today and a `docker compose up` six months from now behave identically until I explicitly decide to bump a version.

## Host Networking for Parity

I kept both services on `network_mode: host`, matching the original setup. The main reason: Home Assistant's Ollama integration is already pointed at `127.0.0.1:11434`, and I didn't want to touch that config as part of what should otherwise be an invisible migration. Everything reaches the same addresses it always did.

> **Note:** Host networking means both containers share the host's network namespace directly, which is a looser isolation boundary than a bridge network with explicit port mappings. It's on my list as a future improvement — swap to a dedicated Docker network with only the necessary ports published — but it wasn't worth doing at the same time as this migration.

Here's a sanitized version of the resulting `docker-compose.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama:0.30.10
    container_name: ollama
    network_mode: host
    restart: unless-stopped
    environment:
      - OLLAMA_HOST=127.0.0.1:11434
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - /usr/share/ollama/.ollama:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.9.6
    container_name: open-webui
    network_mode: host
    restart: unless-stopped
    environment:
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - ENABLE_SIGNUP=false
      - OLLAMA_BASE_URL=http://127.0.0.1:11434
    volumes:
      - open-webui:/app/backend/data
    depends_on:
      - ollama

volumes:
  open-webui:
    external: true
```

And the matching `.env.example` (the real `.env` never leaves the server):

```bash
# Copy to .env and fill in a real value — never commit the real .env
WEBUI_SECRET_KEY=REPLACE_WITH_OPENSSL_RAND_HEX_32
```

> **Note:** `.env` is listed in `.gitignore` on the server and is never committed to any repo. Only `.env.example`, with placeholder values, is meant to be shared.

## The Cutover

The actual switch was anticlimactic, which is what you want from infrastructure work: I stopped and disabled the native Ollama systemd service (`sudo systemctl disable --now ollama`), double-checked the model directory was untouched, and brought the new stack up:

```bash
docker compose up -d
```

Open WebUI reconnected to Ollama immediately, the model list populated exactly as before, and Home Assistant's integration didn't even notice anything had changed. The family stack came back up cleanly, now managed with one command and one file instead of two different mental models.

---

[← Back to Home](../../README.md)
