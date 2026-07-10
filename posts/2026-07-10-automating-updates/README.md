# Automating OS Updates (and What I Chose NOT to Automate)

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, maintenance, security, updates

---

While going through the security audit, I noticed a backlog of pending `apt` security updates that had just been quietly piling up, with nothing in place to apply them automatically. For a box that's now family-facing, that's not a great place to be — I want security patches landing on their own, but not in a way that could surprise me with a broken service on a random Tuesday morning.

## unattended-upgrades

I installed and configured `unattended-upgrades` to automatically apply security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

In `/etc/apt/apt.conf.d/50unattended-upgrades`, I confirmed the security repo was enabled, and in `/etc/apt/apt.conf.d/20auto-upgrades` I set it to run daily:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

I also enabled an automatic reboot, but scheduled for 4:00 AM:

```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

That way, if a kernel update needs a reboot to take effect, it happens while everyone's asleep instead of mid-Plex-movie or mid-homework-help-from-the-AI.

> **Note:** I deliberately left `Automatic-Reboot-WithUsers` alone (default behavior applies) rather than forcing a reboot with active sessions — the 4 AM window makes that mostly moot anyway.

## Holding the GPU Stack

There's one category of package I explicitly did **not** want auto-updating: the NVIDIA driver and CUDA packages. Those are the single most likely thing to break local inference — a driver bump that's incompatible with the currently-running kernel, or a CUDA version mismatch with the Ollama container, would take down the whole AI stack with no warning. I pinned them:

```bash
sudo apt-mark hold nvidia-driver nvidia-kernel-dkms cuda-toolkit
```

Now `unattended-upgrades` (and a stray `apt upgrade`) will skip those packages entirely until I deliberately unhold and upgrade them myself, ideally right before I'm ready to test that everything still works.

## What Stays Manual, and Why

A few things are staying hands-on for now, on purpose:

- **Container image updates.** The Ollama and Open WebUI images from the new compose stack aren't part of this automation. Auto-pulling a new tag on a family-facing service is too risky without a way to test and roll back first — that's waiting for the compose stack to live in git with a real CI/CD pipeline, which is a good topic for a future post.
- **Model management.** Choosing, pulling, and pruning models is a deliberate decision each time (disk space and VRAM are both finite), not something to run on a timer.
- **GPU driver upgrades.** Same reasoning as the hold above — when it's time to move to a new driver version, I want to do it consciously and verify inference still works afterward.
- **External access.** Setting up the eventual WireGuard VPN for remote access is a manual, one-time project, not an "automate and forget" task.

---

The theme here is that automation is great for things with a well-understood, low-risk blast radius — security patches on a schedule the family won't notice — and a poor fit for anything where a bad update could quietly break AI inference or, worse, open up unexpected access. The container update story is the next piece of that puzzle, once there's a proper pipeline to make it safe.

---

[← Back to Home](../../README.md)
