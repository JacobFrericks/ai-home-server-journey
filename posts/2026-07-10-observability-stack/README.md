# Giving the Server Eyes: A Local Prometheus + Loki + Grafana Stack

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, docker, observability, prometheus, grafana, loki, gpu

---

With the whole stack finally living in one Compose file, I had a nagging blind spot: I could *run* everything, but I couldn't *see* anything. How hot does the 3090 get under a transcode? How much VRAM does Ollama actually hold onto between prompts? When Home Assistant hiccups, is there anything in the logs, and where would I even look? The answers were all one `ssh` and a pile of ad-hoc commands away — which is another way of saying I never actually checked. So this post is about building the eyes: a local Prometheus + Loki + Grafana stack, running on the same box, showing me the server's vital signs without a single byte leaving the house.

## Why PLG, and why keep it separate

I went with the classic **Prometheus + Loki + Grafana** trio — Prometheus for metrics, Loki for logs, Grafana as the single pane of glass over both, with **Grafana Alloy** shipping container logs into Loki. It's the boringly standard homelab choice for good reason: every service I run already speaks Prometheus or has an off-the-shelf exporter, and it's all self-hosted, which is the whole point of this project.

The one design decision I want to call out is that the monitoring stack is a **completely separate Compose project** from the app stack:

```
/home/jacob/docker/ai-stack/
  docker-compose.yml            # project "ai-stack" — Ollama, Open WebUI, HA, Plex, ...
  monitoring/
    docker-compose.yml          # project "monitoring" — its own bridge network
```

The reasoning is blunt: monitoring should never take down the thing it's monitoring. Keeping it as its own project (`name: monitoring`, its own bridge network) means I can pull, restart, or blow away the entire observability stack without so much as touching Plex or the LLM. They're neighbors, not roommates.

## The one hard rule: only Grafana faces the LAN

Prometheus and Loki ship with **no authentication**. None. Anyone who can reach their ports can read every metric and every log line, and Prometheus's admin API can even delete data. So the exposure policy here is a hard rule, not a preference:

> **Only Grafana publishes a host port** (`3000:3000`, behind a login). Prometheus, Loki, Alloy, and every exporter have **no published port at all** — they're reachable only by other containers on the `monitoring` bridge, by service name.

The firewall reflects exactly that — `sudo ufw allow 3000/tcp` and nothing else. After deploying, I re-ran a port scan from my laptop to confirm the posture held: `3000` open, and `9090` (Prometheus), `3100` (Loki), `9100`, `9835`, `9115` all refused. If it's not Grafana, it doesn't answer the LAN.

## Scraping host-networked services from inside a container

Here's the wrinkle that took a minute to get right. Most of my app services (Ollama, Open WebUI, Home Assistant, Plex, and the Wyoming voice bits) run on the **host network** — they bind the host's interfaces directly. But Prometheus lives on the isolated `monitoring` bridge. From inside that bridge, `localhost` means *the Prometheus container*, not the host, so it can't just scrape `localhost:8123`.

The fix is Docker's host-gateway alias:

```yaml
prometheus:
  extra_hosts:
    - "host.docker.internal:host-gateway"
```

Now `host.docker.internal` resolves to the host from inside the container, and every scrape target is just `host.docker.internal:<port>`. Same trick for the blackbox and Plex exporters.

For Home Assistant specifically, I didn't need a third-party exporter at all — HA has a **native Prometheus integration**. One line in its config:

```yaml
# configuration.yaml
prometheus:
```

...and a `docker restart homeassistant` later, it exposes `/api/prometheus` with every entity's state as a metric. I secured that endpoint with a long-lived access token that Prometheus reads from a file (`bearer_token_file`), not an env var — which leads directly into the gotcha of the whole build.

## The gotcha: a token the right user couldn't read

I dropped the HA token into `monitoring/prometheus/ha_token`, `chmod 600`, owned by my user (`jacob`, uid 1000) — exactly what you'd do for any secret. The HA scrape target immediately went **down**. No auth error in HA, just... down.

The cause: the `prom/prometheus` container doesn't run as root, and it doesn't run as me. It runs as **uid 65534 (`nobody`)**. A `chmod 600` file owned by uid 1000 is, by definition, unreadable by uid 65534 — Prometheus couldn't open its own bearer-token file. The file was *perfectly* secured, just against the wrong direction.

The fix keeps the tight `600` permission but hands ownership to the uid that actually needs it:

```bash
sudo chown 65534:65534 monitoring/prometheus/ha_token
sudo chmod 600 monitoring/prometheus/ha_token
```

After that the HA target came up green. The lesson I keep re-learning: a container's file permissions aren't about *your* user, they're about whatever uid the image drops to — and "nobody" is a real uid with real access rules.

> **Note:** Rotating this token later is: HA UI → Profile → Security → Long-Lived Access Tokens → delete + recreate → overwrite `ha_token` on the server (keep it `chmod 600`, keep the `65534` ownership) → `docker compose -p monitoring restart prometheus`.

## The dashboards

Grafana provisions its dashboards from JSON on disk, so they're version-controlled and survive a redeploy — no clicking around in a UI that forgets everything on the next `docker compose up`. I pulled in a few excellent community dashboards (Node Exporter Full, cAdvisor, the nvidia_gpu_exporter board) and hand-wrote a few custom ones. Two I'm particularly happy with:

- **Shared GPU / VRAM (Ollama vs Plex)** — the headline. One RTX 3090, 24 GB, shared between LLM inference (CUDA) and Plex hardware transcode (NVENC). This board shows total VRAM used against the 24 GB ceiling, a VRAM-% gauge, GPU util and temp, the **per-process VRAM split** between Ollama and Plex, and NVENC encoder sessions/fps. Getting the per-process split to resolve real names (instead of meaningless in-container PIDs) meant giving the GPU exporter `pid: host` so `nvidia-smi` can see the actual processes.
- **Plex Media Server** — a comprehensive 15-panel board off the Plex exporter: server up/down and version, library counts (**189 movies, 21 shows, 810 episodes**), library size by section (~1 TB total), watch-time by user, plus the Plex container's own CPU/memory and the NVENC transcode signal for live-context. Sessions and clients panels are written to render `0` when nobody's watching, because the exporter simply doesn't emit those series when the server is idle.

## Choosing a Plex exporter (a small detour worth taking)

The exporter I'd originally planned to use, `jsclayton/prometheus-plex-exporter`, ships **no versioned tag** — only `latest`. That fails my "pin every image to an explicit tag" rule, and I'm not building a long-lived scrape job on a moving `latest`. A second candidate was stale (last published 2019). I landed on **`ajalewis/plex-media-server-exporter`**, which ships real semver tags, takes `PLEX_SERVER` + `PLEX_TOKEN` env vars directly, and fits the existing `host.docker.internal` + `.env` pattern cleanly.

Before committing I actually read the `v3.0.0` changelog rather than trusting the version number — no breaking changes, benign release — and pinned it. Worth the ten minutes: an exporter is a long-lived dependency, and "it has a version tag" and "the version is sane" are two different checks.

## Wiring it into the same pipeline

The whole reason the last few months of work exist is so that *nothing* runs outside the pipeline, and the monitoring stack is no exception. Dependabot watches `monitoring/docker-compose.yml`'s pinned tags through a second `docker-compose` entry in `dependabot.yml`; CI validates *both* compose files and yamllints the monitoring config on every PR; and the existing auto-merge policy applies unchanged — patch/minor bumps auto-merge once CI is green, majors (and anything touching Home Assistant) stay open for a human. Secrets (`monitoring/.env` and the `ha_token`) are git-ignored and checked for by `deploy.sh`, which quietly skips the monitoring stack if they're missing so a fresh clone never fails a deploy over a secret it can't have.

---

The satisfying part isn't any single graph — it's that the server can now tell me how it's doing before I have to go asking. VRAM creeping toward the ceiling, a disk filling up, a container stuck in a restart loop, the GPU running hot mid-transcode: all of it is a glance at `:3000` away, and none of it leaves the house. The next honest gap is that alerts currently *evaluate* but don't *notify* anywhere — the rules fire into Grafana's own alerting UI and stop there. Wiring those to actually reach my phone is the obvious v2. But for now, the box has eyes, and that changes how it feels to run it.

---

[← Back to Home](../../README.md)
