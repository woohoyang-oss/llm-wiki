---
title: 사방넷 대시보드 종합 요약
tags: [sabang, dashboard, automation]
created: 2026-04-06
updated: 2026-04-06
sources:
  - sabang--docs--architecture.md
  - sabang--docs--database.md
  - sabang--docs--deployment.md
  - sabang--docs--setup-guide.md
  - sabang--docs--ui-design.md
  - sabang--docs--session-2026-03-08.md
  - sabang--docs--session-2026-03-09.md
  - sabang--skills--SKILL.md
  - claude--sabang--CLAUDE.md
  - claude--sabang--sabang.md
  - claude--sabang--작업.md
---

# 사방넷 대시보드 종합 요약

## 개요

사방넷 대시보드는 [[keychron]] / [[gtgear]]의 멀티채널 이커머스 주문 관리 도구이다. 사방넷(Sabangnet) API를 통해 지마켓, 옥션, 11번가, 쿠팡, 29CM 등 외부 마켓플레이스의 주문/배송/재고를 통합 관리한다. Soldout(품절) 자동화 기능을 포함한다.

## architecture.md: 시스템 아키텍처

### 전체 구조

- 프론트엔드: Next.js + TypeScript
- 백엔드: Node.js API 서버
- 데이터: PostgreSQL + Redis 캐시
- 외부 연동: 사방넷 API, [[naver-smartstore]] API

### 주요 모듈

- **주문 수집기**: 사방넷 API 폴링으로 신규 주문 자동 수집
- **주문 처리기**: 주문 확인, 송장 입력, 배송 상태 업데이트
- **재고 동기화**: ERP 재고 -> 사방넷 재고 실시간 반영
- **품절 관리**: 재고 0 도달 시 자동 품절 처리

### 데이터 흐름

1. 사방넷 API에서 채널별 주문 수집
2. 주문 정규화 및 DB 저장
3. ERP 연동: 주문 정보 전달, 재고 차감
4. 송장 정보 수신 후 채널별 자동 전송
5. 배송 완료 추적 및 상태 업데이트

## database.md: 데이터베이스

### 주요 테이블

- **sabang_orders**: 사방넷 통합 주문 (채널, 주문번호, 상태, 금액)
- **sabang_products**: 채널별 상품 매핑 (채널 상품코드 <-> G-code)
- **sabang_inventory**: 채널별 재고 현황
- **sabang_shipments**: 송장/배송 정보
- **sabang_sync_log**: 동기화 이력 및 에러 로그

### 데이터 연관

- [[tobe-ai-schema-overview]] 4-Layer 구조의 Layer 1 데이터소스
- G-code 기준으로 ERP 상품/재고 테이블과 JOIN
- tbnws_sabangnet_order 테이블과 동기화

## deployment.md: 배포

- Docker 기반 컨테이너 배포
- 환경 변수 관리: .env 파일 구성
- Nginx 리버스 프록시 설정
- PM2 프로세스 매니저로 Node.js 데몬 관리
- CI/CD: GitHub Actions 기반 자동 배포

## setup-guide.md: 셋업 가이드

- 로컬 개발 환경 구성 순서
- Node.js/PostgreSQL/Redis 설치
- 사방넷 API 키 발급 및 설정
- DB 마이그레이션 실행
- 개발 서버 실행 및 테스트

## ui-design.md: UI 디자인

### 주요 화면

- **대시보드**: 채널별 일별 매출/주문수 요약
- **주문 목록**: 필터(채널, 상태, 날짜), 검색, 일괄 처리
- **주문 상세**: 주문 정보, 배송 추적, 메모
- **상품 관리**: 채널별 상품 매핑, 가격 관리
- **재고 현황**: 통합 재고 뷰, 품절 알림

### 디자인 원칙

- 최소한의 클릭으로 핵심 작업 완료
- 대량 처리 지원: 체크박스 선택 후 일괄 송장 입력
- 실시간 상태 반영: WebSocket 기반 업데이트

## 개발 세션 기록

### session-2026-03-08

- 초기 프로젝트 셋업 및 아키텍처 구현
- 사방넷 API 연동 모듈 개발
- 주문 수집 및 DB 저장 파이프라인 구축
- 기본 UI 프레임 구성

### session-2026-03-09

- 주문 목록 페이지 구현
- 필터링/검색 기능 개발
- 송장 입력 및 배송 상태 업데이트
- 재고 동기화 로직 개선

## Soldout 자동화 스킬 (SKILL.md)

### 기능

- 재고 0 도달 시 사방넷 API를 통해 채널별 자동 품절 처리
- 입고 시 자동 판매 재개
- 스케줄 기반 재고 체크 (매 시간)

### 동작 방식

1. ERP 재고 테이블 모니터링
2. 재고 0 감지 -> 해당 상품의 전 채널 품절 처리 API 호출
3. 입고 감지 -> 채널별 판매 재개 API 호출
4. 처리 결과 로그 기록 및 알림 발송

### 주의사항

- 채널별 API 호출 제한(Rate Limit) 준수
- 부분 품절(옵션별) 처리 지원
- 에러 발생 시 재시도 로직 (최대 3회)

## 관련 페이지

- [[gfa-intelligence-overview]] - GFA 인텔리전스 (사방넷 매출 데이터 활용)
- [[tobe-ai-schema-overview]] - ToBe AI 데이터 스키마
- [[keychron]] - 키크론 브랜드
- [[naver-smartstore]] - 네이버 스마트스토어
- [[knowledge-base-overview]] - CS 지식베이스 (GTGear 업무 가이드)
