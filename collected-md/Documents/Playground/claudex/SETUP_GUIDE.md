# Claudex — Implementation Guide

> This guide is written for both humans and AI agents (Claude, Codex, GPT).
> If you are an AI: follow the steps exactly. If you are a human: copy-paste the commands.

---

## What is Claudex?

**Claudex = OpenClaude + Codex**

- **OpenClaude**: A patched version of Claude Code (Anthropic's agentic coding CLI) that works with any OpenAI-compatible LLM instead of Claude.
- **Codex**: OpenAI's coding backend (GPT-5.4) available through ChatGPT Plus/Pro subscriptions.
- **Claudex**: Combines both — run Claude Code's full tool system (Bash, Read, Write, Edit, Grep, Glob, Agent, MCP, etc.) powered by GPT-5.4 via the Codex API.

### Why?

Claude Code has the best agentic tool UX. Codex has GPT-5.4. Claudex gives you both for $20/month (ChatGPT Plus).

---

## Architecture

```
User Input
    │
    ▼
┌─────────────────────────────────┐
│      Claude Code Runtime        │  ← Ink-based terminal UI
│  (tools, agents, MCP, memory)   │
└───────────────┬─────────────────┘
                │
         ┌──────▼──────┐
         │  openaiShim  │  ← Converts Anthropic SDK format ↔ OpenAI format
         │  codexShim   │  ← Converts OpenAI Chat Completions ↔ Codex Responses API
         └──────┬──────┘
                │
    ┌───────────┼───────────────┐
    ▼           ▼               ▼
Codex API    Ollama        OpenAI API
(GPT-5.4)   (local LLM)   (gpt-4o)
```

### Key Translation Layers

1. **openaiShim.ts** — Translates between Anthropic message format and OpenAI Chat Completions API. Handles: messages, tool_use/tool_result, streaming SSE, system prompts, reasoning fields.

2. **codexShim.ts** — Translates between OpenAI Chat Completions and Codex Responses API. Handles: input format conversion, tool schema strict mode enforcement, SSE event mapping.

---

## Setup Steps

### Prerequisites

| Requirement | How to install |
|---|---|
| macOS or Linux | — |
| Node.js v22+ | `brew install node@22` |
| Bun | `curl -fsSL https://bun.sh/install \| bash` |
| ChatGPT Plus ($20/mo) | [chatgpt.com](https://chatgpt.com) |

### Step 1: Clone and Build

```bash
git clone https://github.com/woohoyang-oss/claudex.git
cd claudex
bun install
bun run build
# Output: dist/cli.mjs
```

### Step 2: Codex Authentication

```bash
npm install -g @openai/codex
codex login
# Browser opens → sign in with ChatGPT account
# Result: ~/.codex/auth.json is created with access_token
```

### Step 3: Run

```bash
CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node dist/cli.mjs
```

### Step 4 (optional): Create Alias

```bash
echo 'alias claudex="CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node ~/claudex/dist/cli.mjs"' >> ~/.zshrc
source ~/.zshrc
claudex
```

---

## Patches Applied (2 files changed)

### Patch 1: `src/services/api/openaiShim.ts`

**Problem**: Models like qwen3 return responses in `delta.reasoning` instead of `delta.content` when tools are present.

**Fix**: Check `delta.content || delta.reasoning || delta.reasoning_content` in both streaming and non-streaming code paths.

```typescript
// Streaming (line ~348)
const _deltaText = delta.content || delta.reasoning || delta.reasoning_content
if (_deltaText) { ... }

// Non-streaming (line ~690)
const _msgText = choice?.message?.content || choice?.message?.reasoning || choice?.message?.reasoning_content
if (_msgText) { ... }
```

### Patch 2: `src/services/api/codexShim.ts`

**Problem**: Codex API strict mode rejects tool schemas that have optional properties not in `required`, use `additionalProperties` as schema objects, or contain keywords like `format`, `pattern`, `propertyNames`.

**Fix**: Added `enforceStrictSchema()` function (~60 lines) that recursively:

1. Removes disallowed JSON Schema keywords (`format`, `pattern`, `propertyNames`, `minLength`, `maxLength`, `$ref`, `$id`, `$schema`, `$comment`, etc.)
2. Adds all property keys to `required` array
3. Wraps previously-optional properties as nullable: `{anyOf: [originalType, {type: "null"}]}`
4. Forces `additionalProperties: false` (replaces schema objects with `false`)
5. Recurses into `items`, `anyOf`, `oneOf`, `allOf`, and nested `properties`

```typescript
// Applied at tool conversion (line ~365)
parameters: enforceStrictSchema(tool.input_schema ?? { type: 'object', properties: {} }),
```

---

## Model Reference

| OPENAI_MODEL value | Actual model | API endpoint | Auth method |
|---|---|---|---|
| `codexplan` | GPT-5.4 (reasoning: high) | chatgpt.com/backend-api/codex | ~/.codex/auth.json |
| `codexspark` | GPT-5.3-codex-spark | chatgpt.com/backend-api/codex | ~/.codex/auth.json |
| `gpt-4o` | GPT-4o | api.openai.com/v1 | OPENAI_API_KEY |
| `qwen3:14b` | Qwen3 14B (Q4_K_M) | localhost:11434/v1 | none (Ollama) |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `CLAUDE_CODE_USE_OPENAI` | — | **Required.** Set to `1` to enable OpenAI provider. |
| `OPENAI_MODEL` | `gpt-4o` | Model name or alias (`codexplan`, `codexspark`, etc.) |
| `OPENAI_API_KEY` | — | API key. Not needed for Codex (uses auth.json) or Ollama. |
| `OPENAI_BASE_URL` | auto | API endpoint. Auto-detected for Codex. Set for Ollama/custom. |
| `CODEX_API_KEY` | — | Override Codex access token (instead of auth.json). |
| `CODEX_AUTH_JSON_PATH` | `~/.codex/auth.json` | Custom path to Codex auth file. |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Codex API error 400: Invalid schema` | Tool schema incompatible with strict mode | Already patched — rebuild with `bun run build` |
| Empty output from qwen3 | Model returns `reasoning` instead of `content` | Already patched |
| `OPENAI_API_KEY is required` | Remote Ollama not recognized as local | Set `OPENAI_API_KEY=ollama` |
| `Codex API error 401` | Auth token expired | Run `codex login` again |
| Slow response with Ollama + tools | qwen3 generates long reasoning with tools | Use `qwen3-nothink:14b` or reduce tool count |

---

## For AI Agents

If you are Claude, Codex, GPT, or another AI reading this to help a user set up Claudex:

1. Check prerequisites: Node.js, Bun, ChatGPT subscription
2. Clone repo, `bun install`, `bun run build`
3. Run `codex login` for authentication
4. Run with `CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node dist/cli.mjs`
5. If schema errors occur, they should already be fixed. If new ones appear, the pattern is: add the offending keyword to `DISALLOWED_KEYS` in `codexShim.ts` and rebuild.
