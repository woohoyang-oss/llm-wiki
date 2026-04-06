# Eywa — Claude Team Orchestration Platform

## Overview
Multi-machine distributed Claude CLI team orchestration platform (MIT, open-source).
GitHub: https://github.com/woohoyang-oss/eywa

## Architecture
- **Frontend**: Next.js 16.2.1 + shadcn/ui v4 + Tailwind CSS (dark theme)
- **Backend**: Express + WebSocket + SQLite (server/, port 3001)
- **Bridge**: bridge.py (Python) wraps Claude CLI as subprocess, communicates via Slack Socket Mode
- **CLI Setup**: setup.sh + Claude CLI interactive prompts for server/client node configuration

## Key Features Implemented
1. **Setup Wizard** (6 steps): Team → Members → Roles/Claude Settings → Integrations → Deploy Envs → Summary/Deploy
2. **Dashboard**: Team overview with CPU/MEM/Session gauges, status indicators, action buttons
3. **Add Node**: Generate join tokens from dashboard, display curl install command for new nodes
4. **Direct Chat**: Talk to any node's Claude CLI from the dashboard UI
5. **Log Viewer**: Real-time log streaming with level/keyword filtering
6. **Member Detail**: Resource charts, editable settings, throttle config
7. **Settings Page**: Integrations (Slack tokens, memory provider), Throttle defaults
8. **Apply Recommendation**: Bulk model optimization based on resource usage
9. **CLI-First Setup**: `curl ... | bash` → Claude CLI guides interactive setup on both server and client nodes
10. **Join Token System**: Server generates tokens, clients self-register and auto-configure SSH keys

## Tech Stack Details
- shadcn/ui v4: Select onValueChange passes `string | null`, no `asChild` on Button
- SSH: Auto-detect key types (ed25519 → rsa → ecdsa), tilde expansion
- Local members: `is_local` flag bypasses SSH, uses execSync for Claude validation
- All UI text in English

## File Structure
- `/app/` — Next.js pages (dashboard, wizard, settings, member detail, chat, logs)
- `/components/` — UI components (wizard steps, dashboard cards, log viewer)
- `/lib/` — API client, WebSocket client, types
- `/server/src/` — Express backend (routes, db, ssh, monitor)
- `/templates/bridge.py` — Slack↔Claude CLI bridge
- `/setup.sh` — CLI entry point (server init / client join)
- `/cli/` — Claude CLI system prompts for setup wizards
- `/docs/` — Architecture and setup guide documentation

## Recent Changes (2026-03-25)
- Added CLI-first setup system with join tokens
- Added "Add Node" button on dashboard with curl command display
- Fixed local Claude validation, SSH tilde expansion, auto-detect key types
- Translated all Korean UI to English
- Implemented Direct Chat, Settings button, Apply Recommendation
