---
title: TBNWS Admin 통합 관리 플랫폼
tags: [tbnws-admin, spring-boot, erp, admin]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# TBNWS Admin 통합 관리 플랫폼

## 개요

TBNWS Admin은 [[keychron]], [[gtgear]], Aiper 등 멀티 브랜드를 통합 관리하는 ERP+CRM+HR+Finance+WMS 플랫폼이다. Spring Boot 3.4.8과 React로 구축되었으며, 9개의 DataSource와 14개의 API 도메인을 운용한다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| Backend | Spring Boot 3.4.8 (Java/Kotlin) |
| Frontend | React |
| ORM | MyBatis |
| DB | MySQL (9 DataSources) |
| 인프라 | AWS EC2 |

## 아키텍처

### Multi-DataSource 구성

9개의 DataSource를 MyBatis의 multi-datasource config로 관리한다:

| DataSource | 용도 |
|-----------|------|
| TBNWS_ADMIN | 핵심 관리 데이터 |
| TBNWS_AD | 광고 데이터 |
| TBNWS_HR | 인사/근태 |
| TBNWS_FINANCE | 재무/회계 |
| TBNWS_WMS | 창고 관리 |
| TBNWS_CRM | 고객 관리 |
| TBNWS_IMPORT | 수입/통관 |
| TBNWS_EXPORT | 수출/물류 |
| TBNWS_BATCH | 배치 처리 |

### 14 API Domains

REST API가 14개 비즈니스 도메인으로 분류된다:
- 상품 관리 (Product)
- 주문 관리 (Order)
- 재고 관리 (Inventory)
- 고객 관리 (Customer/CRM)
- 인사 관리 (HR)
- 재무/회계 (Finance)
- 광고 관리 (Advertising)
- 수입/수출 (Import/Export)
- 창고 관리 (WMS)
- AS/클레임 (After-Service)
- 배치 서비스 (Batch)
- 시스템 설정 (System)
- 인증/권한 (Auth)
- 리포트 (Report)

## 주요 기능

### ERP (Enterprise Resource Planning)
- 상품 마스터 관리 (erp_product_info)
- 발주/입고 관리 (erp_order_item, erp_stock)
- 원가 관리 및 마진 분석

### CRM (Customer Relationship Management)
- 고객 문의 관리
- 클레임 처리 (취소/반품/교환)
- AS 접수 및 처리 현황

### HR (Human Resources)
- 직원 정보 관리
- 근태 관리
- 급여 처리

### Finance
- 매출/매입 관리
- 비용 처리
- 세금계산서 관리

### WMS (Warehouse Management)
- GT 창고 + CJ 물류센터 재고 통합
- 입출고 이력 관리
- 재고 실사

### Batch Services
- 네이버 API 데이터 자동 수집
- 정산 데이터 처리
- 일/월 단위 집계 배치

## 멀티 브랜드 지원

| 브랜드 | 설명 |
|--------|------|
| [[keychron]] | 기계식 키보드 |
| [[gtgear]] | IT 주변기기 유통 |
| [[aiper]] | 로봇 풀 클리너 |

## 관련 시스템

- [[tbnws-ad]] — 광고 분석 대시보드 (TBNWS_AD DB 공유)
- [[tobe-ai]] — Text-to-SQL AI (DB 직접 조회)
- [[tobe-mcp]] — MCP 서버 연동
- [[sabang-dashboard]] — 사방넷 상품 동기화
