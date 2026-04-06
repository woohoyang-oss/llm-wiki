# auto-email 대시보드/통계 UX 디자인 요구사항

## 참고 디자인
- shadcn/ui 공식 차트: https://ui.shadcn.com/charts/area, /charts/pie
- shadcn/ui 블록: https://ui.shadcn.com/blocks
- 이미 shadcn 컴포넌트는 거의 가져옴. 디자인 퀄리티만 올리면 됨

## 현재 문제점 (양대표님 피드백)
1. 차트가 회색/연회색 — 데이터 있는데도 비어보임
2. 상태별 분포 도넛 — 시각적 임팩트 제로
3. 처리 성과 0% — 안내 메시지 필요
4. 우선순위 도넛 — 하나만 있으니 의미 없음
5. 브랜드별 분포 — 색상 구분 없음
6. 일별 볼륨 차트 — 회색 gradient라 안 보임
7. 전체적으로 "개선이 하나도 안 된" 느낌

## 원하는 방향
- shadcn/ui 공식 예제 수준의 깔끔한 디자인
- KPI 카드: 트렌드 표시, sparkline
- Area Chart: 파란색 gradient fill, interactive tooltip
- Bar Chart: 파란색 계열, 멀티 데이터 구분
- 도넛/파이: 의미 있는 데이터만
- 빈 데이터: 행동 유도 메시지
- hover 효과, transition 자연스럽게

## 기술 제약
- Recharts 사용 중
- shadcn ChartContainer + ChartTooltip 사용
- CSS 변수: --chart-1 ~ --chart-5
- KpiCard 컴포넌트 이미 있음

## 작업 방식
- bridge 팀원은 UX 수정 불가 (화면 못 봄)
- Claude Desktop에서 직접 보면서 수정하는 게 효과적
