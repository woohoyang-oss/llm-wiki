# API Key Authentication + Regional Manager API Guide

## Context
Regional managers (Korea, Japan, etc.) need to upload insights via API without browser login. Currently only cookie-based auth exists. Need: per-account API keys, Bearer token auth alongside cookies, API documentation.

## Implementation

### 1. `src/lib/api-keys.ts` (new)
- `loadKeys()` — reads `data/api-keys.json`
- `validateKey(bearer)` — returns `ApiKey | null`
- `addKey(account, label)` — generates `kychr_` + UUID key, appends to file
- Key shape: `{ key, account, label, createdAt }`

### 2. `data/api-keys.json` (new, gitignored)
- Seed with one key for `keychron` account

### 3. `src/env.d.ts` (new)
- Declare `App.Locals` with `account: string` and `apiKeyLabel?: string`

### 4. `src/middleware.ts` (modify)
- Check cookie first (cheap) → `locals.account = 'web-session'`
- Then check `Authorization: Bearer <token>` → validate → `locals.account = key.account`
- API routes: return 401 JSON if no auth (not redirect)
- Page routes: keep existing redirect to /login

### 5. `src/pages/api/insights/upload.ts` (modify)
- Pull `locals` from context, include `account` in response

### 6. `src/lib/insights.ts` (modify)
- Add `uploadedBy?: string` to `InsightEntry`
- `saveInsight()` accepts optional `uploadedBy` parameter

### 7. `scripts/gen-api-key.ts` (new)
- CLI: `npx tsx scripts/gen-api-key.ts keychron "Korea HQ"`
- Prints generated key to stdout

### 8. `src/pages/guide.astro` (modify)
- Add "API Access" section with curl examples:
  - Upload: `POST /api/insights/upload` with Bearer + FormData
  - Reanalyze: `POST /api/insights/reanalyze`
  - Get analysis: `GET /api/insights/analysis-md`
- Environment variable pattern: `export KYCHR_API_KEY="kychr_..."`

### 9. `CLAUDE.md` (modify)
- Document API key auth pattern and file locations

## Verification
1. Browser login still works (cookie auth unchanged)
2. `curl -H "Authorization: Bearer kychr_<key>" -F file=@test.html .../api/insights/upload` → 200
3. Bad/missing key → 401 JSON (not redirect)
4. Page routes without cookie → 302 redirect (unchanged)
5. `npx tsx scripts/gen-api-key.ts` generates key in `data/api-keys.json`
