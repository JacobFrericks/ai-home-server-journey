# Giving the Local AI Internet Access: Self-Hosted SearXNG over MCP

**Date:** July 11, 2026  
**Author:** Jacob Frericks  
**Tags:** homelab, ai, ollama, home-assistant, searxng, mcp, web-search, privacy

---

A local LLM has one deeply annoying limitation: it only knows what it was trained on. Ask my home assistant "who won the most recent F1 race?" or "what's the current weather?" and it would confidently answer with something months or years stale — or just make it up. The whole appeal of running the model at home is that it *doesn't* phone a cloud provider, but that same isolation means it's frozen in time. I wanted to fix that without undoing the privacy: let the model reach the internet itself, on my terms, with no third-party search API and no query ever tied to an account.

The answer turned out to be two small self-hosted pieces — a private metasearch engine and a tool server that hands it to the model — wired so that **both** of my AI surfaces (the Open WebUI chat and the Home Assistant voice assistant) reach the web the exact same way.

## The shape of it

```
gemma4:31b  --(tool call)-->  searxng-mcp  --->  SearXNG  --->  public search engines
             searxng_web_search   127.0.0.1:9200/mcp   127.0.0.1:8888
```

- **SearXNG** is a self-hosted metasearch engine. It queries the public search engines on my behalf and returns aggregated results — but *I* run it, there's no API key, and no account sees my queries.
- **searxng-mcp** is a small [MCP](https://modelcontextprotocol.io/) server that wraps SearXNG's API as a set of *tools* (`searxng_web_search`, `searxng_search_suggestions`, `searxng_instance_info`, `web_url_read`). MCP is the emerging standard for handing tools to an LLM, and — crucially — it's the same protocol whether the caller is a chat UI or a smart-home voice agent.

Both pieces bind **loopback only** (`127.0.0.1:8888` and `127.0.0.1:9200`), so nothing new is exposed to the LAN and no firewall rule changes. The only traffic that leaves the house is SearXNG → the public engines, exactly as if I'd typed the search myself.

## SearXNG: two settings that matter

SearXNG mostly just works, but two config values were non-obvious enough to cost me time:

```yaml
# searxng/settings.yml
search:
  formats: [html, json]   # without json, the API returns HTTP 403
server:
  limiter: false          # single local user — no bot limiter, no redis sidecar
```

The **`json` format is mandatory** here — SearXNG serves an HTML UI by default and returns a flat **403** to programmatic JSON requests unless you explicitly opt in. And turning the **rate limiter off** is what lets me skip a whole redis/valkey sidecar; the limiter exists to stop bots hammering a public instance, which is meaningless for a one-user loopback service.

One more subtlety worth writing down: `settings.yml` holds a real `server.secret_key`, so it's **git-ignored** and lives only on the server. The image only auto-generates a secret when the file is *absent* — since I bind-mount my own, I own the secret, and because it's git-ignored it survives the weekly `deploy.sh` git reset instead of being clobbered on every pull. The tracked template is `settings.yml.example`.

## One path, no toggle

Open WebUI actually ships its own built-in web-search feature, and I used it first. But it lives behind a per-chat toggle, and it's a *different* mechanism than anything Home Assistant could use. I didn't want two implementations and a switch I'd forget to flip. So I tore it out and standardized on MCP for both surfaces — **"MCP for everything, no toggle."**

In **Open WebUI**, that meant registering searxng-mcp as an external MCP (Streamable HTTP) tool at `http://127.0.0.1:9200/mcp`, then attaching that tool to the `gemma4:31b` workspace model as a default so it rides along on *every* chat with no per-conversation toggle. Native web search is switched off (`web.search.enable = false`) so there's exactly one path. The model uses ordinary function-calling to decide when to reach for it.

## The Home Assistant wiring — and the gotcha that cost me an hour

Home Assistant's MCP Client integration points at the same `/mcp` URL. HA tries the streamable-HTTP transport first and only falls back to SSE, so it talks to searxng-mcp directly — no SSE bridge or proxy needed, which most older guides insist on.

The part that stumped me: adding the MCP integration did nothing. The voice agent still couldn't search. The reason is a genuinely sharp edge in how HA exposes tools to a conversation agent:

> MCP registers its **own** LLM tool API, named `mcp-<entry_id>` — and that API is **not** part of the built-in `assist` API. So even with the integration loaded, the Ollama conversation agent can't see the search tool until you add that `mcp-<entry_id>` id to the agent's `llm_hass_api` **alongside** `assist`.

Once the conversation agent's `llm_hass_api` listed **both** `assist` and `mcp-<entry_id>`, the tool showed up and the model could call it. Only `gemma4:31b` is wired for this — it's the tool-capable model, and per my one standing rule I left the other model completely alone.

## Teaching it *when* to search

Tools available isn't the same as tools used. My first end-to-end test worked only when I made freshness explicit — ask "what's the *latest* Home Assistant version" and it searched; ask "what version of Home Assistant is current" and it happily answered from stale memory. A model won't reach for a tool it doesn't think it needs.

The fix was a one-line nudge in the conversation agent's system prompt: for anything about current events, news, weather, prices, or otherwise uncertain, **search instead of answering from memory**. That flipped the behavior. The proof I was after came from a question I *didn't* prime at all:

> **Me:** "Who won the most recent F1 race?"  
> **HA (debug log):** `tool_calls=[searxng_web_search(query="who won the most recent Formula 1 race")]` → answered from live results.

It decided, on its own, that this was a "go look it up" question — which is exactly the judgment I wanted it to have.

---

What I like about this build is that it's the same two components, one privacy posture, and one code path serving two completely different front-ends — a chat window and a voice assistant. The model reaches the live web when it decides it needs to, every query is proxied through something I run, and nothing leaves the house except the searches themselves. The frozen-in-time problem is gone, and I didn't have to trade away the reason I built this in the first place.

---

[← Back to Home](../../README.md)
