# Locking Down the Server: SSH, Firewall, and Secrets

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, security, ssh, firewall

---

The server has quietly graduated from "thing I tinker with on weekends" to "thing my family actually depends on" — Open WebUI, Home Assistant, and Plex are all real services now, used by real people every day. That's exactly the moment to stop and do a proper security audit, before I ever think about exposing anything to the outside world. I went through the box top to bottom and found several gaps worth closing.

## SSH Hardening

I was still SSHing in with a password, which is a habit from the "it's just a lab box" days that needed to go. First, I generated a dedicated ed25519 key just for this server:

```bash
ssh-keygen -t ed25519 -C "homeserver-access" -f ~/.ssh/homeserver_ed25519
```

Why a dedicated key instead of reusing one I already had? If it's ever compromised, or I want to revoke access from a device, I can kill this one key without touching access to anything else.

I installed it with `ssh-copy-id`:

```bash
ssh-copy-id -i ~/.ssh/homeserver_ed25519.pub jacob@192.168.86.63
```

Then added an alias to `~/.ssh/config` so I don't have to remember an IP or a key path ever again:

```
Host homeserver
    HostName 192.168.86.63
    User jacob
    IdentityFile ~/.ssh/homeserver_ed25519
```

Now it's just `ssh homeserver`. With the key confirmed working, I disabled password authentication entirely in `/etc/ssh/sshd_config`:

```bash
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

> **Note:** Test the key-based login from a *second* terminal before you restart `ssh` or log out. If something's wrong with the key setup, you want a still-open session to fix it from, not a locked-out box.

I verified password logins are now refused (`Permission denied (publickey)`), which is exactly what I wanted to see.

---

## A Stable IP Without a Static Config

The server had been on DHCP this whole time, which meant its IP could theoretically shift on a lease renewal — annoying for an SSH alias, and a real problem the moment other services start pointing at it by IP. My instinct was to just set a static IP on the machine, but I went a different route: I reserved its address in the router instead.

Since I'm running Google Wifi, this was done through the Google Home app: **Devices → the server → reserve IP**, binding the wired NIC's MAC address to `192.168.86.63`.

Why reserve it at the router instead of hardcoding it on the box? The router is the single source of truth for the network either way — if I set a static IP on the machine *and* the router's DHCP pool didn't know to avoid it, I'd be one unlucky lease away from an IP conflict. Letting the router hand out the same address every time avoids that entirely, and the server's own network config stays simple.

I rebooted the server to confirm it came back up with the same address, and it did.

---

## Firewall: From Nothing to Deny-by-Default

This was the biggest gap. The server had **no firewall at all** — every port any service opened was reachable by anything on the LAN. Fine when it was just me experimenting; not fine now.

I installed `ufw` and set a deny-by-default posture:

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Then I allowed only the ports actually in use on the LAN:

```bash
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 8080/tcp      # Open WebUI
sudo ufw allow 8123/tcp      # Home Assistant
sudo ufw allow 32400/tcp     # Plex
sudo ufw allow 10200/tcp     # Wyoming voice (STT)
sudo ufw allow 10300/tcp     # Wyoming voice (TTS)
sudo ufw enable
```

> **Note:** Ollama's own port, `11434`, was deliberately **left closed**. Ollama ships with no authentication of any kind — anyone who can reach that port can run inference, pull models, and burn GPU time. It only needs to be reachable from `localhost` (Open WebUI and Home Assistant both talk to it via `127.0.0.1`), so there's no reason to open it on the LAN interface at all.

---

## Open WebUI Was Signing Sessions With an Empty Key

While auditing the Open WebUI container config, I found something I should have caught on day one: `WEBUI_SECRET_KEY` was unset, meaning session tokens were being signed with an empty key. Not great for something that gates access to the whole family's chat history.

I generated a proper random secret:

```bash
openssl rand -hex 32
```

and set it as `WEBUI_SECRET_KEY` in the container's environment, along with disabling open signups so the login page can't be used to self-register a new account.

> **Note:** The real secret lives in a server-local `.env` file, never committed to git. Anything in this post referencing it is a placeholder.

---

Remote access — reaching any of this from outside the house — is still completely off, and it's going to stay that way until it goes through a self-hosted WireGuard VPN. I have no intention of ever exposing Open WebUI, Home Assistant, or Plex directly to the internet; a VPN means the attack surface stays exactly one endpoint, one that I control.

---

[← Back to Home](../../README.md)
