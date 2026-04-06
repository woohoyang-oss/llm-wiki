# Reddit Post v7

**Title:** I'm a messy vibe coder, keep losing context between devices — so I built my own AI memory layer

**Body:**

I built [Akaxa.space](https://akaxa.space) — a lightweight memory layer that lets any AI save and recall context across conversations. No MCP, no API keys — just read a readme and go.

I'm not a professional developer — just a messy vibe coder who ended up building something out of frustration.

**My problem:** multiple MacBooks, two Claude accounts (work + personal), constantly switching between VS Code and Claude Desktop. The project folder is shared but conversation context isn't — so I'd Tailscale into my other Mac just to grab a session MD I forgot to save. Every. Single. Time.

I tried Notion as a memory layer. It works, but here's what actually happens under the hood:

**Notion + Gmail** → 4 tool schemas load (~4,000 tokens before anything happens), 4 API calls, 4 commands

**Akaxa** → 1 key-value save, done

| | Notion + Gmail | Akaxa |
|---|---------------|-------|
| Tool schemas | ~4,000 tokens | ~200 tokens |
| API calls | 4 | 1 |
| Request/response | ~4,200 tokens | ~1,600 tokens |
| **Total** | **~8,200** | **~1,800** |
| **Reduction** | | **~4.5x fewer** |
| Time | 30s~1min+ | 5~10s |
| Recipient gets it in | Browser (outside AI) | AI conversation |

Think of it like a Post-it note for your AI — stick it, grab it, done. No filing cabinet needed.

In fact, this entire post was drafted, saved to Akaxa, loaded back, reviewed, and revised — all inside the same AI conversation. No browser tabs, no copy-pasting, no context switching.

Notion is great for what it does — editing, comments, version history. Akaxa doesn't do any of that. It's just memory. There are other AI memory tools too (Mem0, Hindsight, etc.) but most require MCP setup. I wanted something that works with just a fetch.

**Don't take my word for it — ask your AI:**

> Read https://akaxa.space/p/DUXVU-YSM5H and execute it

Your AI reads it, reasons through both routes, and shows you the numbers.

https://akaxa.space