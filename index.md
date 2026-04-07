---
title: Wiki Index
updated: 2026-04-07
---

# Wiki Index

위키의 모든 페이지를 카탈로그합니다. 348개 소스에서 70개 위키 페이지 생성. eywa 74 tools, 13 guides.

## 질문 → 도구/가이드 빠른 매핑

### 재고/발주 질문 → `/inventory-health` 가이드
| 질문 | 도구 |
|------|------|
| 발주는 정상인가? | `get_po_risk_check` + `get_po_brand_summary` |
| 발주해야 하는데 안한 건? | `get_smart_reorder(action="REORDER")` |
| DRAFT만 있고 확정 안한 건? | `get_smart_reorder(action="DRAFT_ONLY")` |
| 갑자기 잘 팔리는 상품은? | `get_trend_movers(signal="SURGE")` |
| 갑자기 안 팔리는 상품은? | `get_trend_movers(signal="DECLINE")` |
| 재고 쌓여있는데 안팔리는 건? | `get_stale_inventory` |
| 프로모션 해야 할 상품? | `get_trend_movers(signal="DECLINE")` + `get_stale_inventory` |
| 특정 상품 발주 필요한가? | `get_trend_analysis(keyword=...)` → `get_order_recommendation(keyword=...)` |

### 매출 질문 → `/period-review` 가이드
| 질문 | 도구 |
|------|------|
| 오늘/이번주/이번달 매출? | `get_sales_overview` ★ERP 전 채널 |
| 어떤 상품이 잘 팔려? | `get_sales_by_product` |
| 브랜드별 매출 비중은? | `get_sales_by_brand` ★ERP 전 채널 |
| 전주/전월 대비 변화? | `get_period_comparison(compare="week"/"month")` ★ERP |
| 채널별 매출 비교? | `get_sales_by_channel` |
| 특정 상품 일별 판매? | `get_product_daily_sales(product_no=...)` |
| 스위치 선호도? | `get_switch_preference` |
| 이번달 목표 달성률? | `get_sales_target_progress` |

### 현금 흐름/원가 질문 → `/cash-health` 가이드
| 질문 | 도구 |
|------|------|
| 재고에 묶인 자본은? | `get_inventory_capital` |
| 환율 변동 시 원가 영향? | `get_fx_sensitivity` |
| 적자 판매 상품? | `get_margin_analysis` |
| 재고 회전율 건강한가? | `get_inventory_turnover` |
| 체류 재고 묶인 자본? | `get_stale_inventory` |
| 특정 상품 원가는? | `get_product_cost_by_keyword(keyword=...)` |

### 상품 예측 질문
| 질문 | 도구 |
|------|------|
| 상품 수명주기(PLC)? | `get_product_lifecycle` |
| 신제품이 기존 모델을 잠식? | `get_cannibalization_check(keyword=...)` |
| 옵션 간 재고 불균형? | `get_sibling_analysis(keyword=...)` |

### CS 질문 → `/cs-urgency` 가이드
| 질문 | 도구 |
|------|------|
| CS 긴급 건 있어? | `get_cs_overview` |
| 미답변 CS 건? | `get_unanswered_inquiries` |
| CS 부하 예측? | `get_cs_workload_forecast` |

### 광고 질문 → `/ad-daily` 가이드
| 질문 | 도구 |
|------|------|
| 오늘 광고 성과? | `get_gfa_realtime_kpi` |
| 광고 예산 효율? | `get_gfa_budget_insight` |
| GFA 실시간 성과? | `get_gfa_realtime_campaigns` |
| 낭비 키워드/소재? | `get_waste_keywords` + `get_gfa_waste_creatives` |

### 브랜드 분석 → `/brand-scorecard` 가이드
| 질문 | 도구 |
|------|------|
| 브랜드별 성적표? | `get_sales_by_brand` + `get_inventory_capital` + `get_inventory_turnover` |
| Keychron 의존도? | `get_sales_by_brand` |

### 복합 워크플로우 → 가이드 먼저!
| 시나리오 | 가이드 |
|----------|--------|
| 아침 브리핑 | `/morning-brief` |
| 재고 건강 종합 점검 | `/inventory-health` |
| 현금 흐름 점검 | `/cash-health` |
| 브랜드 성적표 | `/brand-scorecard` |
| 기간 비교 리뷰 | `/period-review` |
| PI 리스크 점검 | `/pi-risk-check` |
| 발주 점검 | `/reorder-check` |
| 주간 보고서 | `/weekly-report` |
| 예산 최적화 | `/budget-optimize` |

## Entities (25)

프로젝트/시스템별 엔티티 페이지.

- [[auto-email]] — AI 이메일 자동응답 시스템 (mail.tbe.kr)
- [[claude-team]] — 멀티머신 Claude 에이전트 팀 (Slack 기반)
- [[eywa]] — Claude Team 오케스트레이션 플랫폼
- [[eywa-dev-guide]] — eywa MCP 개발자 가이드 (통합 서버, WikiIndex, Streamable HTTP)
- [[eywa-queue]] — Eywa Queue 독립 태스크 큐 (SQLite, DAG)
- [[claude-assistant]] — 매니저 업무비서 시스템 (Notion 연동)
- [[claude-assistant-guide]] — Claude 비서 시스템 매니저 가이드 (하루 3회 체크인, 프로덕트 연동)
- [[keychron-supply-tracker]] — 키크론 발주(PO) 추적 시스템
- [[weekly-inventory]] — 주간 재고 리포트 자동화 (Excel→Gmail)
- [[kychr-space]] — 키크론 마켓 인텔리전스 대시보드
- [[akaxa-space]] — 크로스AI 메모리 레이어
- [[claudex]] — OpenClaude + Codex (GPT-5.4 백엔드)
- [[tobe-mcp]] — ToBe Networks MCP 스킬 (74개 도구, ERP 전 채널 통합)
- [[tobe-mcp-dev-guide]] — ToBe MCP 개발자 가이드 (3계층 아키텍처, 도구 등록 패턴)
- [[tobe-mcp-improvement]] — ToBe MCP 개선 프로젝트 (4Phase, 18Task)
- [[tobe-ai]] — ToBe AI 시스템 (Text-to-SQL, 231테이블)
- [[tbnws-admin]] — TBNWS Admin 통합 관리 플랫폼 (Spring Boot)
- [[tbnws-ad]] — TBNWS 네이버 광고 분석 대시보드 (SA+GFA+스마트스토어)
- [[gfa-ops-ai]] — GFA OPS AI 광고 자동 최적화
- [[sabang-dashboard]] — 사방넷 자동화 대시보드 (20개 쇼핑몰)
- [[keychron-launcher]] — 키크론 런처 클론 (WebHID/VIA)
- [[rhkorea]] — RH Korea 웹사이트 리부트
- [[auto-dm]] — Auto-DM 자동 DM 서비스
- [[pawswing]] — PawSwing 포즈윙 (고양이 그루밍)
- [[aiper]] — Aiper 에이퍼 (로봇 풀 클리너)

## Concepts (15)

프로젝트 횡단 개념/아키텍처 페이지.

- [[bridge-architecture]] — Slack ↔ Claude CLI 브릿지 구조
- [[mailbox-architecture]] — 포워딩+SendAs 메일박스 구조
- [[claude-team-bible]] — Claude 팀 구축 바이블 요약
- [[ci-cd-pipeline]] — CI/CD 및 배포 프로세스
- [[gsd2-methodology]] — GSD-2 자율 에이전트 방법론
- [[testing-strategy]] — E2E/Playwright 테스트 전략
- [[gfa-funnel-framework]] — GFA 퍼널 프레임워크 (STDC, 60/40)
- [[messy-middle]] — Messy Middle 탐색-평가 루프
- [[attribution-modeling]] — Attribution Modeling 기여도 모델링
- [[ad-decay-saturation]] — 광고 이월효과와 포화 모델
- [[product-code-hierarchy]] — 상품 코드 체계 (G-code)
- [[confidence-routing]] — Confidence-based Email Routing
- [[sales-data-pipeline]] — 매출 데이터 파이프라인 (4-Layer)
- [[telegram-integration]] — 텔레그램 연동 패턴
- [[eywa-mcp-guide]] — eywa MCP 사용 가이드 (읽기/쓰기/수정/삭제/스킬)

## Summaries (17)

그룹별 통합 요약 페이지.

- [[auto-email-overview]] — auto-email 전체 요약
- [[auto-email-docs-overview]] — auto-email 기술문서 요약 (19개 docs)
- [[claude-team-timeline]] — Claude Team 타임라인 (03-23~03-27)
- [[tester-results-summary]] — 테스터 결과 종합
- [[eywa-overview]] — Eywa 프로젝트 요약
- [[claude-assistant-overview]] — 비서 시스템 요약
- [[keychron-tools-overview]] — 키크론 도구 모음 요약
- [[akaxa-marketing-overview]] — Akaxa 마케팅 요약
- [[plans-overview]] — Claude 계획 파일 요약
- [[skills-overview]] — ToBe MCP 스킬 요약
- [[meetings-and-misc]] — 미팅/기타 요약
- [[knowledge-base-overview]] — CS 지식베이스 요약 (4개 브랜드)
- [[tbnws-ad-sessions-overview]] — TBNWS-AD 개발 세션 기록
- [[gfa-intelligence-overview]] — GFA 인텔리전스 & 전략
- [[tobe-ai-schema-overview]] — ToBe AI 데이터 스키마
- [[sabang-overview]] — 사방넷 대시보드 요약
- [[pi-update-overview]] — PI 업데이트 브랜드별 발주/입고 현황 (7개 브랜드)

## Organization (15)

조직별 문서 (온보딩, FAQ, 카탈로그).

### CS팀
- [[org/cs/index]] — CS팀 문서 카탈로그
- [[org/cs/onboarding]] — CS팀 온보딩 가이드
- [[org/cs/faq]] — CS팀 FAQ

### 영업팀
- [[org/sales/index]] — 영업팀 문서 카탈로그
- [[org/sales/onboarding]] — 영업팀 온보딩 가이드
- [[org/sales/faq]] — 영업팀 FAQ

### 마케팅팀
- [[org/marketing/index]] — 마케팅팀 문서 카탈로그
- [[org/marketing/onboarding]] — 마케팅팀 온보딩 가이드
- [[org/marketing/faq]] — 마케팅팀 FAQ

### AS팀
- [[org/as/index]] — AS팀 문서 카탈로그
- [[org/as/onboarding]] — AS팀 온보딩 가이드
- [[org/as/faq]] — AS팀 FAQ
- [[org/as/as-customer-faq]] — Keychron AS 고객 응대 FAQ (10개 트러블슈팅)
- [[org/as/as-repair-manual]] — AS 내부 수리 매뉴얼 (Keychron/Aiper/GTGear)

### MD(상품기획)팀
- [[org/md/index]] — MD팀 문서 카탈로그
- [[org/md/onboarding]] — MD팀 온보딩 가이드
- [[org/md/faq]] — MD팀 FAQ
