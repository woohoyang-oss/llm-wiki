# Why I built Akaxa.space

## The problem
Every AI conversation starts from zero. You explain your stack, your preferences, your context — again and again. I tried solving this with Notion: custom skills, MCP config, database setup, permissions. It works, but it is heavy.

## What made it heavy
- 4 tool schemas just to share a document (~4,000 tokens loaded before anything happens)
- 4 API calls: create page → update permissions → draft email → send
- Notion API responses are bloated with nested blocks, metadata, and rich formatting
- Setup per AI: each AI needs its own MCP config or integration
- The document lands in a browser, not inside the AI conversation

## What Akaxa does differently
One command. One API call. Done.
- save("key", "value") — plain key-value, no page/block overhead
- Auth with a star name or 4-digit PIN — no API keys, no OAuth
- Works across any AI — save in GPT, load in Claude, or vice versa
- The document lands inside the conversation, ready to work with
- Just tell your AI: "read akaxa.space/readme" — no install, no config

## Side by side: sharing a handover doc

| | Notion + Gmail | Akaxa |
|---|---------------|-------|
| Tool schemas loaded | ~4,000 tokens | ~200 tokens |
| API calls | 4 | 1 |
| Request/response | ~4,200 tokens | ~900 tokens |
| **Total tokens** | **~8,200** | **~1,100** |
| **Reduction** | | **~7.5x fewer tokens** |
| Commands from you | 4 | 1 |
| Recipient gets it in | Browser (outside AI) | AI conversation (ready to use) |
| Recipient needs | Email + Notion account | Any AI + star name |

## Try it yourself
Ask your Claude right now:

> Fetch akaxa.space/readme and follow the instructions to save something for me.

No install. No API key. No config. Just one sentence.

https://akaxa.space