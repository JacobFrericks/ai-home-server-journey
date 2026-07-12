# From Dashboards to Pings: Getting Server Alerts on My Phone

**Date:** July 11, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, grafana, alerting, home-assistant, notifications, prometheus

---

When I built the observability stack, I closed that post with an honest gap: the alert rules *evaluated*, but they didn't *notify* anywhere. A disk creeping past 85%, the 3090 running hot, a container stuck in a restart loop — Grafana knew, and Grafana kept it to itself. An alert that only shows up on a dashboard I have to remember to open is really just a slower dashboard. This is the v2 I promised: making those alerts actually reach my phone, still without any cloud service or a byte leaving the house until the very last hop.

## Reuse the notifier I already have

The temptation was to bolt on Alertmanager plus some notification provider. But I already own a perfectly good, reliable push channel: the **Home Assistant Companion app** on my Pixel. It's already installed, already authenticated, and delivers push even when the app is closed. So instead of a new moving part, the design routes Grafana's alerts *through Home Assistant* and out to the phone:

```
Grafana alert rule
  --> contact point "ha-mobile" (webhook)
    --> HA webhook  /api/webhook/grafana_alerts
      --> notify.mobile_app_pixel_9_pro   (push notification)
```

I stuck with **Grafana-managed alerting** — Grafana-managed rules, no separate Alertmanager. For a single-node homelab, one fewer service to run and secure is the right trade, and Grafana's built-in alerting is more than enough to fire a webhook.

## Provisioned, not clicked

Same principle as the dashboards: alerting config is **files on disk**, not state I poke into a UI and lose on the next redeploy. Three YAML files under the provisioning directory Grafana already mounts read-only:

```
monitoring/grafana/provisioning/alerting/
  contactpoints.yaml   # the "ha-mobile" webhook
  policies.yaml        # route everything to it
  rules.yaml           # the alert rules themselves
```

Because that directory is already a `:ro` provisioning mount, adding these needed **no compose change at all** — a restart of Grafana picks them up. The rules are version-controlled and reproducible; a fresh deploy comes up already knowing how to reach me.

## The critical few

The fastest way to make alerting useless is to make it noisy — train yourself to swipe the notification away and you've built nothing. So I deliberately wired only the handful of conditions I'd actually want to be interrupted for, reusing the PromQL and thresholds I'd already written for Prometheus:

| Alert | Condition |
|---|---|
| **DiskAlmostFull** | disk usage > 85% for 10m |
| **GPUTempHigh** | 3090 temp > 80 °C for 5m |
| **TargetDown** | a scrape target `up == 0` for 5m |
| **ContainerRestartLoop** | > 2 restarts in 15m |

I left others on the cutting-room floor on purpose — a swap-usage alert was pure noise on this box, and a VRAM-near-ceiling alert, while tempting, fires often enough during normal inference that it'd cost more attention than it saves. Four alerts I'll trust beats a dozen I'll mute.

## The gotcha: the container that couldn't find home

The webhook is the one place the "everything talks over the internal bridge" model breaks down. Grafana runs on the isolated `monitoring` bridge; Home Assistant runs on the host network. My instinct was to point the contact point at `host.docker.internal:8123` — the same host-gateway trick that lets Prometheus scrape host services.

It didn't resolve. `host.docker.internal` only exists inside containers I *gave* the `extra_hosts: host-gateway` alias to — I'd added it to Prometheus, not Grafana. Rather than sprinkle that alias around, the contact point just uses the host's **LAN IP** directly:

```yaml
# contactpoints.yaml (address redacted)
url: http://<home-server-LAN-IP>:8123/api/webhook/grafana_alerts
```

It's a hair less elegant than a hostname, but it's unambiguous and it works from any container without special DNS wiring.

## A root-owned file, again

The HA side is a single automation, `grafana_alerts_to_mobile`, that catches the webhook and calls `notify.mobile_app_pixel_9_pro`. It lives in `automations.yaml` — which, like most of HA's config tree in this container, is **root-owned**, so my user can't just overwrite it over SSH. (This is a recurring theme in this project: the file permissions are about whatever uid the container runs as, not about me.) I moved the root-owned file aside, wrote the new one, and reloaded automations with a `docker restart homeassistant`, keeping a backup of the original.

## Does it actually fire?

A notification path you haven't tested is a notification path that doesn't work. I fired a test POST at the webhook and confirmed HA recorded an execution trace for `grafana_alerts_to_mobile`, and that all four rules load clean (`inactive` / `ok`) in Grafana. For *reading* the dashboards on the phone there's no native Grafana app worth having, so the "app" is just the Grafana PWA added to my home screen — over mDNS on the home Wi-Fi (`http://homeserver.local:3000`). Getting to it from outside the house is a deliberate non-goal for now; I only wanted to be reachable while I'm home.

---

The shift is small but real: the server went from something I have to *check on* to something that *tells me* when it needs attention — a hot GPU, a filling disk, a flapping container — as a push notification, with the whole evaluation path self-hosted and only the final ping touching Google's FCM. The eyes from last time now come with a voice, and it only speaks up when it matters.

---

[← Back to Home](../../README.md)
