# shadcn 현재 적용 상태 분석

## 1. 차트/UI 라이브러리
- shadcn ^4.0.4, tailwindcss ^4, recharts ^3.8.0, lucide-react ^0.577.0
- chart.js 미사용 — Recharts만 사용 중

## 2. shadcn UI 컴포넌트 (19개)
badge, button, card, checkbox, collapsible, dialog, input, label, progress, scroll-area, select, separator, sheet, sidebar, skeleton, switch, tabs, textarea, tooltip

## 3. 대시보드 차트 구조
- Recharts: BarChart, AreaChart, PieChart
- 직접 SVG: PerformanceRings, sparkline
- CSS: 수평 진행바
- shadcn Chart 컴포넌트(Recharts 래퍼) 미사용

## 4. shadcn 설정
- style: base-nova (shadcn v4 최신)
- Tailwind v4 CSS-based config
- oklch 색상 시스템

## 5. 개선 포인트
- 즉시: shadcn Chart 추가, alert-dialog, toast/sonner, table, dropdown-menu
- 중기: 차트 색상 CSS 변수 통일, 다크모드, KpiCard 리팩토링
