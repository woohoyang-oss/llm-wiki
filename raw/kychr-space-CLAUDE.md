# kychr.space

Keychron Market Intelligence Dashboard (Astro SSR + React + Tailwind)

## Architecture
- Astro 5.9 SSR with Node standalone adapter
- EC2 t3.micro (ap-northeast-2) + nginx reverse proxy + pm2
- Deploy: GitHub Actions (main push → build → rsync dist → pm2 restart)
- Auth: dual — cookie-based (`kychr-auth`) for browser + API key (`Bearer kychr_...`) for scripts
- API keys: `data/api-keys.json` (per-account, generated via `npx tsx scripts/gen-api-key.ts`)
- Middleware: `src/middleware.ts` checks cookie first, then Bearer token. API routes return 401 JSON, pages redirect to /login
- Data: filesystem (`data/insights/`) + `index.json` metadata

## UI/UX Rules — MANDATORY for all pages
- **No inner scroll / overflow containers.** Full-page scroll only. Never use `overflow-auto`, `overflow-hidden`, `h-screen` with flex-col, or iframe. Sidebar is the only exception.
- **No dark mode.** Light warm background only.

## Design System (reference: /rapid-trigger page)
All insight pages and generated reports MUST follow this design system:

### Layout
- `max-w-7xl mx-auto px-6 py-8 lg:px-10` — main content wrapper
- `flex flex-col gap-8` — section spacing
- Full-page scroll, no fixed headers

### Cards
- `rounded-[1.5rem] border border-border/70 bg-card/95 shadow-soft backdrop-blur` — standard card
- `p-6` padding inside cards
- Section title inside card: `<h2 class="text-xl font-semibold tracking-tight">`

### KPI Cards (top metrics)
- Grid: `grid gap-4 md:grid-cols-2 xl:grid-cols-4`
- Label: `text-xs uppercase tracking-wider text-muted-foreground`
- Value: `text-3xl font-bold text-foreground` (or `text-primary` / `text-emerald-600` for highlights)
- Subtitle: `text-sm text-muted-foreground`

### Hero/Header Section (STANDARD — all pages must match)
- Container: `rounded-[2rem] border border-border/60 bg-[#f7f6ef] shadow-soft`
- Inner padding: `px-8 py-8 lg:px-10` (NOT py-10)
- First row: date badge + metadata badges in `flex items-center gap-2 mb-4`
- Date badge: `rounded-full border border-border bg-white/80 px-2.5 py-1 text-xs font-medium`
- Title: `text-3xl font-semibold tracking-tight lg:text-4xl`
- Subtitle: `mt-3 text-base leading-7 text-muted-foreground`
- Tags (if any): `mt-4 flex flex-wrap gap-2`
- **NO back links** (Dashboard/All Insights) — sidebar handles navigation
- **NO extra top margin** before header section

### Tables
- Container: `overflow-x-auto px-6 pb-6` inside card
- Table: `w-full text-left text-sm`
- Header: `bg-muted/50 text-muted-foreground` with `px-4 py-3 font-medium`
- Rows: `border-t border-border/70`
- Highlighted row (Keychron): `bg-primary/5 text-primary font-bold`

### Badges/Tags
- Priority high: `rounded-full bg-emerald-100 px-2 py-0.5 text-[11px] font-semibold text-emerald-700`
- Warning: `bg-red-100 text-red-700`
- Stable: `bg-yellow-100 text-yellow-700`
- Info: `bg-violet-100 text-violet-700`
- Tag: `rounded-full bg-primary/10 px-2 py-0.5 text-[10px] font-medium text-primary`

### Bar charts (inline progress bars)
- `h-2 w-36 rounded-full bg-border` container
- `h-2 rounded-full bg-{color}` fill with percentage width

### Insight boxes (callouts)
- `rounded-xl border-l-4 border-primary bg-primary/5 p-4`
- Bold key phrase + supporting text in `text-sm`

### Strengths/Threats cards
- Strengths: `text-emerald-700` header, `✓` prefix in `text-emerald-500`
- Threats: `text-red-600` header, `⚠` prefix in `text-red-500`

### Charts (Chart.js)
- Light theme: grid `rgba(200,200,210,0.2)`, ticks `#64748b`
- Brand colors: Keychron `#6366f1`, Preflow `#ec4899`, Hansung `#eab308`, Dareu `#f97316`, Corsair `#22c55e`, SteelSeries `#06b6d4`

### Colors (HSL CSS variables)
- background: warm `#fcfcf9`
- foreground: `#1e293b`
- primary: `#2563eb`
- muted-foreground: `#4b5563`
- border: `#e2e8f0`
- card: white

## Insight Upload & Analysis — STANDARD PROCESS
Both upload paths (insights page + chatbot) MUST follow the same pipeline:

1. **Save file** → `data/insights/{date}/{slug}.{ext}`
2. **Generate MD** → `{slug}.analysis.md` via Claude CLI (detailed markdown report)
3. **Update index.json** → extract summary/findings/tags from MD
4. **Chatbot context** → always reads `.analysis.md` files first, NOT raw HTML
5. **Chatbot answers** → ONLY based on MD data, never fabricate

### Original data is king — viewer always shows uploaded content
- **Original uploaded file is ALWAYS displayed as the main content** on insight pages
- AI analysis (`.analysis.md`) is shown as a **collapsible "AI Brief"** above the original data
- AI never replaces, regenerates, or transforms the original uploaded content
- No `.analysis.html` generation — MD only
- Supported upload formats: .html, .xlsx, .csv, .tsv, .md

### MD for chatbot knowledge base
- Every page (insights, rapid-trigger, etc.) MUST have a corresponding `.analysis.md`
- Static pages (rapid-trigger) also generate MD for chatbot knowledge base
- ALL analysis MDs live in `data/insights/{date}/`
- Chatbot reads ALL `.analysis.md` files as knowledge base

### Language rules for insight creation
- **ALL generated content** (title, MD analysis, summary, key findings, tags): **MUST be in English**
- Analysis pipeline generates English title via `# Title` section in MD, updates index.json automatically
- If the uploaded data is in Korean/Japanese/Chinese, **translate analysis content to English**
- Product names, brand names keep original form (e.g., "키크론" → "Keychron", "다얼유" → "Dareu")
- Currency values: include both original and USD equivalent where applicable
- The chatbot responds in the user's language, but stored analysis data is always English

### Uploaded file handling
- **Original files are preserved as-is** on disk — never modify the uploaded file
- HTML is rendered in an **iframe (srcdoc)** to isolate CSS — no style stripping needed
- Only `<script>` tags are stripped from iframe content for security
- The viewer page structure: Header → AI Brief (collapsible) → Report Data (with language tabs if translated)

### Analysis pipeline (per upload)
The pipeline generates these artifacts automatically:
1. **`{slug}.analysis.md`** — AI analysis report (English). Used for chatbot knowledge + dashboard cards
2. **index.json update** — English title, summary, findings, tags extracted from MD
- Pipeline code is in `src/lib/analyze.ts` — `runAnalysis()` function
- All artifacts go to `data/insights/{date}/`
- No HTML translation — upload English content directly if needed

## Commands
- `npm run dev` — local dev server
- `npm run build` — production build
- `npm start` — run built server
