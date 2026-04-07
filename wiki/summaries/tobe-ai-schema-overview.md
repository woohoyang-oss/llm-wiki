---
title: ToBe AI 데이터 스키마 종합 요약
tags: [tobe-ai, schema, database, ai]
created: 2026-04-06
updated: 2026-04-06
sources:
  - tobe-ai--tbnws-ai--01_DATABASE_SCHEMA.md
  - tobe-ai--tbnws-ai--02_WEB_PAGES_ANALYSIS.md
  - tobe-ai--tbnws-ai--03_AI_SYSTEM_PLAN.md
  - tobe-ai--tbnws-ai--04_AI_QUERY_INDEX.md
  - tobe-ai--tbnws-ai--05_DATA_STRUCTURE.md
  - tobe-ai--tbnws-ai--06_DEMAND_FORECAST.md
  - tobe-ai--tbnws-ai--07_NOISE_FILTER.md
  - tobe-ai--tbnws-ai--08_API_AUTH.md
  - tobe-ai--tbnws-ai--QA_VERIFICATION_2026_03_01.md
  - tobe-ai--tbnws-ai--SESSION_2026_03_01.md
  - tobe-ai--tbnws-ai--slow_query.md
  - claude--tobe-ai--CLAUDE.md
---

# ToBe AI 데이터 스키마 종합 요약

## 개요

ToBe AI는 [[keychron]] / [[gtgear]] 통합 관리 시스템(TBNWS)의 AI 레이어이다. 231개 테이블을 보유한 대규모 데이터베이스 위에 Router + Specialist 구조의 AI 시스템을 구축한다. 총 12개 문서로 시스템 아키텍처, 데이터 구조, AI 질의 체계, 수요 예측, 노이즈 필터 등을 정의한다.

## 01_DATABASE_SCHEMA: 데이터베이스 스키마

### 규모

- **총 231개 테이블**
- ERP, CRM, 주문, 재고, 광고, CS 등 전 영역 커버

### 주요 테이블 그룹

- **주문 관련**: orders, order_items, order_status_history
- **상품 관련**: erp_product_info, erp_stock, product_prices
- **광고 관련**: naver_ad_gfa_campaign_daily, naver_ad_sa_keyword_daily
- **CS 관련**: naver_store_customer_inquiries, naver_smartstore_qna
- **재고/발주**: erp_order_item (발주), erp_stock (재고)
- **고객**: customer_info, customer_orders
- **정산**: settlement, revenue_reports

### 데이터 연관 관계

- 상품 코드(G-code)를 기준으로 ERP-주문-재고-광고 연결
- 주문번호로 CS 문의-주문-배송 추적
- 캠페인 ID로 광고 성과-매출 어트리뷰션

## 02_WEB_PAGES_ANALYSIS: 웹 페이지 분석

- TBNWS 관리자 웹앱의 페이지 구조 분석
- 각 페이지별 데이터 소스 매핑
- 프론트엔드-백엔드 API 엔드포인트 정리
- 페이지별 필요 쿼리 목록 도출

## 03_AI_SYSTEM_PLAN: AI 시스템 설계

### Router + Specialist 아키텍처

- **Router**: 사용자 질의를 분석하여 적절한 Specialist로 라우팅
- **Specialist**: 도메인별 전문 AI 에이전트
  - 매출 분석 Specialist
  - 재고/발주 Specialist
  - 광고 성과 Specialist
  - CS/클레임 Specialist
  - 종합 리포트 Specialist

### 질의 처리 플로우

1. 사용자 자연어 질의 입력
2. Router가 의도 파악 및 Specialist 선택
3. Specialist가 필요한 데이터 쿼리 실행
4. 결과 조합 및 자연어 응답 생성

### 컨텍스트 관리

- 세션 기반 대화 히스토리 유지
- 멀티턴 질의 지원: 이전 답변 기반 후속 질문 처리
- 암묵적 컨텍스트: 날짜 미지정 시 최근 30일 기본 적용

## 04_AI_QUERY_INDEX: 질의 매핑

### 질의 카테고리

- **매출 질의**: 일별/주별/월별 매출, 상품별 매출, 채널별 매출
- **재고 질의**: 현재 재고, 발주 필요 상품, 재고 소진 예측
- **광고 질의**: GFA/SA 성과, ROAS, 키워드 분석, 낭비 키워드
- **CS 질의**: 미답변 문의, 클레임률, AS 현황
- **종합 질의**: 대시보드 KPI, 경영 리포트

### 질의-쿼리 매핑

- 각 자연어 질의 패턴에 대응하는 SQL 쿼리 템플릿
- 파라미터 바인딩: 날짜, 상품코드, 채널명 등 동적 치환
- 복합 질의 분해: 여러 테이블 JOIN이 필요한 질의 처리

## 05_DATA_STRUCTURE: 4-Layer 데이터 구조

### Layer 1: Raw Data (원본 데이터)

- 외부 API에서 수집한 원본 데이터
- 네이버 광고 API, 스마트스토어 API, ERP 동기화
- 정제 없이 원본 그대로 저장

### Layer 2: Cleaned Data (정제 데이터)

- 중복 제거, 타입 변환, NULL 처리
- 코드 표준화 (G-code, F-code 매핑)
- 날짜/시간 포맷 통일

### Layer 3: Aggregated Data (집계 데이터)

- 일별/주별/월별 집계 테이블
- 상품별, 채널별, 캠페인별 요약 통계
- 계산 지표: ROAS, 마진율, 재고 회전일수

### Layer 4: Insight Data (인사이트 데이터)

- AI 분석 결과 저장
- 예측 데이터: 수요 예측, 재고 소진 예측
- 이상 탐지 결과: 급등/급락 알림
- 추천 데이터: 발주 추천, 예산 재배분 추천

## 06_DEMAND_FORECAST: 수요 예측

- 주간 판매 트렌드 기반 수요 예측 모델
- 시즈널 패턴 반영: 월별/요일별 가중치
- 리드타임 고려: 기본 35일 발주 리드타임
- 안전 재고 계산: 수요 변동성 기반
- 기존 발주(미입고 PO) 자동 반영

## 07_NOISE_FILTER: 노이즈 필터링

- 발주 판단 시 노이즈 제거 로직
- 모델 전환/단종 상품 자동 감지 및 제외
- 채널 불일치: 특정 채널 전용 상품 구분
- 시즈널 상품: 시즌 외 기간 과잉 발주 방지
- 소량 판매 품목: 최소 월판매량 기준 필터
- [[knowledge-base-overview]] 상품 정보 연동

## 08_API_AUTH: API 인증

- MCP(Model Context Protocol) 서버 인증 체계
- JWT 토큰 발급 및 검증
- API Rate Limiting 설정
- IP 화이트리스트 관리
- 에러 응답 코드 정의

## QA_VERIFICATION: 품질 검증

- AI 응답 정확도 검증 테스트 결과 (2026-03-01)
- 질의 유형별 정답률 측정
- 오답 패턴 분석 및 개선 사항 도출
- 엣지 케이스: 데이터 없는 기간 질의, 모호한 표현 처리

## SESSION 노트

- 2026-03-01 개발 세션 기록
- 슬로우 쿼리 최적화: 인덱스 추가, 쿼리 리팩토링
- N+1 문제 해결 사례
- 캐시 전략: 자주 조회되는 집계 데이터 Redis 캐시

## 관련 페이지

- [[tbnws-ad-sessions-overview]] - TBNWS-AD 개발 세션
- [[gfa-intelligence-overview]] - GFA 인텔리전스 전략
- [[keychron]] - 키크론 브랜드
- [[knowledge-base-overview]] - CS 지식베이스
