# kychr.space — Keychron Market Intelligence Dashboard

## Repo
- GitHub: https://github.com/woohoyang-oss/kychr.space (private)
- Local: /Users/wooho/kychr.space

## Stack
- Astro 5.9 SSR + React + Tailwind CSS
- EC2 t3.micro (ap-northeast-2, IP: 13.124.196.73) + nginx + pm2
- Deploy: GitHub Actions (main push → build → rsync dist → pm2 restart)

## Auth
- Browser: cookie-based (kychr-auth), user=keychron, pass=8989
- API: Bearer token (kychr_...), keys in data/api-keys.json
- Middleware: src/middleware.ts (cookie first, then Bearer)
- Key generator: npx tsx scripts/gen-api-key.ts <account> <label>

## SSH
- Key: ~/.ssh/kychr-key.pem → vault "kychr-key.pem"
- User: ec2-user
- Host: 13.124.196.73
- Cmd: ssh -i ~/.ssh/kychr-key.pem ec2-user@13.124.196.73

## Key Files
- src/middleware.ts — dual auth (cookie + Bearer)
- src/lib/api-keys.ts — API key management
- src/lib/analyze.ts — Claude CLI analysis pipeline
- src/lib/insights.ts — insight CRUD + index.json
- src/pages/insights/[date]/[slug].astro — insight viewer (iframe srcdoc)
- data/insights/ — uploaded files + analysis artifacts
- data/api-keys.json — API keys (not in git)
- CLAUDE.md — project instructions

## Analysis Pipeline
1. Upload → data/insights/{date}/{slug}.{ext}
2. MD analysis → {slug}.analysis.md (English, via Claude CLI)
3. Translation → {slug}.translated.html (if Korean detected)
4. index.json → English title, summary, findings, tags
5. Viewer: English tab (default) + Original tab

## API Endpoints
- POST /api/auth/login — browser login
- POST /api/insights/upload — file upload (FormData: file, title?, date?)
- POST /api/insights/reanalyze — trigger analysis (JSON: slug? or {} for all)
- GET /api/insights/analysis-md?date=...&slug=... — get MD
- POST /api/chat — chatbot

## Current State (2026-04-06)
- 6 insights, all analyzed with English titles
- API key auth working
- iframe srcdoc for original HTML preservation
- Korean→English auto-translation pipeline
- Bulk reanalyze endpoint