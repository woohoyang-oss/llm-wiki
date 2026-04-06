---
title: ToBe Networks MCP 스킬
tags: [MCP, 스킬, ToBe Networks, 자동화, 이커머스]
created: 2026-04-06
updated: 2026-04-06
sources: [tobe-repo-CLAUDE.md, skill-ad-check.md, skill-cs-pending.md, skill-daily-report.md, skill-inventory-check.md, skill-margin-check.md, skill-review.md, skill-trend.md, skill-weekly-brief.md]
---
# ToBe Networks MCP 스킬

## 개요

ToBe Networks 클로드 엔터프라이즈 직원용 커스텀 스킬 프로젝트다. 클로드 데스크탑이나 [[claudex|클로드 코드]]에서 `/스킬명` 슬래시 커맨드로 실행하며, MCP 도구를 통해 매출, 광고, CS, 재고, 원가 등 이커머스 경영 데이터를 실시간으로 분석한다.

## 8개 스킬 목록

| 슬래시 커맨드 | 대상 | 용도 |
|--------------|------|------|
| `/daily-report` | 경영진, 팀장 | 매출/광고/CS 일일 종합 리포트 생성 |
| `/ad-check` | 마케팅팀 | GFA 실시간 KPI, 예산 소진율, 피로도, 낭비 키워드/소재 분석 |
| `/inventory-check` | 상품/물류팀 | 현재 재고, 품절 임박 알림, 스마트 발주 제안 |
| `/cs-pending` | CS팀 | 미답변 문의 우선순위 정리, 답변 초안 자동 생성 |
| `/margin-check` | MD, 상품기획 | 원가/실판가/마진율 분석, 적자 상품 식별 |
| `/review` | 개발팀 | 코드 리뷰 — 보안, 성능, 버그, 코드 품질 4관점 분석 |
| `/trend` | 마케팅/MD | 상위 키워드, 트렌드 변화, 매출-광고 상관관계 분석 |
| `/weekly-brief` | 전사 | 주간 경영 브리핑 작성 + 노션 페이지 자동 생성 |

## MCP 도구 의존성

스킬들은 다음 MCP 도구 그룹에 의존한다:

- **매출**: `get_sales_overview`, `get_sales_daily`, `get_sales_by_channel`, `get_sales_by_product`, `get_sales_by_model`
- **광고**: `get_ad_overview`, `get_gfa_*` 시리즈, `get_sa_*` 시리즈, `get_top_keywords`, `get_waste_keywords`
- **CS**: `get_cs_overview`, `get_unanswered_inquiries`, `get_qna_unanswered`, `get_claims_*`
- **재고**: `get_inventory_current`, `get_reorder_alerts`, `get_smart_reorder`, `get_purchase_orders`
- **원가**: `get_margin_analysis`, `get_problem_products`, `get_product_cost`, `get_product_real_price`
- **트렌드**: `get_trend_analysis`, `get_revenue_ad_correlation`
- **기타**: `get_bizmoney`, [[Notion]] MCP(주간 브리핑 저장용)

## 실행 패턴

모든 스킬은 3단계 패턴을 따른다:

1. **데이터 수집** — MCP 도구를 병렬로 호출하여 원시 데이터 확보
2. **분석 및 리포트** — 정해진 템플릿에 따라 구조화된 리포트 작성
3. **액션/심화** — 추가 분석 제안 또는 노션 저장 등 후속 작업

## 배포

저장소를 클론하면 `.claude/skills/` 디렉토리 내 스킬 파일이 자동 인식된다.

## 관련 소스

- `tobe-repo-CLAUDE.md` — 프로젝트 개요 및 스킬 목록
- `skill-ad-check.md` — 광고 성과 분석 스킬 상세
- `skill-cs-pending.md` — CS 미답변 관리 스킬 상세
- `skill-daily-report.md` — 일일 리포트 스킬 상세
- `skill-inventory-check.md` — 재고/발주 관리 스킬 상세
- `skill-margin-check.md` — 마진 분석 스킬 상세
- `skill-review.md` — 코드 리뷰 스킬 상세
- `skill-trend.md` — 트렌드 분석 스킬 상세
- `skill-weekly-brief.md` — 주간 브리핑 스킬 상세
