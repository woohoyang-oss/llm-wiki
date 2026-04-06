# Plan: Dashboard Newspaper Redesign (현재 스타일 가이드 유지)

## Context
대시보드를 "Keychron 뉴스페이퍼" 느낌으로 재구성. 인사이트가 추가되면 요약 섹션이 대시보드에 나타나고, 더보기 → 인사이트 페이지로 이동. 글로벌 디스트리뷰터들이 업로드하는 인사이트를 일목요연하게 볼 수 있는 구조.

## Changes

### 1. Dashboard (index.astro) — 전면 재설계
**기존**: 하드코딩된 Hero + KPI + GitHub 섹션 중심
**변경**: 뉴스페이퍼 레이아웃

```
┌─────────────────────────────────────────────────┐
│  Keychron Market Intelligence                   │
│  Latest market data from global distributors    │
├─────────────────────────────────────────────────┤
│ ┌──────────────────┐ ┌──────────────────┐       │
│ │ Featured Insight │ │ Latest Upload    │       │
│ │ (큰 카드, 요약)   │ │ (작은 카드)       │       │
│ └──────────────────┘ └──────────────────┘       │
│ ┌────────┐ ┌────────┐ ┌────────┐               │
│ │ Card 3 │ │ Card 4 │ │ Card 5 │               │
│ └────────┘ └────────┘ └────────┘               │
│                                                  │
│  ← Page 1 of 3 →                                │
├─────────────────────────────────────────────────┤
│  Keychron GitHub Activity (컴팩트)               │
└─────────────────────────────────────────────────┘
```

- 첫 번째 인사이트 = Featured (큰 카드, summary 전문 + key findings)
- 나머지 = 3열 그리드, 제목 + 요약 1줄 + 태그 + 날짜 + "Read more"
- 페이지당 6개 (1 featured + 5 grid)
- `?page=2` 쿼리 파라미터로 페이지네이션
- GitHub Activity 섹션은 하단 컴팩트로 유지

### 2. New Component: InsightCard.astro
```
┌─────────────────────────────────┐
│ 2026-04-05 · HTML · 53KB       │
│ Keychron Korea Pricing Audit   │  ← 제목
│ Pricing audit of 47 matched... │  ← summary 1줄
│ ┌─────┐ ┌──────┐ ┌─────┐      │  ← 태그
│ │korea│ │price │ │audit│      │
│ └─────┘ └──────┘ └─────┘      │
│                    Read more → │
└─────────────────────────────────┘
```

### 3. Upload Pipeline 표준화 (이미 구현됨, 문서화만)
1. Save file → `data/insights/{date}/{slug}.{ext}`
2. Generate `.analysis.md` + `.analysis.html` via Claude CLI (`-p -` stdin)
3. Extract summary/findings/tags → update `index.json`
4. Chatbot auto-reads new MD
5. Dashboard auto-shows new card (SSR, no rebuild needed)

## Files to Change

| Action | File | Description |
|--------|------|-------------|
| Rewrite | `src/pages/index.astro` | Newspaper layout, pagination |
| New | `src/components/InsightCard.astro` | Card component |
| Minor | `src/components/InsightsSidebar.astro` | Ensure "Dashboard" active on / |
| No change | `src/lib/insights.ts` | Data layer unchanged |
| No change | `src/lib/analyze.ts` | Pipeline unchanged |
| No change | `src/pages/api/*` | APIs unchanged |

## Verification
1. `npm run build` 성공
2. https://kychr.space/ → 뉴스페이퍼 스타일 대시보드
3. 인사이트 카드 클릭 → 해당 페이지로 이동
4. ?page=2 → 다음 페이지
5. 챗봇 기능 유지
6. 사이드바 기능 유지
