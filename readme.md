# Personal ChatGPT RAG — Chat UI

A retrieval-augmented chat interface built over my own ChatGPT export (3,800+ conversations). Everything — retrieval, embeddings, and generation — runs **locally** on my machine via LM Studio. The web UI is just a thin client that talks to that local backend through a public tunnel.

---

## 🔗 Live URL

**Frontend (chat UI):** [rag_chat_ui.html](https://github.com/yasirsamin821/cloudfare_deployment/blob/main/rag_chat_ui.html)


> ⚠️ This is only reachable while my machine is on and the tunnel is running (see [Uptime notes](#known-limitations) below). If using a quick tunnel, the backend URL changes on every restart — check the terminal output or `~/.cloudflared/config.yml` for the current one, and update the frontend config if it's stale.

---

## How this is deployed

```
┌─────────────────┐      HTTPS       ┌──────────────────┐      localhost      ┌────────────────────┐
│  GitHub Pages    │ ───────────────▶ │  Cloudflare       │ ──────────────────▶ │  FastAPI backend    │
│  (frontend UI)   │                  │  Tunnel           │                     │  (my machine)        │
└─────────────────┘                  └──────────────────┘                     └────────────────────┘
                                                                                          │
                                                                                          ▼
                                                                                 ┌────────────────────┐
                                                                                 │  LM Studio           │
                                                                                 │  - qwen2.5-1.5b-instruct (gen)
                                                                                 │  - granite-embedding-107m-multilingual (embed)
                                                                                 └────────────────────┘
                                                                                          │
                                                                                          ▼
                                                                                 ┌────────────────────┐
                                                                                 │  Chroma vector store │
                                                                                 │  (local, on disk)    │
                                                                                 └────────────────────┘
```

**In short: nothing is hosted on a server or in the cloud.** The models, the vector database, and the retrieval logic all live on my local machine. Cloudflare just opens a secure tunnel from the internet to my `localhost`, and GitHub Pages hosts the static frontend that talks to that tunnel.

### What that means in practice
- The chat UI is only reachable **while my machine is on and the tunnel is running.** If the backend is offline, the frontend will fail to connect (you'll see connection errors or timeouts).
- No conversation data or query leaves my machine except through the tunnel — there's no third-party LLM API involved in generation.
- Retrieval is hybrid: BM25 (keyword) + dense vector search over the ChatGPT export, followed by a `condense_question()` step to fold chat history into a standalone query before hitting the retriever.

---

## Finding the live URL

The public URL changes depending on how the Cloudflare tunnel is started:

- **Named/persistent tunnel** → the URL is fixed (e.g. `https://your-subdomain.example.com`) and won't change between restarts.
- **Quick tunnel** (`cloudflared tunnel --url http://localhost:PORT`) → Cloudflare generates a **new random `*.trycloudflare.com` URL every time it's started.**

To get the current URL:
1. Check the terminal output where `cloudflared` was started — it prints the active URL on launch.
2. If a named tunnel is configured, check `~/.cloudflared/config.yml` for the `hostname` field.
3. The frontend (GitHub Pages) is configured to point at whatever backend URL is set in its config — if the tunnel URL changed, that config needs updating too, or the UI will fail to reach the backend.

> If you're reading this as someone other than me: this is a personal project and the tunnel is only live when I'm actively running it — don't expect 24/7 uptime.

---

## Stack

| Layer | Tech |
|---|---|
| Frontend | Static site (terminal-aesthetic UI) on GitHub Pages |
| Tunnel | Cloudflare Tunnel (`cloudflared`) |
| Backend | FastAPI |
| Generation model | `qwen2.5-1.5b-instruct` via LM Studio |
| Embedding model | `text-embedding-granite-embedding-107m-multilingual` via LM Studio |
| Vector store | Chroma (local) |
| Retrieval | Hybrid BM25 + dense retrieval, with query condensing for multi-turn chat |
| Source data | Personal ChatGPT export, classified/chunked/AST-parsed for code vs. text |

---

## Running it locally (for future me / anyone cloning this)

1. Start LM Studio and load both models (generation + embedding), serving on their configured local ports.
2. Make sure the Chroma vector store directory exists and is populated (see data pipeline scripts).
3. Start the FastAPI backend:
   ```bash
   uvicorn main:app --host 0.0.0.0 --port <PORT>
   ```
4. Start the Cloudflare tunnel pointing at that port:
   ```bash
   cloudflared tunnel --url http://localhost:<PORT>
   ```
5. Copy the printed tunnel URL into the frontend config (if using a quick tunnel) and push/redeploy GitHub Pages if it changed.
6. Open the GitHub Pages URL — the chat UI should now be talking to your local RAG backend.

---

## Known limitations

- **Uptime is manual** — the whole system depends on my laptop and LM Studio being on.
- **Quick tunnel URLs are ephemeral** — anyone bookmarking the chat link may find it dead after a restart unless a named tunnel is set up.
- **CPU-only inference** (Ryzen 7 7700, 16GB RAM, no GPU) — responses may be slower than cloud-hosted equivalents, especially under load.
