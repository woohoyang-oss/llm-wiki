## r/ClaudeAI

**Title:** How I gave Claude persistent memory with a simple readme page

**Body:**

I wanted Claude to remember things across conversations without any setup. Here's what I built and how it works.

**The problem:** Every conversation starts from zero. You re-explain your stack, your preferences, your project context — every single time.

**The approach:** I made [akaxa.space](https://akaxa.space) — a memory layer that any AI can use just by reading a readme page.

**How it works:**

Just tell Claude: "Read akaxa.space/readme" — no MCP install, no API key, no config. Claude reads the page, picks up the auth flow, and can save/load memories from that point on.

> "Remember that my deploy target is EC2 and we use Postgres."
> Done. Next conversation, next week, it's still there.

> "What did I save?" → Claude shows a numbered list → pick a number → done.

**Why not GitHub or Notion?** You could store things there, but you'd still copy-paste between tabs. This lives inside the conversation — Claude saves and loads directly. You never leave the chat.

**How auth works (no passwords):** Instead of passwords, users get a star name — a constellation name like Vela. It's fuzzy-matched and works in any language. Forgot it? OTP to your email.

**Security:** AES-256-GCM encryption per user. Even the server operator can't read your data.

**Tech details for anyone curious:**
- Backend: Hono + PostgreSQL + Redis
- Encryption: AES-256-GCM, per-user derived keys
- Also available as MCP server (`https://akaxa.space/mcp`) and [REST API](https://akaxa.space/docs)

Beta and free — happy to answer questions about the implementation.