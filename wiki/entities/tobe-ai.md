---
title: ToBe AI 시스템 (Text-to-SQL)
tags: [tobe-ai, text-to-sql, agent, mcp]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# ToBe AI 시스템 (Text-to-SQL)

## 개요

ToBe AI는 자연어 질의를 SQL로 변환하여 비즈니스 데이터를 조회하는 AI 에이전트 시스템이다. 9개의 MySQL 데이터베이스와 1개의 PostgreSQL(pgvector) 벡터 DB를 활용하며, 231개 이상의 테이블에서 데이터를 추출한다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| AI Backend | FastAPI (Python) |
| Web Frontend | Next.js |
| ERP Backend | Spring Boot |
| 벡터 DB | PostgreSQL + pgvector |
| 관계형 DB | MySQL x 9 |
| 프로토콜 | MCP (Model Context Protocol) |

## 아키텍처

### 에이전트 구조: Router + Specialists

사용자의 질의가 들어오면 **Router Agent**가 의도를 분류하고 적절한 **Specialist Agent**로 라우팅한다.

#### 7 Specialist Agents
1. **Sales Agent** — 매출, 주문, 상품별 판매 분석
2. **Finance Agent** — 비용, 수익, 마진 분석
3. **Inventory Agent** — 재고 현황, 발주 필요 상품, 입출고
4. **HR Agent** — 인사, 근태, 조직 관련 조회
5. **Import-Export Agent** — 수입/수출, 통관, 물류
6. **Marketing Agent** — 광고 성과, 캠페인, ROAS 분석
7. **CS Agent** — 고객 문의, 클레임, AS 현황

### 4-Layer Sales Data Pipeline

매출 데이터는 4단계 레이어로 정제된다:

| Layer | 설명 | 데이터소스 |
|-------|------|-----------|
| Layer 1 | 원시 주문 데이터 | 네이버 API, 사방넷 |
| Layer 2 | 정규화된 주문 | orders 테이블 |
| Layer 3 | 집계/통계 | 일별/월별 매출 집계 |
| Layer 4 | 수동 입력 보정 | erp_customer_order_list |

### 데이터베이스 구성

9개의 MySQL 데이터베이스:
- `TBNWS_ADMIN` — ERP/CRM 통합 관리
- `TBNWS_AD` — 광고 데이터
- `TBNWS_HR` — 인사/근태
- `TBNWS_FINANCE` — 재무/회계
- `TBNWS_WMS` — 창고 관리
- 기타 도메인별 DB

1개의 PostgreSQL + pgvector:
- 테이블 스키마 임베딩
- 유사 질의 검색
- 컨텍스트 캐싱

## MCP 연동

[[tobe-mcp]]를 통해 Claude Desktop, Claude Code 등에서 직접 호출 가능:
- 광고 성과 조회 (SA/GFA)
- 매출/재고/클레임 현황
- AI 예산 재배분 제안

## 관련 시스템

- [[tobe-mcp]] — MCP 서버 (외부 AI 클라이언트 연동)
- [[tbnws-ad]] — 광고 분석 대시보드
- [[tbnws-admin]] — 통합 관리 플랫폼
- [[gfa-ops-ai]] — GFA 자동 최적화 엔진
