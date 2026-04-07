---
title: 매출 데이터 파이프라인 (4-Layer)
tags: [data-pipeline, sales, erp, naver]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# 매출 데이터 파이프라인 (4-Layer)

[[keychron]] / [[gtgear]] 매출 데이터가 수집·가공되는 4단계 계층 구조. 데이터 소스 간 우선순위와 용도가 명확히 구분된다.

## Layer 구조 개요

```
Layer 1 (최우선)   naver_smartstore_orders     실시간 주문
Layer 2            naver_commerce API          상품·고객 보강
Layer 3            tbnws_sabangnet             19개 채널 통합
Layer 4 (보조)     erp_customer_order_list     수동 입력
```

## Layer 1: naver_smartstore_orders

- **데이터 소스**: [[naver-smartstore]] 주문 API
- **갱신 주기**: 실시간 (수분 단위)
- **주요 필드**: 주문번호, 상품명, 수량, 실거래가(total_price), 주문일시
- **용도**: 일별 매출 집계, 상품별 판매 랭킹, 실시간 대시보드
- **우선순위**: 매출 분석의 기본 데이터 소스

### 실거래가 기준

`total_price / quantity`로 산출하는 실거래가가 기준이다. `erp_product_info.actual_price`는 관리자 설정값이므로 실거래가와 다를 수 있다.

## Layer 2: naver_commerce API

- **데이터 소스**: 네이버 커머스 API
- **갱신 주기**: 일 1회 배치
- **주요 필드**: 상품 상세정보, 카테고리, 리뷰, 문의
- **용도**: 상품 메타데이터 보강, 문의 답변율 추적
- **특이사항**: Layer 1과 상품번호(product_no)로 조인

## Layer 3: tbnws_sabangnet (사방넷)

- **데이터 소스**: 사방넷 주문 통합 시스템
- **갱신 주기**: 일 1~2회 배치
- **커버리지**: 19개 온라인 채널
  - 지마켓, 옥션, 11번가, 쿠팡, 위메프
  - 29CM, 무신사, SSG, 롯데ON 등
- **주요 필드**: 채널명, 주문번호, 상품코드, 결제금액
- **용도**: 채널별 매출 비교, 멀티채널 분석

### 채널 코드 매핑

사방넷 채널코드 → 내부 채널명으로 매핑 테이블이 존재한다. 신규 채널 추가 시 매핑 갱신 필요.

## Layer 4: erp_customer_order_list

- **데이터 소스**: ERP 수동 입력
- **갱신 주기**: 비정기 (담당자 수동)
- **주요 필드**: 고객사, 모델, 수량, 단가
- **용도**: B2B/도매 주문, 오프라인 매출, 특수 거래
- **특이사항**: 온라인에서 포착되지 않는 매출 보완

### 모델 단위 집계

Layer 4는 SKU가 아닌 모델 단위로 집계되는 경우가 많다. [[product-code-hierarchy|G-code]] Goods 레벨까지만 매칭되고 Option이 누락될 수 있다.

## Price Flow (가격 흐름)

상품 가격이 원가에서 판매가까지 흐르는 경로:

```
Buying Price (USD)
  → KRW Conversion (환율 적용, cur2krw > 100 필터)
  → VAT 포함 원가 (×1.1)
  → MSRP (권장소비자가)
  → Actual Selling Price (실거래가, 할인 적용 후)
```

### 마진 산출

```
마진 = avg_selling_price - (buying_price_krw × 1.1)
```

- `avg_selling_price`: Layer 1에서 산출한 평균 실거래가
- `buying_price_krw`: [[product-code-hierarchy|G-code]]별 원가 (USD → KRW)
- VAT 1.1배 적용 후 비교

## 데이터 정합성

### Layer 간 불일치 대응

| 상황 | 기준 Layer | 조치 |
|------|-----------|------|
| 일일 매출 합산 | Layer 1 | 최우선 데이터 소스 |
| 채널별 비교 | Layer 3 | 사방넷 통합 기준 |
| B2B 포함 전체 매출 | Layer 1 + 4 | 온라인 + 수동 합산 |
| 상품 메타 조회 | Layer 2 | 커머스 API 기준 |

### stat_date vs date

매출 날짜 필드는 **stat_date**를 사용한다. `date` 필드가 아닌 점에 주의. 광고 데이터와 조인할 때 날짜 키 일치 확인 필수.

## 관련 개념

- [[product-code-hierarchy]] — 상품 코드 체계
- [[gfa-funnel-framework]] — 광고 전환 매출과의 연결
- [[attribution-modeling]] — 전환 매출의 기여도 배분
