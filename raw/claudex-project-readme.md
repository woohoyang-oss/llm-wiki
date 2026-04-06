# Claudex — OpenClaude + Codex

## What is it?
Claudex = OpenClaude + Codex. Run Claude Code's full agentic tool system (Bash, Read, Write, Edit, Grep, Glob, Agent, MCP) powered by GPT-5.4 (Codex) or any LLM.

## GitHub
https://github.com/woohoyang-oss/claudex

## Architecture
```
Claude Code Tools → openaiShim (format translation) → codexShim (strict schema) → Codex API / Ollama / OpenAI
```

## Patches (2 files)
1. openaiShim.ts: Handle reasoning/reasoning_content fields from thinking models (qwen3, DeepSeek-R1)
2. codexShim.ts: enforceStrictSchema() for Codex API strict mode (required fields, disallowed keywords, nullable optionals)

## Quick Start
```bash
git clone https://github.com/woohoyang-oss/claudex.git
cd claudex && bun install && bun run build
npm install -g @openai/codex && codex login
CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node dist/cli.mjs
```

## Models
- codexplan: GPT-5.4 (ChatGPT Plus $20/mo)
- codexspark: GPT-5.3 Codex Spark
- qwen3:14b: Ollama (free)

## Shell Alias
```bash
alias claudex='CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node ~/claudex/dist/cli.mjs'
```