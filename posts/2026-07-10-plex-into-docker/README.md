# Plex Joins the Stack: Containerizing a 1 TB Library Without Losing It

**Date:** July 10, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, docker, plex, migration, gpu

---

In the last post I said Plex's migration "isn't unfinished so much as intentionally waiting for a human." This is me being that human. Plex was the last service still living outside the Compose stack — a native `.deb` install updating itself through `apt`, with a containerized replacement already written but deliberately gated off so it couldn't start by accident. Time to actually flip it.

I picked the [LinuxServer.io](https://docs.linuxserver.io/images/docker-plex/) image (`lscr.io/linuxserver/plex`) for a few concrete reasons: its `PUID`/`PGID` convention maps cleanly onto my user's uid/gid `1000`, it has the best-documented NVIDIA hardware-transcode recipe (which reuses the Container Toolkit I'd already installed for Ollama, so the 3090 does NVENC for Plex and CUDA for inference), and its clean `1.43.2`-style tags play nicely with Dependabot.

## The Surprise: Where the Media Actually Lived

My first draft of the Plex service bind-mounted `/media` into the container. That turned out to be wrong in a way that would have quietly broken everything: `/media` was **empty**. My actual ~1 TB library (Movies and TV Shows) lived *inside* the native Plex data directory itself — `/var/lib/plexmediaserver/Library/Movies` and `/TVShows`. And critically, Plex's database records the absolute path of every file. If I'd started the container with the media somewhere new, Plex would have come up with every library showing as "unavailable."

The clean fix was to stop trying to relocate anything and instead **bind-mount the media at its original absolute path**:

```yaml
volumes:
  - /home/jacob/docker/plex/config:/config
  - /var/lib/plexmediaserver/Library/Movies:/var/lib/plexmediaserver/Library/Movies
  - /var/lib/plexmediaserver/Library/TVShows:/var/lib/plexmediaserver/Library/TVShows
```

Because the path inside the container is identical to the path the database recorded, every library entry resolves with zero re-matching — no "fix match," no re-scan, watch history intact.

## The Empty-Init Trick

The other thing I didn't want to guess at was *where* inside `/config` the container expects its database. Rather than assume, I started the container once against an **empty** config directory and looked at the skeleton it created:

```
/config/Library/Application Support/Plex Media Server/
```

So the LinuxServer image nests the data one level deeper than I'd have guessed. Knowing that for certain, I stopped the container, `rsync`'d the native `Plex Media Server` folder (the ~1.4 GB database and preferences — the media is separate) into exactly that path, and `chown`'d it to `1000:1000`.

> **Note:** Discovering the layout empirically took about thirty seconds and saved me from a subtle failure where Plex starts with an empty database sitting next to my real one, ignored. When a path convention matters, it's worth letting the software tell you rather than trusting a forum post (including my own earlier notes, which had this detail wrong).

## The Cutover

With the data in place, the actual switch was small: stop, disable, and mask the native `plexmediaserver.service` so it couldn't grab port 32400, then bring up the container. Because I'd copied the original `Preferences.xml` wholesale, the server kept its identity — it came back **already claimed to my account**, no re-claim needed. A quick check confirmed the important things:

- `/identity` reported `claimed="1"`.
- Both libraries resolved: **189 movies, 21 TV shows**.
- `/dev/nvidia0` and `/dev/dri/renderD129` were present inside the container, so hardware transcode works.

## Removing the Native Install (Carefully)

The final step was cleaning up the native package — and this is where the "keep the media where it is" constraint got interesting. My media now lived *inside* the native package's own data directory. So I read the package's removal script before running anything, and found this:

```sh
if [ "$1" = "purge" ]; then
  ...
  rm -rf /var/lib/plexmediaserver
```

That `rm -rf` — the one that would delete my entire library — only fires on **`purge`**. A plain `apt remove` leaves the data directory alone. So:

```bash
sudo apt-get remove plexmediaserver   # NOT --purge
```

removed the binaries and the systemd unit while leaving `Library/Movies` and `Library/TVShows` untouched. I then cleaned up two things the package's own script missed (its `postrm` has a typo that skips deleting the apt source, and my earlier `mask` left a dangling symlink), and reclaimed the 1.4 GB of now-redundant native database — the container has its own copy.

> **Note:** `apt purge plexmediaserver` would delete the library. That's now written in a warning right next to the service definition in the Compose file, because the one-word difference between `remove` and `purge` is a 1 TB mistake.

---

The stack is now genuinely one Compose file — every service on the box, Plex included, described in one place, updated through the same pipeline. The lesson that kept recurring in this migration is that the safe path is almost always the boringly literal one: bind the media where it already is, ask the software where its files go, and read the uninstall script before you run it. Nothing here was clever — it was just careful, and for a family's 1 TB of movies, careful is exactly the right amount of clever.

---

[← Back to Home](../../README.md)
