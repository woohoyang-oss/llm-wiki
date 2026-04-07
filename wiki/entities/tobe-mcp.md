---
title: ToBe Networks MCP 스킬
tags: [MCP, 스킬, ToBe Networks, 자동화, 이커머스, eywa]
created: 2026-04-06
updated: 2026-04-07
sources: [tobe-repo-CLAUDE.md]
---
# ToBe Networks MCP 스킬

## 개요

ToBe Networks 클로드 엔터프라이즈 직원용 커스텀 스킬 프로젝트다. 클로드 데스크탑이나 [[claudex|클로드 코드]]에서 `/스킬명` 슬래시 커맨드로 실행하며, MCP 도구를 통해 매출, 광고, CS, 재고, 원가 등 이커머스 경영 데이터를 실시간으로 분석한다.

## 가이드 (13개)

| 가이드 | 용도 |
|--------|------|
| `/morning-brief` | 아침 운영 브리핑 |
| `/cs-urgency` | CS 긴급 대응 리스트 |
| `/ad-daily` | 광고 일일 리포트 |
| `/reorder-check` | 발주 의사결정 |
| `/product-health` | 상품 종합 진단 |
| `/weekly-report` | 주간 경영 리포트 |
| `/budget-optimize` | 광고 예산 최적화 |
| `/daily-checkin` | 매니저 업무 체크인 |
| `/pi-risk-check` | PI(발주) 리스크 점검 |
| `/inventory-health` | 재고 건강 종합 점검 |
| `/cash-health` | 현금 흐름 건강 점검 |
| `/brand-scorecard` | 브랜드 성적표 |
| `/period-review` | 기간 비교 리뷰 |

## MCP 도구 (74개)

- **매출 (12)**: `get_sales_overview`★ERP전채널, `get_sales_daily`, `get_sales_by_channel`, `get_sales_by_product`, `get_sales_by_model`, `get_sales_by_brand`★ERP전채널, `get_product_real_price`, `get_product_daily_sales`, `get_switch_preference`, `get_period_comparison`★ERP, `get_sales_target_progress`★ERP, `get_channel_sales`
- **재고 (13)**: `get_inventory_current`, `get_inventory_detail`, `get_inventory_capital`, `get_inventory_turnover`, `get_reorder_alerts`, `get_trend_analysis`, `get_smart_reorder`, `get_purchase_orders`, `get_sibling_analysis`, `get_shipment_expedition`, `get_order_recommendation`, `get_stale_inventory`, `get_trend_movers`
- **원가 (5)**: `get_product_cost`, `get_products_cost_batch`, `get_product_cost_by_keyword`★실거래가매칭, `get_margin_analysis`, `get_fx_sensitivity`
- **상품분석 (2)**: `get_product_lifecycle`, `get_cannibalization_check`
- **CS (12)**: `get_cs_overview`, `get_unanswered_inquiries`, `get_claims_summary`, `get_as_status`, `get_as_diagnosis`, `get_as_products`, `get_inquiry_answer_rate`, `get_inquiry_categories`, `get_claims_rate`, `get_problem_products`, `get_qna_unanswered`, `get_cs_workload_forecast`
- **광고 SA (4)**: `get_ad_overview`, `get_sa_performance`, `get_sa_daily`, `get_bizmoney`
- **광고 GFA 기간 (5)**: `get_gfa_performance`, `get_gfa_funnel`, `get_waste_keywords`, `get_top_keywords`, `get_revenue_ad_correlation`
- **GFA 실시간 (4)**: `get_gfa_realtime_kpi`, `get_gfa_realtime_hourly`, `get_gfa_realtime_campaigns`, `get_gfa_realtime_alerts`
- **GFA OPS AI (3)**: `get_gfa_ops_status`, `get_gfa_ops_scores`, `get_gfa_ops_proposals`
- **GFA 대시보드 (3)**: `get_gfa_budget_insight`, `get_gfa_fatigue`, `get_gfa_waste_creatives`
- **발주 리스크 (2)**: `get_po_risk_check`, `get_po_brand_summary`
- **위키 (5)**: `wiki_search`, `wiki_read`, `wiki_list`, `wiki_graph`, `wiki_contribute`
- **가이드 (2)**: `guide_list`, `guide_read`

## 2026-04-07 변경: product_cost_by_keyword 실거래가 매칭

### 문제
`product_cost_by_keyword`의 마진 계산이 `erp_product_info.actual_price`(관리자 설정 정가)를 사용하여 실제 마진과 큰 괴리 발생.
- 예: EX75 actual_price 119,000원 → 마진 52.1% (허수)
- 실제: 스마트스토어 평균 실거래가 71,257원 → 마진 약 20% (실제)

### 수정
1. `sql/cost_queries.py`: 원가 쿼리에서 마진 계산 제거, `REAL_PRICE_BY_GOODS_NAME` 쿼리 추가
2. `services/cost_service.py`: 2단계 처리로 변경
   - 1단계: 원가 데이터 조회 (LATERAL JOIN)
   - 2단계: 같은 키워드로 `naver_smartstore_orders`에서 90일 실거래가 조회
   - 3단계: 가중평균 실거래가로 마진 재계산
3. 응답 필드 추가: `avg_real_price`, `real_order_count`
4. `margin_pct` = (실거래가 - VAT포함원가) / 실거래가 × 100

## 실행 패턴

모든 스킬은 3단계 패턴을 따른다:

1. **데이터 수집** — MCP 도구를 병렬로 호출하여 원시 데이터 확보
2. **분석 및 리포트** — 정해진 템플릿에 따라 구조화된 리포트 작성
3. **액션/심화** — 추가 분석 제안 또는 노션 저장 등 후속 작업

## 배포

- eywa MCP: `http://192.168.1.59:8080/mcp` (Streamable HTTP)
- Claude Code 등록: `claude mcp add --transport http -s user eywa http://192.168.1.59:8080/mcp`
- 코드 수정 시: 로컬 `tobe-mcp-v1/` 수정 → rsync → eywa 재시작

## 관련

- [[tobe-mcp-dev-guide]] — 개발자 가이드
- [[tobe-mcp-improvement]] — 개선 프로젝트
- [[eywa-dev-guide]] — eywa MCP 개발자 가이드
