# Reddit Post v4

**Title:** Notion vs Akaxa: the real token cost of sharing a document through AI — 4.5x difference

**Body:**

I built Akaxa.space — a lightweight memory layer that lets any AI save and recall context across conversations. No MCP, no API keys — just read a readme and go.

I've been using Notion as a memory layer for my AI workflows — custom skills, MCP config, the whole setup. It works, but I started noticing how much overhead it adds.

So I broke down the cost: share the same handover document two ways and compare the tokens.

**Route A: Notion + Gmail (the "standard" way)**

1. "Save this as a doc" → generate markdown
2. "Upload to Notion" → create-pages API
3. "Make it shareable" → update permissions API
4. "Email the link" → Gmail draft + send API

Before anything even happens, 4 tool schemas load into context — Notion's create-pages schema alone includes block types, property definitions, parent options, and a full markdown spec. That's ~4,000 tokens just to start.

Total: **~8,200 tokens, 4 API calls, 4 commands, 30s~1min+.**

And the recipient? They get a browser link. Not inside their AI conversation — they have to copy-paste it in.

**Route B: Akaxa**

1. "Save to Akaxa"

That's it. One key-value save. ~200 tokens for the tool schema, ~500 for the readme fetch, ~200 for auth, ~900 for the request/response.

Total: **~1,800 tokens, 1 API call, 1 command, 5~10 seconds.**

Think of it like a Post-it note for your AI — stick it, grab it, done. No filing cabinet needed.

The recipient opens any AI, says "load my handover doc from Akaxa" — it's right there in the conversation, ready to work with.

**Side by side:**

| | Notion + Gmail | Akaxa |
|---|---------------|-------|
| Tool schemas | ~4,000 tokens | ~200 tokens |
| API calls | 4 | 1 |
| Request/response | ~4,200 tokens | ~1,600 tokens |
| **Total tokens** | **~8,200** | **~1,800** |
| **Reduction** | | **~4.5x fewer** |
| Commands from you | 4 | 1 |
| Time | 30s~1min+ | 5~10s |
| User interactions | 4 round trips | 1 round trip |
| Recipient gets it in | Browser (outside AI) | AI conversation |

To be clear — Notion is great for what it does. Editing, comments, version history, team collaboration. Akaxa doesn't do any of that. It's just memory — save and recall, that's it. There are other dedicated AI memory tools too (Mem0, Hindsight, etc.) but most require MCP setup. I wanted something that works with just a fetch.

**Don't take my word for it — ask your AI.**

I published a test doc that any AI can read and estimate the token costs themselves. Just paste this into any conversation:

**→ Read https://akaxa.space/p/DUXVU-YSM5H and execute it**

No install. No API key. No setup. Your AI reads it, reasons through both routes, and shows you the numbers.

https://akaxa.space