# Reddit Post v2

**Title:** I built a memory layer that works across ChatGPT, Claude, and any AI — no API keys, no setup

**Body:**

I kept running into the same problem: every time I started a new conversation, my AI had no idea what we talked about last time.

I tried using Notion as a memory layer — custom GPT actions, MCP config, database setup, permissions. It works, but it's heavy. The API responses alone eat up tokens with nested blocks, metadata, and formatting. And you have to set it up separately for each AI.

So I built Akaxa.space — a persistent memory layer designed specifically for AI agents.

**How it's different:**

| | Notion / Docs | Akaxa |
|---|---------------|-------|
| Built for | Human workspace | AI memory |
| Setup | Integration + DB + permissions + MCP | Star name, done |
| Data model | Pages / blocks / properties | Key-value |
| Cross-AI | MCP config per AI | Auth once, use anywhere |
| Token cost | Heavy — nested blocks, metadata, rich formatting | Minimal — plain key-value |

**How it works:**
- Your AI asks for your email or nickname
- You verify with a "star name" (like a password, but cooler)
- Save and recall memories from any AI, any conversation

No API keys. No OAuth. No database setup. Just save("key", "value") and load("key").

Think of it like a library — you drop off a book in one conversation, pick it up in another. Doesn't matter if you're on ChatGPT, Claude, or something else entirely.

Data is encrypted at rest (AES-256-GCM) and in transit (TLS 1.3).

**Connect in one line:**
claude mcp add --transport http akaxa https://akaxa.space/mcp

Or use the REST API from any AI that supports actions/functions.

I'd love feedback — what would make this useful for your workflow?

https://akaxa.space

---

# FAQ

**Q: How is this different from Claude Projects or ChatGPT memory?**
A: Those are per-AI, per-project. Akaxa works across any AI and any conversation. Save in GPT, recall in Claude — or vice versa.

**Q: I already use Notion for this. Why switch?**
A: You don't have to — Notion is great for structured workspace stuff. But for pure AI memory, it's overkill. Notion API responses are token-heavy (nested blocks, metadata, properties), and you need MCP setup per AI. Akaxa is key-value, lightweight, and works everywhere with one auth.

**Q: Can't I just paste context into instructions every time?**
A: You can, and some people do. But that's manual — summarize, copy, paste, repeat. Akaxa automates that: save once, recall anywhere, no copy-pasting.

**Q: Is my data safe?**
A: Encrypted at rest (AES-256-GCM, per-user keys) and in transit (TLS 1.3). Auth via star name or email OTP — no passwords stored. Note: this is server-side encryption, not end-to-end. E2EE is on the roadmap.

**Q: What AIs does it work with?**
A: Anything. Claude (MCP), ChatGPT (custom GPT actions), or any AI that can make HTTP requests.

**Q: Is it free?**
A: Yes, currently free while in early access.