---
title: ToBe MCP 스킬 카탈로그
tags: [ToBe Networks, 스킬, MCP, Claude, 자동화]
created: 2026-04-06
updated: 2026-04-07
sources: [tobe-repo-CLAUDE.md, skill-ad-check.md, skill-cs-pending.md, skill-daily-report.md, skill-inventory-check.md, skill-margin-check.md, skill-review.md, skill-trend.md, skill-weekly-brief.md]
---
# ToBe MCP 스킬 카탈로그

[[ToBe Networks]] 직원용 Claude 커스텀 스킬 8종의 카탈로그. `/스킬명`으로 실행하며, 각 스킬은 MCP 도구를 병렬 호출하여 데이터를 수집하고 정형화된 리포트를 생성한다.

---

## 스킬 목록

| 스킬 | 커맨드 | 대상 부서 | 설명 | 주요 MCP 도구 |
|------|--------|----------|------|--------------|
| 일일 리포트 | `/daily-report` | 경영진, 팀장 | 매출/광고/CS 종합 일일 리포트 | get_sales_overview, get_ad_overview, get_cs_overview |
| 광고 분석 | `/ad-check` | 마케팅팀 | GFA 실시간 KPI, 예산, 피로도, 낭비 분석 | get_gfa_realtime_kpi, get_gfa_fatigue, get_waste_keywords |
| 재고 관리 | `/inventory-check` | 상품/물류팀 | 재고 현황, 품절 알림, 스마트 발주 제안 | get_inventory_current, get_smart_reorder, get_purchase_orders |
| CS 미답변 | `/cs-pending` | CS팀 | 미답변 문의 정리, 우선순위 분류, 답변 초안 | get_unanswered_inquiries, get_qna_unanswered, get_claims_summary |
| 마진 분석 | `/margin-check` | MD, 상품기획 | 원가/실판가/마진율, 문제 상품 식별 | get_margin_analysis, get_problem_products, get_product_cost |
| 코드 리뷰 | `/review` | 개발팀 | 보안/성능/버그/코드 품질 분석 | Bash, Read, Grep, Glob |
| 트렌드 | `/trend` | 마케팅/MD | 상위 키워드, 트렌드, 매출-광고 상관관계 | get_top_keywords, get_trend_analysis, get_revenue_ad_correlation |
| 주간 브리핑 | `/weekly-brief` | 전사 | 주간 경영 브리핑 + 노션 저장 | 12종 MCP 도구 전체 + Notion MCP |

---

## 스킬 공통 구조

모든 비즈니스 스킬은 동일한 3단계 패턴을 따른다:

1. **데이터 수집:** MCP 도구 병렬 호출로 원천 데이터 확보
2. **분석 및 리포트:** 정형화된 리포트 포맷으로 데이터 정리
3. **심화/전달:** 추가 분석 또는 노션 저장 등 후속 액션

---

## 스킬별 트리거

| 스킬 | 트리거 키워드 |
|------|-------------|
| `/daily-report` | "오늘 현황", "일일 리포트", "매출 요약" |
| `/ad-check` | "광고 성과", "ROAS", "광고비" |
| `/inventory-check` | "재고", "발주", "품절", "입고" |
| `/cs-pending` | "미답변", "CS", "문의", "클레임" |
| `/margin-check` | "마진", "원가", "수익성", "단가" |
| `/review` | "코드 리뷰", "리뷰해줘", "PR 리뷰" |
| `/trend` | "트렌드", "키워드", "검색어" |
| `/weekly-brief` | "주간 보고", "위클리", "주간 브리핑" |

---

## MCP 도구 의존성

스킬이 정상 동작하려면 다음 MCP 도구 그룹이 연결되어 있어야 한다:

- **매출 (11):** get_sales_overview, get_sales_daily, get_sales_by_channel, get_sales_by_product, get_sales_by_model, get_sales_by_brand, get_period_comparison, get_sales_target_progress, get_switch_preference, get_product_real_price, get_product_daily_sales
- **광고:** get_ad_overview, get_gfa_* 시리즈, get_sa_* 시리즈
- **CS (12):** get_cs_overview, get_unanswered_inquiries, get_claims_*, get_cs_workload_forecast
- **재고 (18):** get_inventory_current, get_smart_reorder, get_purchase_orders, get_trend_movers, get_stale_inventory, get_inventory_capital, get_inventory_turnover, get_product_lifecycle, get_cannibalization_check 등
- **원가 (5):** get_margin_analysis, get_product_cost, get_products_cost_batch, get_fx_sensitivity, get_product_cost_by_keyword
- **기타:** get_bizmoney, [[Notion]] MCP (주간 브리핑 저장용)

코드 리뷰 스킬(`/review`)만 MCP 도구 대신 Bash/Read/Grep/Glob을 사용한다.

---

## eywa MCP 가이드 (13종)

eywa MCP에서 제공하는 복합 워크플로우 가이드. `guide_list` → `guide_read(skill)` 순으로 조회.

| 가이드 | 용도 | 주요 도구 |
|--------|------|----------|
| `/morning-brief` | 아침 브리핑 | 매출+광고+CS+재고 종합 |
| `/cs-urgency` | CS 긴급 건 점검 | CS 대시보드+미답변 |
| `/ad-daily` | 광고 일일 성과 | GFA 실시간+예산+낭비 |
| `/reorder-check` | 발주 점검 | smart_reorder+trend_analysis |
| `/product-health` | 상품 건강 점검 | 재고+마진+판매 추세 |
| `/weekly-report` | 주간 보고서 | 전체 도구 종합 |
| `/budget-optimize` | 예산 최적화 | 광고 예산+ROAS |
| `/daily-checkin` | 일일 체크인 | 주요 KPI 요약 |
| `/pi-risk-check` | PI 리스크 점검 | po_risk_check+po_brand_summary |
| `/inventory-health` | 재고 건강 종합 | trend_movers+stale_inventory+smart_reorder+po_risk_check |
| `/cash-health` | 현금 흐름 점검 | inventory_capital+fx_sensitivity+margin_analysis |
| `/brand-scorecard` | 브랜드 성적표 | sales_by_brand+inventory_capital+inventory_turnover |
| `/period-review` | 기간 비교 리뷰 | period_comparison+cs_workload_forecast+gfa_realtime_kpi |

### 신규/개선 도구 (2026-04-07)

| 도구 | 설명 |
|------|------|
| `get_sales_by_brand` | ★ ERP 전 채널 브랜드별 매출 (naver → erp_customer_order_list) |
| `get_sales_overview` | ★ ERP 전 채널 매출 요약 (naver → erp_customer_order_list) |
| `get_period_comparison` | ★ ERP 전 채널 기간 비교 |
| `get_sales_target_progress` | 매출 목표 달성률 (erp_customer_month_targert_sales) |
| `get_product_cost_by_keyword` | 상품별 원가 검색 (키워드 → 매입가/USD/환율/마진율) |
| `get_trend_movers` | 카탈로그 전체 판매 추세 급변 (SURGE/DECLINE/INFLECTION) |
| `get_stale_inventory` | 장기 체류 재고 (DEAD/DYING/SLOW + 묶인 자본) |
| `get_inventory_capital` | 전체 재고 자본 (브랜드별 재고금액) |
| `get_inventory_turnover` | 재고 회전율 (EXCELLENT~POOR) |
| `get_product_lifecycle` | 상품 수명주기 PLC (LAUNCH/GROWTH/MATURE/DECLINE) |
| `get_cannibalization_check` | 카니발라이제이션 감지 |
| `get_fx_sensitivity` | 환율 민감도 시뮬레이션 |
| `get_cs_workload_forecast` | CS 부하 예측 (12개월 추이) |
| `get_po_brand_summary` | 브랜드별 발주 요약 + 리스크 플래그 |
| `get_po_risk_check` | 발주 리스크 자동 검출 (4유형) |
