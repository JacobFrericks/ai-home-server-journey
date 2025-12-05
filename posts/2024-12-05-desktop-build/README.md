# Desktop Build Parts List

**Date:** December 5, 2024  
**Author:** Jacob Frericks  
**Tags:** homelab, hardware, build

---

A desktop build was chosen for future expansion capabilities. I didn't want to have to create all new parts for a minor upgrade.

**Purpose:** This server will run Kubernetes (so I can learn it better) and run a lot of containers necessary for the home server. It will also run AI workloads, so it needed to be beefy.

**Budget:** $3000

---

## Case

**CORSAIR 7000D Airflow Full Tower ATX**

**Price:** $250

### Why This Case?

I'm upgrading the current Plex server, which is a mid tower. Unfortunately, a NVIDIA 3090 wouldn't fit in this case. I don't want that to happen again in the future, so I'm getting a large case so I can hopefully reuse the case in the future.

---

## Motherboard

**Gigabyte x870E AORUS Xtreme AI TOP**

**Price:** $800

### Why This Motherboard?

This is one of the only motherboards I could find that would run two GPUs at 8x. Most other motherboards would run two GPUs at 8x/4x. I wanted the flexibility to add a second GPU if I needed more VRAM.

---

## HDD

**4TB NVME**

**Price:** $201

### Why This Storage?

I have two spinning hard drives at 4TB, and found the space to be good. The motherboard has other NVME slots, so I can always expand if necessary. I was lucky, and was able to purchase this before the current HDD shortage.

---

## RAM

**128GB DDR5**

**Price:** $464

### Why This RAM?

This is the max the motherboard allows. I was lucky, and was able to purchase this before the current RAM shortage.

---

## GPU

**NVIDIA 3090**

**Price:** $650 (used)

### Why This GPU?

This is where most of the expense could go, if I'm not careful. The NVIDIA xx90 series are best for AI due to the large amount of VRAM (24GB). However, the 4090 and the 5090 would balloon over my budget. So I settled for an NVIDIA 3090, just because of the memory. I could have gotten a 5070 for about the same price, but it would cut my VRAM in half (12GB). I decided for an older gen with more memory.

---

## OS

**Debian 13 Trixie**

**Price:** $0

### Why This OS?

Most of the servers I deal with are Debian, and research shows that most of what I want to do can run on Debian Linux. I'm no stranger with the OS, so this seemed like the logical choice.

---

## PSU

**Corsair HX1500i**

**Price:** $370

### Why This PSU?

If I were to get a second 3090, I only wanted to purchase the card, and not upgrade any parts. This PSU would allow for that goal.

---

## Total Cost

| Component | Price |
|-----------|-------|
| Case | $250 |
| Motherboard | $800 |
| HDD | $201 |
| RAM | $464 |
| GPU | $650 |
| OS | $0 |
| PSU | $370 |
| **Total** | **$2,735** |

---

[← Back to Home](../../README.md)
