# Claudex Session Notes

This repo should be treated as a provider-adaptation fork, not a general rewrite target.

Read [AGENTS.md](./AGENTS.md) first before making meaningful changes.

## Short Version

- Keep `dist/cli.mjs` as the working build output.
- Prefer fixes in `src/services/api/providerConfig.ts`, `src/services/api/openaiShim.ts`, and `src/services/api/codexShim.ts`.
- Avoid broad edits in `src/main.tsx` unless the issue is clearly outside provider translation.
- After runtime changes, run:

```bash
bun run build
node dist/cli.mjs --version
```

For full repo guidance, use `AGENTS.md` as the source of truth.
