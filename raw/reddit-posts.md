# Reddit Posts for Akaxa.space (v3)

## Post 1: r/ClaudeAI, r/ChatGPT, r/artificial

**Title:** I built a memory layer for AI — just say "remember this" and it stays forever.

**Body:**

Every AI conversation starts from zero. You explain the same thing again and again.

So I built [akaxa.space](https://akaxa.space) — a place where your AI can actually remember things.

**How to start:**

Just tell any AI: "Read akaxa.space/readme" — that's it. No install, no API key, no config.

**Then it's just conversation:**

> "Remember that my deploy target is EC2 and we use Postgres."
> Done. Next conversation, next week, it's still there.

> "What did I save?" → AI shows a numbered list → pick a number → done.

No copy-paste. No files. No setup. Just talk.

**Star names** replace passwords. You get a constellation name like Vela. Works in any language. Forgot it? OTP to your email.

**Security:** AES-256-GCM encryption per user. Even the server operator can't read your data.

*Also available:* MCP server (`https://akaxa.space/mcp`), [Smithery](https://smithery.ai/servers/akaxa/memory), [REST API](https://akaxa.space/docs)

Beta. Free. Feedback welcome.

What would you want your AI to remember?

---

## Post 2: r/mcp

**Title:** [Show] Akaxa — MCP server that gives AI agents persistent encrypted memory

**Body:**

Built an MCP server so AI agents can remember things between conversations.

**Easiest way to try:** Tell any AI with web access: "Read akaxa.space/readme" — it picks up the API and auth flow automatically.

**Tools:** remember, recall, forget, share, explore + auth flow (auth_start, auth_verify, vault_save, vault_load, vault_search)

**Auth (no API keys):**
1. `auth_start` with email or nickname
2. `auth_verify` with star name (fuzzy matched, multilingual — Vela = 벨라)
3. `vault_save/load` with session token

Forgot star name? OTP via email.

**Stack:** Hono + PostgreSQL + Redis + Next.js + Three.js
**Encryption:** AES-256-GCM per user, HMAC-SHA256 key hashing
**Transport:** Streamable HTTP

MCP endpoint: `https://akaxa.space/mcp`
Smithery: [smithery.ai/servers/akaxa/memory](https://smithery.ai/servers/akaxa/memory)
Docs: [akaxa.space/docs](https://akaxa.space/docs)

---

## Post 3: r/SideProject, r/webdev

**Title:** I gave AI agents a place to store memories — here's how I built it

**Body:**

I kept losing context between AI conversations. Explain the stack, re-share the schema, re-describe the goal — every single time.

So I built [akaxa.space](https://akaxa.space) — persistent memory for AI agents. Say "remember this," and it stays. Say "recall," and it comes back. Even in a new conversation.

**The simplest way to use it:** Tell any AI "Read akaxa.space/readme" — no install, no config. The AI reads the instructions and handles everything.

**Tech stack:**
- Backend: Hono
- DB: PostgreSQL + Redis
- Frontend: Next.js + Three.js
- Auth: Email + star name + OTP fallback (Resend)
- Encryption: AES-256-GCM per user
- MCP server for Claude integration

**What I learned:**
- A simple readme page is enough for any AI to discover and use your service — no SDK needed
- Star name auth is surprisingly fun — people remember Lyra better than passwords
- The real pain isn't switching AIs — it's that every conversation starts from scratch
- MCP is powerful but only works in Claude Code/Desktop, not Claude.ai web

Beta, free: [akaxa.space](https://akaxa.space)

---

## Subreddits:
1. r/ClaudeAI — Post 1
2. r/ChatGPT — Post 1
3. r/artificial — Post 1
4. r/mcp — Post 2
5. r/SideProject — Post 3
6. r/webdev — Post 3