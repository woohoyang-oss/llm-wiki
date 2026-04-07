# Claudex

### OpenClaude + Codex = **Claudex**

> Claude Code's powerful agentic tools, powered by **GPT-5.4 (Codex)**, Ollama, or any LLM.

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║   ┌──────────────┐         ┌──────────────┐                  ║
║   │  OpenClaude   │    +    │    Codex     │   =   Claudex   ║
║   │ (Claude Code  │         │  (GPT-5.4)  │                  ║
║   │   tool system)│         │             │                  ║
║   └──────────────┘         └──────────────┘                  ║
║                                                              ║
║   All of Claude Code's tools — Bash, Read, Write, Edit,      ║
║   Grep, Glob, Agent, MCP — powered by the model you choose.  ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

Claudex is a fork of [OpenClaude](https://github.com/Gitlawb/openclaude) with patches for:
- **Codex API strict schema compatibility** — GPT-5.4 via ChatGPT subscription ($20/mo)
- **Reasoning model support** — qwen3, DeepSeek-R1, and other thinking models

---

## How It Works

```
┌─────────────────────────────────────────────┐
│         Claude Code Tool System             │
│  Bash · Read · Write · Edit · Grep · Glob   │
│  Agent · MCP · LSP · Tasks · Memory         │
└──────────────────┬──────────────────────────┘
                   │
            ┌──────▼──────┐
            │  openaiShim  │   ← Anthropic ↔ OpenAI format translation
            │  codexShim   │   ← Codex Responses API adapter
            └──────┬──────┘
                   │
       ┌───────────┼───────────────┐
       ▼           ▼               ▼
   Codex API    Ollama        OpenAI API
   (GPT-5.4)   (qwen3,       (gpt-4o,
               llama, etc)    etc)
```

Claude Code doesn't know it's talking to a different model.

---

## Quick Start — Terminal

### 1. Install

```bash
# Install Bun (if not installed)
curl -fsSL https://bun.sh/install | bash

# Clone and build
git clone https://github.com/woohoyang-oss/claudex.git
cd claudex
bun install
bun run build
```

### 2. Set Up Codex Auth (one-time)

```bash
# Install Codex CLI
npm install -g @openai/codex

# Login with your ChatGPT account (opens browser)
codex login
```

> Requires ChatGPT Plus ($20/mo) or Pro ($200/mo) subscription.

### 3. Run

```bash
# Codex (GPT-5.4) — recommended
CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node dist/cli.mjs
```

That's it. All tools, streaming, multi-step reasoning — everything works.

### Shell Alias (recommended)

```bash
# Add to ~/.zshrc or ~/.bashrc
echo 'alias claudex="CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node ~/claudex/dist/cli.mjs"' >> ~/.zshrc
source ~/.zshrc

# Now just type:
claudex
```

---

## Quick Start — VS Code

Use Claudex inside VS Code's integrated terminal. Full agentic coding with GPT-5.4, right in your editor.

> **Note:** The Claude Code VS Code extension requires Anthropic login and cannot be bypassed.
> Instead, Claudex runs in VS Code's built-in terminal — same workflow, no Anthropic account needed.

### Step 1: Complete the Terminal Setup Above

Make sure you've done the Terminal quick start (clone, build, `codex login`, alias).

### Step 2: Use in VS Code

1. Open your project: `code ~/your-project`
2. Open the integrated terminal: **Ctrl+`** (backtick)
3. Type `claudex` and start coding!

```
┌─────────────────────────────────────────────────┐
│  VS Code                                        │
│  ┌─────────────────────────────────────────────┐ │
│  │  your-project/src/app.ts        (editor)    │ │
│  │  ...                                        │ │
│  ├─────────────────────────────────────────────┤ │
│  │  TERMINAL                                   │ │
│  │  $ claudex                                  │ │
│  │  ╭── Claude Code v0.1.4 ──────────────────╮ │ │
│  │  │  codexplan · GPT-5.4                   │ │ │
│  │  ╰───────────────────────────────────────╯ │ │
│  │  ❯ fix the bug in app.ts                   │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

Claudex can read and edit any file in your VS Code workspace, run terminal commands, and use all Claude Code tools — just powered by GPT-5.4 instead of Claude.

### Switching Models

```bash
# GPT-5.4 (best quality)
OPENAI_MODEL=codexplan claudex

# GPT-5.3 Codex Spark (faster)
OPENAI_MODEL=codexspark claudex
```

---

## Other Providers

### Ollama (local, free)

```bash
CLAUDE_CODE_USE_OPENAI=1 \
OPENAI_BASE_URL=http://localhost:11434/v1 \
OPENAI_MODEL=qwen3:14b \
OPENAI_API_KEY=ollama \
node dist/cli.mjs
```

### OpenAI API

```bash
CLAUDE_CODE_USE_OPENAI=1 \
OPENAI_API_KEY=sk-... \
OPENAI_MODEL=gpt-4o \
node dist/cli.mjs
```

### DeepSeek / Groq / Together / Mistral / OpenRouter

```bash
CLAUDE_CODE_USE_OPENAI=1 \
OPENAI_API_KEY=... \
OPENAI_BASE_URL=https://api.deepseek.com/v1 \
OPENAI_MODEL=deepseek-chat \
node dist/cli.mjs
```

Any OpenAI-compatible API endpoint works.

---

## What's Included

| Feature | Status |
|---|---|
| All tools (Bash, Read, Write, Edit, Grep, Glob, Agent, MCP) | ✅ |
| Streaming | ✅ |
| Multi-step tool chains | ✅ |
| Sub-agents | ✅ |
| Slash commands (/commit, /review, /diff, etc.) | ✅ |
| Memory system | ✅ |
| Images (base64/URL) | ✅ |
| **VS Code integration** | ✅ **NEW** |
| **Codex API (GPT-5.4, GPT-5.3)** | ✅ **NEW** |
| **Reasoning models (qwen3, etc.)** | ✅ **NEW** |

---

## Available Models

| `OPENAI_MODEL` | Model | Auth | Cost |
|---|---|---|---|
| `codexplan` | GPT-5.4 (reasoning: high) | ChatGPT Plus/Pro | $20/mo included |
| `codexspark` | GPT-5.3 Codex Spark | ChatGPT Plus/Pro | $20/mo included |
| `gpt-4o` | GPT-4o | OpenAI API key | Pay-per-token |
| `qwen3:14b` | Qwen3 14B | Ollama | Free |
| `llama3.3:70b` | Llama 3.3 70B | Ollama | Free |
| `deepseek-chat` | DeepSeek V3 | DeepSeek API | Pay-per-token |

---

## Pricing

No separate billing for Codex — it's included in your ChatGPT subscription.

| Plan | Price | Codex Usage |
|---|---|---|
| ChatGPT Plus | $20/mo | 30–150 messages per 5 hours |
| ChatGPT Pro | $200/mo | 300–1,500 messages per 5 hours |
| Ollama | Free | Unlimited (local) |

---

## What Changed from OpenClaude

Only **2 source files** modified + 1 wrapper script added:

| File | Change |
|---|---|
| `src/services/api/openaiShim.ts` | Handle `reasoning` field from thinking models (qwen3, DeepSeek-R1) |
| `src/services/api/codexShim.ts` | `enforceStrictSchema()` — fix tool schemas for Codex strict mode |
| `bin/claudex` | Wrapper script for VS Code integration |

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `CLAUDE_CODE_USE_OPENAI` | Yes | Set to `1` to enable |
| `OPENAI_MODEL` | Yes | `codexplan`, `codexspark`, `gpt-4o`, etc. |
| `OPENAI_API_KEY` | Varies | Not needed for Codex or Ollama |
| `OPENAI_BASE_URL` | No | API endpoint (auto-detected for Codex) |
| `CODEX_API_KEY` | No | Override Codex auth token |

---

## Troubleshooting

| Issue | Solution |
|---|---|
| Codex 400 schema error | Already patched in this fork |
| Empty output (qwen3) | Already patched — reasoning field handled |
| `OPENAI_API_KEY required` | Set `OPENAI_API_KEY=ollama` for remote Ollama |
| Codex auth expired | Run `codex login` again |
| Claude Code extension asks for login | Expected — use VS Code terminal + `claudex` command instead |

---

## Credits

- [OpenClaude](https://github.com/Gitlawb/openclaude) — OpenAI-compatible shim for Claude Code
- [Claude Code](https://claude.ai/code) — Anthropic's agentic coding CLI
- [Codex CLI](https://github.com/openai/codex) — OpenAI's coding agent

## License

Educational and research purposes. Original Claude Code source is property of Anthropic.
OpenAI shim and Codex patches are public domain.
