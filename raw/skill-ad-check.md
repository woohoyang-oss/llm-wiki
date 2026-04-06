---
name: ad-check
description: 광고 성과 분석 및 최적화 제안. GFA 실시간 KPI, 예산 소진율, 피로도, 낭비 키워드/소재 분석. "광고 성과", "광고 분석", "ROAS", "광고비" 등의 요청 시 활성화.
user-invocable: true
allowed-tools: Bash, Read, Write
---

# 광고 성과 분석

## 1단계: 데이터 수집 (병렬 실행)

다음 MCP 도구를 병렬로 호출하세요:

- `get_ad_overview` — 광고 전체 개요
- `get_gfa_realtime_kpi` — GFA 실시간 KPI
- `get_gfa_realtime_campaigns` — 실시간 캠페인 현황
- `get_gfa_realtime_hourly` — 시간대별 성과
- `get_gfa_budget_insight` — 예산 소진 현황
- `get_gfa_fatigue` — 광고 피로도
- `get_gfa_funnel` — 퍼널 분석
- `get_gfa_waste_creatives` — 낭비 소재
- `get_waste_keywords` — 낭비 키워드
- `get_top_keywords` — 상위 키워드

## 2단계: 분석 및 리포트

### 리포트 형식

```
🎯 광고 성과 분석 리포트 (YYYY-MM-DD)

■ 핵심 KPI
- 총 광고비: ₩XXX / 예산 소진율: X%
- ROAS: X.XX (목표 대비 +/-X%)
- 클릭률(CTR): X.X% / 전환율(CVR): X.X%

■ 캠페인별 성과
| 캠페인 | 비용 | ROAS | CTR | 상태 |
|--------|------|------|-----|------|
| ...    | ...  | ...  | ... | ...  |

■ 시간대별 성과
- 최고 효율 시간대: XX시 (ROAS X.XX)
- 최저 효율 시간대: XX시 (ROAS X.XX)

■ 피로도 경고
- [피로도 높은 소재/키워드 목록]

■ 낭비 요소
- 낭비 키워드: [목록 + 소진 금액]
- 낭비 소재: [목록 + 소진 금액]

■ 최적화 제안
1. [즉시 실행 가능한 액션]
2. [단기 개선 액션]
3. [중기 전략 제안]
```

## 3단계: 인사이트

- 전일/전주 대비 변화 트렌드 분석
- 매출-광고 상관관계가 필요하면 `get_revenue_ad_correlation` 추가 호출
- SA(검색광고) 분석이 필요하면 `get_sa_performance`, `get_sa_daily` 추가 호출
