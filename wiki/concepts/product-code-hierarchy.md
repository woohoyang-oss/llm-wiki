---
title: 상품 코드 체계 (G-code)
tags: [product, erp, inventory, data-model]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# 상품 코드 체계 (G-code)

[[keychron]] / [[gtgear]] 상품의 식별·재고·주문을 관리하는 코드 체계. ERP 시스템의 근간이 되는 데이터 모델이다.

## 코드 구조

```
G - BBBB - GGGG - OOOO
│    │      │      └─ Option (4자리): 색상, 스위치, 레이아웃 등
│    │      └─ Goods (4자리): 개별 상품 모델
│    └─ Brand (4자리): 브랜드 코드
└─ Category: G(일반) 또는 F(CJ 풀필먼트)
```

### Category 접두사

| 접두사 | 물류 센터 | 용도 |
|--------|-----------|------|
| **G** | GT(자체) 물류 | 자체 배송, 직접 관리 |
| **F** | CJ 대한통운 풀필먼트 | 네이버 풀필먼트, 로켓배송 등 |

### Brand 코드

| 코드 | 브랜드 |
|------|--------|
| 0029 | [[keychron]] |
| 0027 | [[gtgear]] (지티기어) |

### 예시

```
G-0029-0150-0012
│  │     │     └─ Option: 적축(Red), 한글 각인
│  │     └─ Goods: K8 Pro
│  └─ Brand: Keychron
└─ Category: GT 물류
```

## GT + CJ 재고 합산 룰

동일 상품이 GT 물류(G-코드)와 CJ 풀필먼트(F-코드)에 분산 보관된다. 총 재고를 산출할 때는 두 코드를 합산해야 한다.

### 변환 공식

```
F-code = CONCAT('F', SUBSTRING(G-code, 2))
```

예시:
- G-코드: `G-0029-0150-0012`
- F-코드: `F-0029-0150-0012`

### 합산 재고 조회

```sql
SELECT
  g.product_code,
  g.stock_qty + COALESCE(f.stock_qty, 0) AS total_stock
FROM erp_stock g
LEFT JOIN erp_stock f
  ON f.product_code = CONCAT('F', SUBSTRING(g.product_code, 2))
WHERE g.product_code LIKE 'G%'
```

## 채널 분류

상품 판매 채널은 크게 두 가지로 구분된다.

### ON (직접판매, 온라인)

- [[naver-smartstore]] (키크론 공식 스토어)
- [[naver-smartstore]] (지티기어 스토어)
- 자사몰
- 29CM, 쿠팡, 지마켓 등 오픈마켓

### OFF (B2B / 도매)

- 기업 납품
- 리셀러 공급
- 오프라인 매장 위탁

채널별 매출은 [[sales-data-pipeline|매출 데이터 파이프라인]]을 통해 집계된다.

## 재고 관련 운영

### 발주 판단

[[sales-data-pipeline|매출 데이터]]와 재고를 결합하여 days_of_supply(잔여 일수)를 산출한다. 14일 미만이면 발주 경고가 발생한다.

### 모델 전환 / 단종

- 후속 모델 출시 시 기존 G-code는 유지하되 판매 상태를 변경
- 재고 소진 후 단종 처리
- 단종 상품의 AS는 [[keychron]] AS 프로세스를 통해 계속 지원

## 관련 개념

- [[sales-data-pipeline]] — 매출 데이터 4-Layer 구조
- [[confidence-routing]] — 상품 코드 기반 자동 이메일 라우팅
