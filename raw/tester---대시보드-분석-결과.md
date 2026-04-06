# 대시보드 & 통계 페이지 코드 분석

## 1. Dashboard (dashboard/page.tsx, 689줄)
- 위젯 9개: KpiCard x4, StatusDistributionBar, PerformanceRings, AgentWorkload, VolumeChart, RecentActivityFeed, MyPerformanceCard, MyResolvedChart, MyRecentHandled
- API: useDashboard() 30초 자동갱신, useDashboardVolume(14) 60초 자동갱신
- 반응형 그리드, RoleGuard 기반 역할별 섹션

## 2. Statistics (statistics/page.tsx, 743줄)
- 위젯 7개: KpiCard x4, DonutChart x2, HorizontalBar x3, AreaChart, BarChart, Sparkline, CSV 내보내기
- 필터: 기간 프리셋, 커스텀 일수, 브랜드, 상담원

## 3. 개선 포인트
- 높음: 로딩 스켈레톤 없음, 필터 URL 동기화 없음, 접근성 부족, 에러 바운더리 없음
- 중간: SVG/Recharts 혼용, 반복 코드, CSV 이중 로직, 하드코딩 색상
- 낮음: 다크모드 미지원, i18n 미적용, 모바일 최적화 부족
