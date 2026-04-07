---
title: Wiki Log
---

# Log

## [2026-04-07] fix | eywa MCP 74개 도구 파이프라인 검증 완료

### 전체 검증 결과
- 핵심 13개: 전부 정상
- 나머지 61개: 48 정상, 7 SQL버그 수정, 6 EC2전용(정상실패)

### SQL 포맷팅 버그 수정 (7개)
- `get_unanswered_inquiries` — LIMIT %s 누락
- `get_as_diagnosis` — LIMIT %s 누락
- `get_as_products` — LIMIT %s 누락
- `get_bizmoney` — WHERE account=%s 누락 + DAILY_AVG_SPEND account 필터 추가
- `get_margin_analysis` — LIMIT %s 누락
- `get_problem_products` — LIMIT %s 누락
- `get_top_keywords` — LIMIT %s 누락

### product_cost_by_keyword 실거래가 매칭
- 문제: actual_price(ERP 정가)로 마진 계산 → 허수 (EX75: 52.1%)
- 수정: naver_smartstore_orders 90일 실거래가 매칭 → 실제 마진 (EX75: 19.9%)
- 새 필드: avg_real_price, real_order_count
- sql/cost_queries.py + services/cost_service.py 수정

---

## [2026-04-07] feat | ERP 전 채널 매출 전환 + 신규 도구 2개

### ERP 전환 (3개 도구)
- `get_sales_overview` — naver_smartstore_orders → erp_customer_order_list (전 채널 통합)
- `get_sales_by_brand` — 네이버 전용 → ERP 전 채널 (B2B 포함, 15개 브랜드)
- `get_period_comparison` — ERP 전 채널 기반 기간 비교

### 신규 도구 (2개)
1. `get_sales_target_progress` — 매출 목표 달성률 (erp_customer_month_targert_sales vs erp_annual_sales)
2. `get_product_cost_by_keyword` — 상품별 원가 검색 (키워드 → 매입가/USD/환율/마진율)

### 서버
- 74 tools, 13 guides로 확대

---

## [2026-04-07] feat | 비즈니스 질문 커버리지 확대 — 신규 도구 8개 + 가이드 3개

### 신규 도구 (8개)
1. `get_sales_by_brand` — 브랜드별 매출 집계 + 기여도
2. `get_period_comparison` — 전주/전월/전분기 자동 비교
3. `get_inventory_capital` — 전체 재고 자본 (브랜드별)
4. `get_inventory_turnover` — 재고 회전율 (EXCELLENT~POOR)
5. `get_product_lifecycle` — PLC 자동 분류 (LAUNCH/GROWTH/MATURE/DECLINE)
6. `get_fx_sensitivity` — 환율 민감도 시뮬레이션
7. `get_cannibalization_check` — 카니발라이제이션 감지
8. `get_cs_workload_forecast` — CS 부하 예측 (12개월 추이)

### 신규 가이드 (3개)
- `/cash-health` — 현금 흐름 건강 점검
- `/brand-scorecard` — 브랜드 성적표
- `/period-review` — 기간 비교 리뷰

### 서버
- 72 tools, 13 guides로 확대

## [2026-04-07] feat | 추세/체류 분석 신규 도구 2개 + inventory-health 가이드

### 신규 도구
1. `get_trend_movers` — 카탈로그 전체 판매 추세 급변 상품 (SURGE/DECLINE/INFLECTION), 노이즈 필터 내장 (min_quarterly)
2. `get_stale_inventory` — 장기 체류 재고 (DEAD/DYING/SLOW 등급 + 묶인 자본 계산)

### 신규 스킬 가이드
- eywa-mcp/guides/inventory-health.md — 재고 건강 종합 점검 (추세+체류+발주+리스크 통합)

### index.md 최적화
- 질문→도구 빠른 매핑 테이블 추가 (재고/매출/CS/광고/복합 워크플로우)
- 노이즈 필터 설명 추가

### 배포
- 서버 반영 완료: 64 tools, 10 guides

## [2026-04-07] fix+feat | 발주(PO) 버그 수정 + 리스크 검출 신규 도구 2개

### 버그 수정
1. `purchase_orders()` 키워드 필터에 po_number/product_code 누락 → 추가 (MOZA 등 PO번호 검색 가능)
2. PURCHASE_ORDERS 외 3개 SQL 기간 제한 180일 → 365일 확장 (장기 미입고 PO 누락 방지)

### 신규 도구
1. `get_po_brand_summary` — 브랜드별 발주 요약 (상태 집계 + 리스크 플래그)
2. `get_po_risk_check` — 발주 리스크 자동 검출 (LONG_PENDING/STALE_DRAFT/PARTIAL_IMPORT/NO_DUE_DATE)

### 신규 스킬 가이드
- eywa-mcp/guides/pi-risk-check.md — PI 리스크 점검 워크플로우

### 배포
- 서버 반영 완료: 62 tools, 9 guides

## [2026-04-07] ingest | Notion PI 업데이트 문서 인제스트
- 소스: Notion "📋 PI 업데이트" (33bc0e218a4f815c8dacf098c2846870)
- 생성: wiki/summaries/pi-update-overview.md
- 내용: 7개 브랜드(Keychron, Aiper, ZOYO, LUMI, TrackRacer, MOZA, Playseat) PI/발주/입고 현황
- index.md 갱신: Summaries 17개, 총 70페이지

## [2026-04-06] init | Wiki initialized
- 비즈니스 위키 초기 구조 생성
- 디렉토리: raw/, wiki/summaries/, wiki/entities/, wiki/concepts/
- CLAUDE.md 스키마 작성

## [2026-04-06] bulk-ingest | Akaxa vault → raw/ 일괄 저장
- akaxa.space vault 141개 항목 중 116개 마크다운 파일로 저장
- 제외: PEM 키 3개, 토큰/크리덴셜 3개, 테스트 데이터 5개 등 (보안/무의미)
- 카테고리: auto-email, claude-team, eywa, keychron, akaxa-marketing, claude-assistant 등
- 인제스트 대기 상태 (wiki 페이지 미생성)

## [2026-04-06] local-copy | 로컬 Claude 생성 파일 복사
- 로컬 Mac에서 Claude가 생성한 md 파일 34개 추가 복사
- CLAUDE.md 2개: tobe-repo, kychr.space
- Plan 파일 8개: rhkorea 리부트, auto-dm 기획, kychr 대시보드 등
- Skill 파일 8개: tobe-repo 스킬 (inventory-check, cs-pending, trend 등)
- AGENTS.md 2개: rhkorea, auto-dm
- Memory 2개: auto-dm GSD-2 방법론
- 전체 raw 파일: 152개

## [2026-04-06] github-ingest | GitHub 11개 레포 md 인제스트
- 대상 레포: kychr.space, auto-email, eywa-queue, tobe-mcp-improvement, akaxa.space, claude-team, weekly-inventory, tbe.kr, tbnws-ai, tbnws-ad, gfa-auto
- GitHub API로 전체 md 파일 목록 조회 → 중복 제거 후 40개 신규 파일 다운로드
- 새 Entity 3개: eywa-queue, tobe-mcp-improvement, weekly-inventory
- 기존 Entity 4개 업데이트: auto-email(Claude Team 에이전트 구성, 모바일 개편, v2.8-2.9), akaxa-space(활용 사례), tobe-mcp(개선 로드맵), eywa(분리 프로젝트)
- 전체 raw/ 347개, 위키 52개 페이지
- index.md 최종 갱신

## [2026-04-06] bulk-ingest-2 | 155개 추가 소스 인제스트
- collected-md/(186), tobe-macmini/(29), wooho-studio/(67) 총 282개 md 파일 스캔
- 중복 제거: 이미 raw/에 있음 8개, 폴더 간 중복 119개 제거
- 고유 신규 155개 파일 raw/에 복사 (전체 raw/ 307개)
- 새 Entity 8개: tbnws-ad, sabang-dashboard, keychron-launcher, tobe-ai, tbnws-admin, gfa-ops-ai, pawswing, aiper
- 새 Concept 8개: gfa-funnel-framework, messy-middle, attribution-modeling, ad-decay-saturation, product-code-hierarchy, confidence-routing, sales-data-pipeline, telegram-integration
- 새 Summary 6개: auto-email-docs-overview, knowledge-base-overview, tbnws-ad-sessions-overview, gfa-intelligence-overview, tobe-ai-schema-overview, sabang-overview
- index.md 갱신 (49개 위키 페이지 카탈로그)

## [2026-04-06] ingest | 152개 소스 일괄 인제스트
- Entity pages 11개 생성: auto-email, claude-team, eywa, claude-assistant, keychron-supply-tracker, kychr-space, akaxa-space, claudex, tobe-mcp, rhkorea, auto-dm
- Concept pages 6개 생성: bridge-architecture, mailbox-architecture, claude-team-bible, ci-cd-pipeline, gsd2-methodology, testing-strategy
- Summary pages 10개 생성: 그룹별 통합 요약 (auto-email, claude-team, tester, eywa, claude-assistant, keychron, akaxa, plans, skills, misc)
- index.md 전체 갱신 (27개 위키 페이지 카탈로그)
- 총 27개 위키 페이지, 152개 소스 기반

## [2026-04-07] wiki-mcp | Wiki MCP 서버 구축
- wiki-mcp 서버 프로젝트 생성: /Users/yangwooho/wiki-mcp/
- Python 3.13 + MCP SDK 1.27, stdio + SSE 이중 모드
- MCP 도구 10개: user 5 (search, read, list, graph, contribute) + admin 5 (edit, delete, approve, reject, ingest)
- 역할 기반 권한: user(읽기/기여), admin(전체 CRUD), Bearer 토큰 인증
- 위키 인덱서: frontmatter 파싱, 키워드 검색, 태그 필터, [[backlink]] 그래프
- 기여 플로우: contribute → raw/contributions/ → admin approve → wiki/ 반영
- 조직별 문서 15개 생성: wiki/org/{cs,sales,marketing,as,md}/ 각 index/onboarding/faq
- 직원 셋팅 가이드: wiki-mcp/docs/setup-guide.md
- CLAUDE.md, index.md 갱신 (67개 위키 페이지)
