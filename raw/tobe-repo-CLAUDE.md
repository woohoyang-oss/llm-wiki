# ToBe Networks Skills Project

## 개요
ToBe Networks 클로드 엔터프라이즈 직원용 커스텀 스킬 모음.
클로드 데스크탑 또는 클로드 코드에서 `/스킬명`으로 실행 가능.

## 사용 가능한 스킬

| 슬래시 커맨드 | 대상 | 설명 |
|--------------|------|------|
| `/daily-report` | 경영진, 팀장 | 매출/광고/CS 일일 종합 리포트 |
| `/ad-check` | 마케팅팀 | GFA 실시간 KPI, 예산, 피로도, 낭비 분석 |
| `/inventory-check` | 상품/물류팀 | 재고 현황, 품절 알림, 스마트 발주 제안 |
| `/cs-pending` | CS팀 | 미답변 문의 정리, 우선순위, 답변 초안 |
| `/margin-check` | MD, 상품기획 | 원가/실판가/마진율, 문제 상품 식별 |
| `/review` | 개발팀 | 코드 리뷰 (보안/성능/버그/품질) |
| `/trend` | 마케팅/MD | 상위 키워드, 트렌드, 매출-광고 상관관계 |
| `/weekly-brief` | 전사 | 주간 경영 브리핑 + 노션 저장 |

## MCP 도구 의존성
이 스킬들은 아래 MCP 도구가 연결되어 있어야 정상 동작합니다:
- 매출: `get_sales_overview`, `get_sales_daily`, `get_sales_by_channel`, `get_sales_by_product`, `get_sales_by_model`
- 광고: `get_ad_overview`, `get_gfa_*` 시리즈, `get_sa_*` 시리즈, `get_top_keywords`, `get_waste_keywords`
- CS: `get_cs_overview`, `get_unanswered_inquiries`, `get_qna_unanswered`, `get_inquiry_answer_rate`, `get_claims_*`
- 재고: `get_inventory_current`, `get_reorder_alerts`, `get_smart_reorder`, `get_purchase_orders`
- 원가: `get_margin_analysis`, `get_problem_products`, `get_product_cost`, `get_products_cost_batch`, `get_product_real_price`
- 트렌드: `get_trend_analysis`, `get_revenue_ad_correlation`
- 기타: `get_bizmoney`, Notion MCP (주간 브리핑 노션 저장용)

## 배포
이 저장소를 클론하면 `.claude/skills/` 내 스킬이 자동 인식됩니다.
