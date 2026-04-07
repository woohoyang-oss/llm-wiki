# Claudex Agent Guide

This repository is a fork of OpenClaude that keeps the Claude Code style tool/runtime surface while routing model traffic through OpenAI-compatible providers, especially Codex.

Use this file as the default operating guide when working in this repo.

## Primary Goal

Preserve the existing Claude Code style runtime and UX.

When changing behavior, prefer small changes in the provider translation layer over broad edits to the main runtime.

## What Matters Most

- Keep the CLI launch path working: build output must still be `dist/cli.mjs`.
- Treat Codex support as a first-class target, not a side experiment.
- Preserve compatibility with OpenAI-compatible providers such as OpenAI API and Ollama.
- Favor narrow shim fixes over invasive runtime rewrites.
- Rebuild and smoke-test after meaningful changes.

## Default Commands

Run all commands from the repo root: `/Users/yangwooho/Documents/Playground/claudex`.

```bash
bun install
bun run build
node dist/cli.mjs --version
```

Useful commands:

```bash
bun run typecheck
bun run smoke
bun run doctor:runtime
bun run dev:codex
bun run dev:openai
bun run dev:ollama
```

Typical Codex launch:

```bash
CLAUDE_CODE_USE_OPENAI=1 OPENAI_MODEL=codexplan node dist/cli.mjs
```

## Key File Map

- `src/entrypoints/cli.tsx`: CLI entrypoint that gets bundled.
- `src/main.tsx`: main runtime and command wiring. Large file, avoid casual edits.
- `src/services/api/providerConfig.ts`: provider selection, alias mapping, base URL resolution, Codex auth lookup.
- `src/services/api/openaiShim.ts`: Anthropic-style message flow translated to OpenAI-compatible APIs.
- `src/services/api/codexShim.ts`: OpenAI-style requests translated to Codex Responses API.
- `scripts/build.ts`: Bun bundle configuration that emits `dist/cli.mjs`.
- `scripts/provider-launch.ts`: profile-aware local launcher for codex/openai/ollama/gemini.
- `scripts/system-check.ts`: runtime diagnostics and environment validation.
- `README.md`: end-user setup and positioning.
- `SETUP_GUIDE.md`: implementation-oriented explanation of the current fork.

## Preferred Change Strategy

When a bug appears in Codex or another OpenAI-compatible backend:

1. Check `providerConfig.ts` first for wrong transport, model, auth, or base URL selection.
2. Check `openaiShim.ts` if the issue is message conversion, tool call mapping, or streaming shape differences.
3. Check `codexShim.ts` if the issue is specific to Codex Responses API, strict schemas, SSE mapping, or response aggregation.
4. Only edit `main.tsx` when the bug truly belongs to shared runtime behavior rather than provider translation.

## Repo-Specific Expectations

- `codexplan` should resolve to GPT-5.4 with high reasoning effort.
- `codexspark` should resolve to the faster Codex model path.
- `CLAUDE_CODE_USE_OPENAI=1` is the main switch for OpenAI-compatible provider mode.
- Codex auth usually comes from `~/.codex/auth.json` unless overridden by env vars.
- This fork exists mainly to adapt provider behavior, not to redesign the whole app.

## Validation Rules

For docs-only changes, basic review is enough.

For runtime or provider changes, run:

```bash
bun run build
node dist/cli.mjs --version
```

When touching provider selection or auth resolution, also run:

```bash
bun run doctor:runtime
```

When touching TypeScript-heavy logic, prefer also running:

```bash
bun run typecheck
```

## Editing Heuristics

- Prefer minimal patches in shim files.
- Preserve existing naming and flow unless there is a clear reason to refactor.
- Add comments only when the translation logic is genuinely non-obvious.
- Do not silently change model aliases or defaults without checking repo docs too.
- Keep README and setup docs aligned with real runtime behavior when behavior changes.

## Common Pitfalls

- Running `node dist/cli.mjs` outside the repo root.
- Forgetting to rebuild after changing source files.
- Fixing Codex issues in the wrong layer when the real problem is schema normalization in `codexShim.ts`.
- Adding provider-specific logic to wide shared runtime paths when a local shim change would be safer.

## Working Style For Future Agents

- Start by reading `README.md`, this file, and the exact files involved in the bug or feature.
- Summarize the likely execution path before large edits.
- Keep changes easy to diff and easy to revert.
- Prefer concrete verification over assumption.
