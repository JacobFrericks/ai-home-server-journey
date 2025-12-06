# Desktop Build Parts List

**Date:** December 5, 2024  
**Author:** Jacob Frericks  
**Tags:** homelab, hardware, build

---

A desktop build was chosen for future expansion capabilities. I didn't want to have to create all new parts for a minor upgrade.

**Purpose:** This server will run Kubernetes (so I can learn it better) and run a lot of containers necessary for the home server. It will also run AI workloads, so it needed to be beefy.

**Budget:** $3500

---

## Case

**CORSAIR 7000D Airflow Full Tower ATX**

**Price:** $250

### Why This Case?

My current Plex server uses a mid tower case that couldn't accommodate an NVIDIA 3090. To avoid running into size constraints again, I chose this full tower case for its generous internal space, ensuring compatibility with large GPUs and future upgrades.

---

## CPU

**AMD Ryzen 9950x3D**

**Price:** $650

### Why This CPU?

I want to be able to handle Plex videos, AI, and k8s workloads at the same time

---


## Motherboard

**Gigabyte x870E AORUS Xtreme AI TOP**

**Price:** $800

### Why This Motherboard?

This motherboard stands out because it can run two GPUs at 8x PCIe lanes each, whereas most alternatives only offer 8x/4x configurations. This flexibility is important if I ever need to add a second GPU for additional VRAM.

---

## HDD

**4TB NVME**

**Price:** $200

### Why This Storage?

Based on my experience with two 4TB spinning drives, I found this capacity meets my needs well. The motherboard includes additional NVME slots for future expansion, and I was fortunate to purchase this drive before the current storage shortage.

---

## RAM

**128GB DDR5**

**Price:** $465

### Why This RAM?

I maxed out the motherboard's supported RAM capacity to ensure plenty of headroom for running multiple containers and AI workloads. Fortunately, I purchased this before the current RAM shortage drove prices up.

---

## GPU

**NVIDIA 3090**

**Price:** $650 (used)

### Why This GPU?

The GPU is where costs can quickly spiral out of control. NVIDIA's xx90 series cards are ideal for AI workloads thanks to their 24GB of VRAM, but newer models like the 4090 and 5090 would exceed my budget. While a 5070 costs about the same as a used 3090, it only offers 12GB of VRAM—half the memory. I prioritized VRAM capacity over newer architecture.

---

## OS

**Debian 13 Trixie**

**Price:** $0

### Why This OS?

Debian is my go-to server operating system, and my research confirmed that everything I want to run is well-supported on Debian Linux. Given my familiarity with the OS, it was the natural choice.

---

## PSU

**Corsair HX1500i**

**Price:** $370

### Why This PSU?

I selected a high-wattage power supply to accommodate a potential second 3090 in the future without needing to upgrade any other components.

---

## Total Cost

| Component | Price |
|-----------|-------|
| Case | $250 |
| CPU  | 650 |
| Motherboard | $800 |
| HDD | $201 |
| RAM | $464 |
| GPU | $650 |
| OS | $0 |
| PSU | $370 |
| **Total** | **$3,385** |

---

[← Back to Home](../../README.md)
