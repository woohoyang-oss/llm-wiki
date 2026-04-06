# Akaxa.space Session Log — 2026-03-21

## What was built
- Akaxa.space: persistent cross-AI memory service
- Stack: Hono + PostgreSQL + Redis + Next.js + Three.js on EC2 t3.small
- MCP server for Claude + REST API for any AI
- Three.js universe visualization with real-time feed

## Key features implemented
- remember/recall/forget/share/explore (MCP tools)
- Cross-AI auth: email + star name (fuzzy matching, multilingual)
- Nickname support (우호 = wooho)
- OTP email via Resend (akaxa.space domain verified)
- Vault API: save/load/list/search
- Real-time WebSocket feed (Redis pub/sub)
- AES-256-GCM encryption per user
- Levenshtein fuzzy star name matching (포넥스 = Fornax)

## Infrastructure
- EC2 t3.small (3.34.238.104)
- SSL: Let's Encrypt (auto-renew)
- CI/CD: GitHub Actions → EC2 deploy
- PM2 process manager
- nginx reverse proxy

## SEO & Discovery
- Google Search Console: verified + sitemap submitted
- Smithery: registered (akaxa/memory)
- Glama.ai: glama.json added
- awesome-mcp-servers: PR #3609 submitted
- llms.txt, robots.txt, sitemap.xml, Schema.org JSON-LD
- OG image: dynamic generation via Next.js edge

## Auth flow
1. email/nickname → auth/start
2. star name (fuzzy) → auth/verify → session_token
3. save/load with token
4. forgot star name → OTP email

## Favicon: v5 — bold A + asymmetric stars (dark bg)

## Custom GPT setup ready
- OpenAPI spec: https://akaxa.space/openapi-gpt.json
- Name: Akaxa Memory
- Instructions + system prompt prepared

## User account
- Email: wooho.yang@tbnws.com
- Star name: ✦ Fornax
- Nickname: 우호

## Pending
- Custom GPT registration (manual, ChatGPT web)
- Reddit posts (r/ClaudeAI, r/ChatGPT, r/mcp)
- Bing Webmaster
- mcp.so, PulseMCP submission
- awesome-mcp-servers PR approval