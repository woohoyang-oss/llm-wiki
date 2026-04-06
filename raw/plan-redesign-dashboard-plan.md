# Dashboard Redesign Plan: Keychron Newspaper Style

## CURRENT STATE ANALYSIS

### Architecture Summary
- **Framework**: Astro 5.9 SSR with Node standalone adapter
- **Deployment**: EC2 t3.micro (ap-northeast-2) + nginx + pm2
- **Auth**: Cookie-based middleware protection
- **Data Storage**: Filesystem-based (`data/insights/index.json` + dated directories)

### Current Data Flow
1. **Upload** → `POST /api/insights/upload.ts`
   - Accepts: .html, .xlsx, .csv, .tsv, .md (max 5MB)
   - Saves to: `data/insights/{date}/{slug}.{ext}`
   - Triggers async analysis: `tryAnalyze(entry)` (fire-and-forget)

2. **Analysis** → `src/lib/analyze.ts`
   - Calls Claude CLI (`/usr/bin/claude`) to generate:
     - `{slug}.analysis.md` (detailed markdown report)
     - `{slug}.analysis.html` (styled dashboard HTML)
   - Extracts structured data (summary, findings, tags) from MD
   - Updates `index.json` with `InsightAnalysis` object

3. **Display** → Three layers of rendering
   - Priority 1: Claude-generated HTML (`.analysis.html`) - full styled dashboard
   - Priority 2: MD sections fallback (`.analysis.md`) - parsed & styled with Tailwind
   - Priority 3: JSON analysis (structured data) - basic display
   - Original data shown under "Original Data" section

4. **Chatbot** → `src/pages/api/chat.ts`
   - Reads ALL `.analysis.md` files as knowledge base
   - Up to 10 recent uploaded insights + static pages
   - Only references data in MD, never fabricates

### Current Pages & Routes
- `/` - Dashboard (index.astro) - GitHub insights + "Recent Insights" section
- `/insights` - Upload & listing page (insights/index.astro)
- `/insights/[date]/[slug]` - Full insight viewer (insights/[date]/[slug].astro)
- `/rapid-trigger` - Static market analysis page (rapid-trigger.astro)
- `/api/insights/upload` - Upload endpoint
- `/api/chat` - Chatbot endpoint
- `/api/insights/analysis-md` - MD export endpoint

### Key Data Structures
```typescript
interface InsightEntry {
  id: string;
  date: string;
  slug: string;
  title: string;
  filename: string;
  fileType: string;
  uploadedAt: string;
  fileSize: number;
  analysis?: InsightAnalysis;  // Populated after analysis completes
}

interface InsightAnalysis {
  summary: string;
  keyFindings: string[];
  tags: string[];
  analyzedAt: string;
}
```

### UI/UX Rules (from CLAUDE.md)
- No inner scroll/overflow containers - full-page scroll only
- No dark mode - light warm background only
- Sidebar collapse/expand support
- Design system: rounded-[1.5rem] cards, Tailwind with warm colors

---

## REDESIGN REQUIREMENTS

### Goal: Dashboard as "Keychron Newspaper"
Transform the dashboard (`/`) from generic GitHub intel display to a dedicated insights showcase.

### Key Changes

#### 1. Dashboard Layout Restructure
**Current**: 
- Hero section (GitHub description)
- KPIs (GitHub stats)
- Data blocks recommendations
- Competitor positioning table
- VOC framework
- Actions table
- Roadmap + narrative
- GitHub repos as cards
- Firmware commits log
- Recent Insights small section (5 items only)

**New**:
- Keep: Hero section, GitHub insights sections (move down or secondary)
- **Replace "Recent Insights" section** → Full "Market Insights Newspaper"
  - Insights as **summary cards** (like newspaper preview cards)
  - Display more insights (pagination)
  - Each card shows:
    - Title
    - Summary (first ~150 chars from analysis.summary)
    - Key insight callout (first key finding)
    - Tags
    - "Read more →" link
    - Upload date badge
    - (Optional) Small thumbnail/indicator of analysis status

#### 2. Summary Card Component
**New Component**: `src/components/InsightCard.astro`
- Takes: InsightEntry with populated analysis
- Renders: Newspaper-style preview card
- Design: Follows design system (rounded-[1.5rem], card/95, shadow-soft)
- Content:
  ```
  [Date Badge] [Tags...]
  [Title - bold, truncate 1 line]
  [Summary - truncate 2-3 lines]
  
  [Key Finding Highlight Box]:
    ✦ [First key finding with emoji/icon]
  
  [Status: "Read more →" link] [File type badge]
  ```

#### 3. Pagination
- Replace hardcoded `slice(0, 5)` 
- Add pagination controls at bottom
- Default: 6-12 cards per page
- Sort by: uploadedAt (newest first, already sorted in readIndex)
- Show: page number, total count, prev/next buttons

**Implementation**:
- Add to dashboard index.astro: `?page=1` query param
- Calculate: `total / perPage`, `offset = (page-1) * perPage`
- Fetch: `insights.slice(offset, offset + perPage)`
- Render: Pagination component at bottom

#### 4. New Insight Upload Flow (Chatbot Integration)
When distributor uploads new insight:
1. **Receive** → `/api/insights/upload` saves file
2. **Analyze** → Claude generates .analysis.md + .analysis.html
3. **Index** → Extract summary/findings/tags, update index.json
4. **Chatbot** → Already reads .analysis.md, knowledge auto-updates
5. **Dashboard** → Card appears on next page reload (newest first)

This flow is **already implemented** in analyze.ts - no changes needed.

---

## IMPLEMENTATION TASKS

### Phase 1: New Components & Styling
1. Create `src/components/InsightCard.astro`
   - Props: InsightEntry + fallback styling
   - Show analysis summary, first key finding, tags
   - "Read more" link to `/insights/{date}/{slug}`

2. Create `src/components/PaginationControls.astro`
   - Props: currentPage, totalPages, baseUrl
   - Render: [← Prev] [Page X of Y] [Next →]
   - Disable prev/next when at bounds

### Phase 2: Dashboard Refactor
1. Modify `src/pages/index.astro`
   - Keep existing GitHub sections (move after insights for visual hierarchy)
   - Replace inline "Recent Insights" section with new component
   - Add pagination logic:
     ```typescript
     const page = new URL(Astro.request.url).searchParams.get('page') || '1';
     const currentPage = Math.max(1, parseInt(page));
     const perPage = 12; // or 6/9/12
     const offset = (currentPage - 1) * perPage;
     const totalPages = Math.ceil(insights.length / perPage);
     const paginatedInsights = insights.slice(offset, offset + perPage);
     ```
   - Render insights as grid of InsightCards
   - Render PaginationControls below

### Phase 3: Styling Refinement
1. InsightCard design tweaks
   - Consider: newspaper-style layout (headline, subtitle, meta, CTA)
   - Optional: small icon/badge showing file type (html, xlsx, csv)
   - Optional: visual indicator for analysis status (green checkmark if analysis complete)

2. Typography hierarchy
   - Card title: `text-base font-semibold`
   - Summary text: `text-sm text-foreground/80`
   - Key finding: `rounded-xl border-l-4 border-primary bg-primary/5 p-3` (Tailwind insight box)
   - Tags: `rounded-full bg-primary/10 px-2 py-0.5 text-[10px]`

### Phase 4: Testing & Refinement
1. Check pagination UX
   - Does page param work? Does it reset on upload?
   - Is loading fast enough for 50+ insights?

2. Test with real data
   - Upload 10+ insights, verify they appear as cards
   - Verify analysis.md is parsed and displayed
   - Verify chatbot still works with new dashboard layout

---

## TECHNICAL DETAILS

### Files to Modify
- `src/pages/index.astro` - Add pagination + insights grid layout
- `src/components/InsightCard.astro` - **NEW** - Summary card component
- `src/components/PaginationControls.astro` - **NEW** - Pagination UI

### Files to Keep Unchanged
- `src/lib/insights.ts` - readIndex, saveInsight all work
- `src/lib/analyze.ts` - Analysis pipeline already correct
- `src/pages/insights/[date]/[slug].astro` - Full viewer unchanged
- `src/pages/insights/index.astro` - Upload page unchanged
- `src/pages/api/insights/upload.ts` - Upload handler unchanged
- `src/pages/api/chat.ts` - Chatbot reads MD already
- `src/layouts/InsightsLayout.astro` - Sidebar logic unchanged
- `src/components/InsightsSidebar.astro` - Sidebar unchanged

### Data Structure - No Changes Needed
- `index.json` already tracks: date, slug, title, analysis (summary, keyFindings, tags)
- `.analysis.md` already generated by Claude
- `.analysis.html` already generated by Claude

---

## DESIGN SYSTEM ALIGNMENT

### Hero Section (Keep as-is)
```
rounded-[2rem] border border-border/60 bg-[#f7f6ef] shadow-soft
px-8 py-8 lg:px-10
```

### Insights Grid Section (NEW)
```
<section class="flex flex-col gap-8">
  <div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
    {paginatedInsights.map(entry => <InsightCard entry={entry} />)}
  </div>
  <PaginationControls ... />
</section>
```

### InsightCard Component (NEW)
```astro
<div class="rounded-[1.5rem] border border-border/70 bg-card/95 shadow-soft backdrop-blur p-6 flex flex-col gap-3 hover:shadow-md transition-shadow">
  <!-- Meta row: date + tags -->
  <div class="flex items-center justify-between">
    <span class="text-xs text-muted-foreground">{entry.date}</span>
    <div class="flex gap-1">
      {entry.analysis?.tags.slice(0, 2).map(tag => <Tag>{tag}</Tag>)}
    </div>
  </div>
  
  <!-- Title -->
  <h3 class="font-semibold text-foreground line-clamp-2">{entry.title}</h3>
  
  <!-- Summary -->
  {entry.analysis && (
    <p class="text-sm text-foreground/70 line-clamp-2">{entry.analysis.summary}</p>
  )}
  
  <!-- Key finding callout -->
  {entry.analysis?.keyFindings?.[0] && (
    <div class="rounded-xl border-l-4 border-primary bg-primary/5 p-3 text-sm text-foreground/80">
      <strong>Key:</strong> {entry.analysis.keyFindings[0]}
    </div>
  )}
  
  <!-- CTA row: read more + file type -->
  <div class="flex items-center justify-between pt-2 border-t border-border/50">
    <a href={`/insights/${entry.date}/${entry.slug}`} class="text-xs font-medium text-primary hover:underline">
      Read more →
    </a>
    <span class="text-[10px] rounded bg-muted px-1.5 py-0.5 text-muted-foreground">
      {entry.fileType.toUpperCase()}
    </span>
  </div>
</div>
```

---

## EXPECTED OUTCOME

### Before (Current Dashboard)
- Insights section: 5 items, list view, small "View all" link

### After (Redesigned Dashboard)
- Insights section: 6-12 items per page, grid/card view (newspaper style)
- Pagination controls to browse all uploaded insights
- More visual prominence to uploaded market data
- Matches Keychron brand positioning: "Distributor-friendly intelligence"
- Chatbot knowledge automatically updated on each new upload

---

## SUCCESS METRICS
1. Dashboard loads within 2s (with ~50 insights)
2. Pagination works smoothly (page param persists)
3. New insights appear on dashboard immediately after upload
4. Card preview is compelling enough to drive "Read more" clicks
5. Mobile responsive (cards stack to 1-2 cols on mobile)

---

## OPTIONAL ENHANCEMENTS (Post-MVP)
1. Search/filter by tag or date range
2. Sort dropdown (newest, oldest, most findings)
3. Featured/pinned insights
4. Insight statistics (total uploaded, avg analysis time)
5. Dark mode toggle (against CLAUDE.md rules, skip for now)
6. Export insights as CSV/JSON feed for partners

