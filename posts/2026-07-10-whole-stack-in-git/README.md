# Bringing the Whole Stack Under One Roof: Dependabot, CI, and Containerizing Plex

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, docker, compose, dependabot, ci-cd, plex, home-assistant

---

In the last post I said the container-update story was "the next piece of that puzzle, once there's a proper pipeline to make it safe." This is that piece.

The compose migration got Ollama and Open WebUI into one declarative file — but that was only half the box. Home Assistant, Piper (Wyoming TTS), and Whisper (Wyoming STT) were still three hand-run `docker run` containers I'd started once and forgotten, and Plex was a native `.deb` systemd service updating itself through `apt`. So I was right back to the asymmetry I'd just spent a post complaining about, only with more moving parts. The goal here: **one compose file, in git, with automated-but-safe updates for the entire stack.**

## Step Zero: Pin the Moving Tags

The three stray containers were all running moving tags — Home Assistant on `:stable`, Piper and Whisper on `:latest`. Those are convenient and completely opaque: the same tag can point at a different image tomorrow, and there's nothing for an update tool to bump *toward*.

Dependabot's docker ecosystem only opens a meaningful PR when it can compare a concrete version against a newer concrete version, so the first move was to resolve each running image to its actual published version and pin it — the same thing I'd already done for Ollama and Open WebUI:

| Service | Was | Pinned to |
|---|---|---|
| homeassistant | `:stable` | `ghcr.io/home-assistant/home-assistant:2026.2.1` |
| piper | `:latest` | `rhasspy/wyoming-piper:2.2.2` |
| whisper | `:latest` | `rhasspy/wyoming-whisper:3.1.0` |

For the two on `:latest`, I matched the running container's image digest against Docker Hub's published tags to find which concrete version it actually was, so the first `compose up` recreated a byte-equivalent image instead of silently pulling something newer.

## Folding In Home Assistant and Wyoming

Once pinned, these three were a straight translation from `docker run` flags to compose service blocks. The reassuring part is that **all of their state lives in host bind mounts** — Home Assistant's `/config`, Whisper's model cache — so the cutover was genuinely risk-free: stop and remove the old standalone container, and `docker compose up -d` recreates it pointed at the exact same directory on disk. Nothing to migrate, nothing to lose.

Home Assistant keeps `network_mode: host` and `privileged: true` (it needs both for device discovery and its supervisor-style access), Piper and Whisper stay on the bridge network with their `10200`/`10300` ports published, and HA's Assist voice pipeline reached both of them again on the first boot without any config change.

After the cutover, `docker compose ps` showed five services under one project and **zero stray standalone containers left** — exactly the state I wanted.

## The Update Policy: Automatic, but Not Reckless

This is the part that makes automation safe enough to actually turn on. Dependabot watches every `image:` tag in the compose file and opens a PR per available bump; a CI workflow validates each one (`docker compose config` to catch a broken file, `yamllint`, and a `gitleaks` scan so a real secret can never sneak into what's meant to be a public repo). Then an auto-merge workflow applies a deliberately conservative rule:

- **Patch and minor bumps auto-merge** once CI is green — these are the routine "features and fixes" updates.
- **Every major bump stays open for me to review** — including Ollama, where a major version is the most likely thing to disturb inference.
- **Home Assistant is always manual, regardless of bump size.** HA uses calendar versioning, and its monthly "minor" releases routinely carry documented breaking changes. Treating an HA `2026.2 → 2026.3` as a safe minor would be exactly the kind of "broken service on a random Tuesday" I'm trying to avoid.

That maps cleanly onto the rule I set for myself earlier: automate the things with a well-understood, low-risk blast radius, and keep a human in the loop for anything that could quietly break the family's smart home or the AI stack.

## Containerizing Plex (the Deliberate, Not-Yet-Pulled Trigger)

Plex was the odd one out — a native `.deb` service, not a container at all. Since it isn't in active use right now, I chose the higher-effort-but-cleaner long-term path: bring it into the compose stack too, using the LinuxServer.io image (`lscr.io/linuxserver/plex`). That image was the right call for a few concrete reasons:

- Its `PUID`/`PGID` convention maps straight onto my user's uid/gid `1000`, so the config files stay owned by me.
- It has the best-documented NVIDIA hardware-transcode recipe, which reuses the same Container Toolkit I already installed for Ollama — the 3090 does NVENC for Plex and CUDA for inference (light-load contention between the two is fine).
- Its clean `1.43.2`-style semver tags classify correctly under the patch/minor auto-merge rule, unlike the official image's build-hash tags.

The important honesty here: **the Plex service is authored and validated, but not started.** The native install still owns `:32400` and its ~1 TB library, and moving that data needs interactive `sudo` on the console — not something I'd run blind over SSH at 4 AM. So the container is gated behind a compose `profiles: ["plex-cutover"]` flag, which means the weekly deploy's plain `docker compose up -d` **never** starts it and races the native service for the port. It sits dormant until I explicitly run the cutover, and the full runbook (stop/disable/mask the native service, `rsync` + `chown` the library, bring the container up, verify, and a rollback path that just un-masks the native service) is written down in the infra repo for when I'm ready.

> **Note:** Gating a defined-but-dangerous service behind a compose profile turned out to be the quiet hero of this change. It let me commit the *whole* target state to git today without the automation acting on the one piece that still needs my hands on it.

---

The through-line across these last few posts is the same one every time: consolidate everything into one declarative source of truth, then automate only the parts whose failure I can shrug off. This change extended that from "the two AI services" to "essentially the entire box" — and the one exception, Plex's data migration, isn't unfinished so much as *intentionally waiting for a human*, which is its own kind of done.

---

[← Back to Home](../../README.md)
