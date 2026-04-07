# ToBe Networks 데이터 구조 맵 (매출·원가·CS·광고)

> AI 시스템이 매출/마진/원가/CS/광고 분석 질문에 정확한 답변을 하기 위한 데이터 소스 가이드
> 최종 업데이트: 2026-02-28

---

## 1. 전체 데이터 흐름도

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ToBe Networks 가격·원가·매출 데이터 파이프라인               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐       │
│  │  해외 제조사   │────▶│  erp_order_item   │────▶│   erp_stock       │       │
│  │  (Keychron등) │     │  수입원가(USD)     │     │   개별유닛 추적    │       │
│  └──────────────┘     │  buying_price     │     │   item_seq (FK)   │       │
│         ▲              │  cur2krw (환율)    │     └───────────────────┘       │
│         │              └──────────────────┘              │                   │
│    발주(PO)                   ▲                          │                   │
│         │              item_seq (PK↔FK)                  │                   │
│  ┌──────────────┐            │                  product_code               │
│  │  erp_order    │     ┌─────┘                           │                   │
│  │  발주 마스터   │     │                                 ▼                   │
│  └──────────────┘     │                     ┌───────────────────┐           │
│                        │                     │  erp_product_info  │           │
│                        │                     │  상품 마스터 (핵심) │           │
│                        │                     │  product_code (PK) │           │
│                        │                     │  goods_code         │           │
│                        │                     │  msrp, actual_price │           │
│                        │                     └───────┬───────────┘           │
│                        │                             │                       │
│                        │                    tobe_product_code                │
│                        │                             │                       │
│                        │                ┌────────────▼──────────┐            │
│                        │                │  md_keychron_sheet     │            │
│                        │                │  ★ 네이버↔ERP 매핑    │            │
│                        │                │  naver_product_code    │            │
│                        │                │  tobe_product_code     │            │
│                        │                └────────────┬──────────┘            │
│                        │                             │                       │
│                        │                   naver_product_code                │
│                        │                    = product_no                     │
│                        │                             │                       │
│  ┌─────────────────────┴───────────────┐            │                       │
│  │         판매 채널 데이터               │            ▼                       │
│  │                                       │  ┌─────────────────────┐         │
│  │  [1순위] naver_smartstore_orders      │  │  55,398건            │         │
│  │  ★ 실거래가 (현재 운영, 최우선)        │  │  2023-12~현재         │         │
│  │  total_price / quantity = 실 단가     │  │  account: keychron,  │         │
│  │                                       │  │           gtgear     │         │
│  │  [2순위] naver_commerce_order         │  └─────────────────────┘         │
│  │  상세 정산 데이터 (동기화 중단)         │                                   │
│  │  expectedSettlementAmount 등 56필드   │                                   │
│  │                                       │                                   │
│  │  [3순위] tbnws_sabangnet_order        │                                   │
│  │  전 채널 통합 (지마켓,29CM,쿠팡 등)    │                                   │
│  │                                       │                                   │
│  │  [4순위] erp_customer_order_list      │                                   │
│  │  수동 입력 (판매분석 대시보드용)        │                                   │
│  └───────────────────────────────────────┘                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 판매 데이터 소스 4대 계층

### Layer 1 (1순위): naver_smartstore_orders ★ 실거래가 최우선

| 항목 | 내용 |
|------|------|
| **테이블** | `naver_smartstore_orders` |
| **데이터량** | 55,398건 |
| **기간** | 2023-12-21 ~ 현재 (실시간 동기화) |
| **용도** | ★ 실거래가 조회 최우선 소스 |
| **장점** | 현재 운영 중, 실 결제 금액 반영 |
| **한계** | 정산 세부 필드 없음 (수수료, 할인 내역 등) |

```sql
-- 핵심 필드
account              -- 스토어 구분 (keychron: 53,939건, gtgear: 1,459건)
product_order_id     -- 상품주문번호 (PK)
order_id             -- 주문번호
product_no           -- ★ 네이버 상품번호 (md_keychron_sheet.naver_product_code 매핑키)
product_name         -- 상품명
quantity             -- 수량
total_price          -- ★ 총 결제금액
product_order_status -- 주문상태 (PURCHASE_DECIDED, DELIVERED, DELIVERING, PAYED, CANCELED 등)
order_date           -- 주문일시
pay_date             -- 결제일시

-- 유효 주문 필터
WHERE product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')

-- 실 단가 계산
실거래 단가 = total_price / quantity
```

### Layer 2 (2순위): naver_commerce_order — 상세 정산 데이터

| 항목 | 내용 |
|------|------|
| **테이블** | `naver_commerce_order` |
| **데이터량** | 19,332건 |
| **기간** | 2025-01-13 ~ 2026-01-02 (⚠️ 동기화 중단) |
| **용도** | 정산 분석, 수수료/할인 상세, G-code 직접 매핑 |
| **장점** | sellerProductCode = G-code 직접 연결, 정산예정액, 56개 상세필드 |
| **한계** | 2026-01-02 이후 데이터 없음 |

```sql
-- 핵심 필드
sellerProductCode          -- ★ G-code 직접 매핑 (erp_product_info.goods_code)
productName                -- 상품명
productOption              -- 옵션명
quantity                   -- 수량
unitPrice                  -- 개별 표시가 (할인 전)
totalPaymentAmount         -- 실결제 총액
totalProductAmount         -- 상품 총액 (할인 전)
expectedSettlementAmount   -- ★ 정산 예정액 (수수료 차감 후)
paymentCommission          -- 결제 수수료
productDiscountAmount      -- 상품 할인
orderDiscountAmount        -- 주문 할인
sellerBurdenDiscountAmount -- 판매자 부담 할인
generalPaymentAmount       -- 일반 결제
naverMileagePaymentAmount  -- 마일리지 결제
deliveryFeeAmount          -- 배송비
productOrderStatus         -- 주문상태
orderDate                  -- 주문일
paymentDate                -- 결제일

-- 유효 주문 필터
WHERE productOrderStatus IN ('DELIVERING', 'PURCHASE_DECIDED', 'DELIVERED', 'PAYED')

-- 가격 관계
unitPrice × quantity = totalProductAmount (할인 전)
totalProductAmount - productDiscountAmount - orderDiscountAmount = totalPaymentAmount (실결제)
totalPaymentAmount - paymentCommission = expectedSettlementAmount (정산 예정)
```

### Layer 3: tbnws_sabangnet_order — 전 채널 통합

| 항목 | 내용 |
|------|------|
| **테이블** | `tbnws_sabangnet_order` |
| **데이터량** | 22,107건 |
| **기간** | 2024-10-27 ~ 현재 |
| **용도** | 네이버 외 채널 매출 분석 (지마켓, 29CM, 쿠팡, 11번가 등) |

```sql
-- 핵심 필드
mall_id             -- ★ 채널 구분 (ESM지마켓, 29CM, 쿠팡, 11번가, 카카오톡선물하기 등)
p_product_name      -- 상품명
p_ea                -- 수량
pay_cost            -- 총 결제금액
sale_cost           -- 판매가
total_cost          -- 총금액
order_date          -- 주문일
compayny_goods_cd   -- 자사 상품코드
model_no            -- 모델번호

-- 주요 채널 (건수순)
-- ESM지마켓: 3,763 | 29CM: 3,565 | 현대이지웰: 2,197 | 컴퓨존: 1,991
-- 쿠팡: 1,868 | 오늘의집: 1,558 | 카카오톡선물하기: 1,358 | 11번가: 1,257

-- 실 단가 계산
실거래 단가 = pay_cost / p_ea
```

### Layer 4: erp_customer_order_list — 수동 입력 (판매분석용)

| 항목 | 내용 |
|------|------|
| **테이블** | `erp_customer_order_list` |
| **용도** | 판매분석 대시보드 (sales-analysis-prototype.html), 모델 단위 집계 |
| **한계** | 수동 업로드, 모델 단위 집계, 개별 SKU 단가 부재 |

```sql
-- 핵심 필드
order_date     -- 주문일자
partner_name   -- 파트너(거래처)명
product_name   -- 상품명
ea             -- 수량 (환불 시 음수)
price          -- 총금액 (단가 × 수량, 환불 시 음수)

-- ⚠️ price = 라인 총액 (NOT 단가)
-- 평균 단가 = SUM(price) / SUM(ea)
```

---

## 3. 원가 데이터 조회 체계

### 수입원가 조회 경로 (검증 완료)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    원가 조회 체인 (Verified)                                │
│                                                                          │
│  erp_product_info.product_code (G-code)                                 │
│         │                                                                │
│         ▼                                                                │
│  erp_stock.product_code = G-code                                        │
│  + erp_stock.buying_price > 0                                           │
│  + ORDER BY import_date DESC LIMIT 1  ← 최신 입고건                     │
│         │                                                                │
│         │ item_seq (FK)                                                  │
│         ▼                                                                │
│  erp_order_item.item_seq = erp_stock.item_seq                           │
│  + cur2krw > 100  ← 비정상 환율 필터                                     │
│         │                                                                │
│         ▼                                                                │
│  원가 계산:                                                              │
│    외화원가 = erp_stock.buying_price / erp_order_item.cur2krw  (USD)     │
│    원화원가 = erp_stock.buying_price                           (KRW)     │
│    원가VAT = erp_stock.buying_price × 1.1                     (KRW+VAT) │
│                                                                          │
│  ⚠️ erp_stock.buying_price는 이미 KRW 환산값                            │
│  ⚠️ USD 원가 = buying_price / cur2krw                                   │
│  ⚠️ cur2krw > 100 필터 필수 (1.00 등 비정상 데이터 존재)                  │
│  ⚠️ VAT 10%는 DB에 미반영, 계산 시 × 1.1 적용 필요                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 원가 조회 SQL (검증 완료)

```sql
-- 특정 G-code의 최신 수입원가 조회
SELECT
    s.product_code,
    s.buying_price AS buying_krw,                           -- 원화 원가
    s.buying_price / oi.cur2krw AS buying_usd,             -- 외화 원가 (USD)
    oi.cur2krw AS exchange_rate,                            -- 적용 환율
    ROUND(s.buying_price * 1.1) AS cost_with_vat,          -- 원가 + VAT
    s.import_date                                           -- 입고일
FROM erp_stock s
JOIN erp_order_item oi ON s.item_seq = oi.item_seq
WHERE s.product_code = 'G-0029-0509-0001'  -- 예: 키크론 EX75
  AND s.buying_price > 0
  AND oi.cur2krw > 100                      -- 비정상 환율 제외
ORDER BY s.import_date DESC
LIMIT 1;
```

### 원가 조회 대안 경로 (erp-product.xml 기반)

```sql
-- selectGoodsPriceList (erp-product.xml:547) 패턴
-- LATERAL JOIN으로 상품 목록과 최신 원가를 한번에 조회
SELECT
  pi.product_code, pi.brand_name, pi.goods_name, pi.option_name,
  pi.msrp,              -- 권장소비자가
  pi.actual_price,      -- 관리자 설정 판매가 (⚠️ ≠ 실거래가)
  pi.purchase_price,    -- 사입가 (관리자 설정)
  lat.buying_krw,       -- 최신 수입원가 (KRW)
  lat.buying_usd,       -- 최신 수입원가 (USD)
  lat.exchange_rate     -- 적용 환율
FROM erp_product_info pi
LEFT JOIN LATERAL (
    SELECT s.buying_price AS buying_krw,
           s.buying_price / oi.cur2krw AS buying_usd,
           oi.cur2krw AS exchange_rate
    FROM erp_stock s
    JOIN erp_order_item oi ON s.item_seq = oi.item_seq
    WHERE s.product_code = pi.product_code
      AND s.buying_price > 0
      AND oi.cur2krw > 100
    ORDER BY s.import_date DESC LIMIT 1
) lat ON TRUE
WHERE pi.goods_code = '0509';  -- 상품군 필터
```

---

## 4. 가격 데이터 소스 계층

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    가격 정보 계층 구조 (수정 완료)                         │
│                                                                         │
│  [Level 0] 수입원가 (COGS)                                              │
│  erp_stock.buying_price × 1.1(VAT)                                     │
│  USD 환산: buying_price / erp_order_item.cur2krw                       │
│  예: buying_price=56,753원 / cur2krw=1,621.5 = $35 × 1.1 = 62,428원   │
│                                                                         │
│  [Level 1] 상품 마스터 가격 (관리자 설정)                                │
│  erp_product_info                                                       │
│  ├─ msrp = 권장소비자가 (제조사 정가, 고정)                              │
│  ├─ actual_price = 관리자 설정 기준가 (⚠️ 실거래가 아님!)                │
│  ├─ fixed_price = 정가                                                  │
│  ├─ purchase_price = 사입가                                             │
│  └─ channel_price = 채널 상시가                                         │
│                                                                         │
│  [Level 2] 할인/채널별 설정가                                           │
│  ├─ erp_discounted_products.price (관리자 할인 설정가)                   │
│  └─ erp_channel_price.channel_price (채널별 설정가)                      │
│  ⚠️ Level 1, 2는 관리자 설정값 — 실 거래가와 다를 수 있음!                │
│                                                                         │
│  [Level 3] ★ 실거래가 (Actual Transaction Price)                        │
│  ├─ 1순위: naver_smartstore_orders.total_price/quantity ← 현재 운영     │
│  ├─ 2순위: naver_commerce_order.totalPaymentAmount/quantity ← 동기화중단│
│  └─ 3순위: tbnws_sabangnet_order.pay_cost/p_ea ← 전 채널               │
│                                                                         │
│  [Level 4] 정산가 (Settlement)                                          │
│  naver_commerce_order.expectedSettlementAmount                          │
│  → 네이버 수수료 차감 후 실제 판매자 수취액                               │
│  ⚠️ 2026-01-02 이후 데이터 없음 (동기화 중단)                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

★ 영업이익 계산 시 사용해야 할 가격:
   매출 = Level 3 실거래가 (naver_smartstore_orders 최우선)
   원가 = Level 0 수입원가 (erp_stock → erp_order_item)
   ❌ Level 1 (actual_price)는 관리자 설정값이며 실거래가가 아님!
   ❌ erp_discounted_products.price도 관리자 설정값!
```

---

## 5. ★ 핵심 매핑 테이블: md_keychron_sheet

> 네이버 스마트스토어 ↔ ERP(G-code) 사이의 **브릿지 테이블**
> 925행, 331개 유니크 네이버상품코드, 681개 유니크 G-code

```sql
-- md_keychron_sheet (925행)
id                          -- PK
naver_product_code          -- ★ 네이버 상품번호 (= naver_smartstore_orders.product_no)
tobe_product_code           -- ★ G-code (= erp_product_info.product_code)
product_name                -- 상품명
option_name                 -- 옵션명
internal_management_code    -- 내부 관리코드
group_product_code          -- 그룹 상품코드
naver_list_price            -- 네이버 정가
naver_sale_price            -- 네이버 판매가
tobe_list_price             -- 투비 정가
tobe_sale_price             -- 투비 판매가
brand_name                  -- 브랜드
product_line                -- 제품 라인
product_option              -- 제품 옵션
product_category            -- 카테고리

-- ⚠️ 1:N 관계: 하나의 naver_product_code에 여러 옵션(tobe_product_code) 매핑
-- 예: naver_product_code '10310833443' → 39개 옵션(G-code)
-- 따라서 매출 집계 시 중복 제거 필수!
```

### md_keychron_sheet 중복 제거 패턴

```sql
-- ★ 네이버 상품코드당 대표 G-code 1개만 선택 (중복 방지)
SELECT naver_product_code, MIN(tobe_product_code) AS gcode
FROM md_keychron_sheet
WHERE naver_product_code IS NOT NULL AND naver_product_code != ''
  AND tobe_product_code IS NOT NULL AND tobe_product_code != ''
GROUP BY naver_product_code;
-- 결과: 331개 → 331개 (1:1)
```

### md_keychron_sheet_attach (복합 옵션 매핑)

```sql
-- md_keychron_sheet_attach (649행) — 추가 옵션 매핑
-- md_keychron_sheet에서 1차 매핑 안 되는 복합 옵션용
product_code    -- 상품코드
option_code     -- 옵션코드
tobe_code       -- G-code
product_name    -- 상품명
option_name     -- 옵션명
```

---

## 6. ★ 검증된 JOIN 경로 (데이터 연결 맵)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   검증된 테이블 연결 맵 (2026-02 확인)                         │
│                                                                              │
│  ╔══════════════════════════════════════════════════════════════════╗        │
│  ║  경로 A: 네이버 실거래가 + G-code + 원가 (★ 핵심 조인)           ║        │
│  ║                                                                  ║        │
│  ║  naver_smartstore_orders                                         ║        │
│  ║  │  product_no  ───────────────┐                                 ║        │
│  ║  │                              │  COLLATE utf8mb4_unicode_ci    ║        │
│  ║  │                              ▼                                ║        │
│  ║  │                    md_keychron_sheet                           ║        │
│  ║  │                    │  naver_product_code = product_no         ║        │
│  ║  │                    │  tobe_product_code ───────┐              ║        │
│  ║  │                    │                            │             ║        │
│  ║  │                    │                            ▼             ║        │
│  ║  │                    │              erp_product_info            ║        │
│  ║  │                    │              │  product_code = gcode     ║        │
│  ║  │                    │              │                           ║        │
│  ║  │                    │              ▼                           ║        │
│  ║  │                    │    erp_stock (product_code)              ║        │
│  ║  │                    │    │  item_seq ──▶ erp_order_item       ║        │
│  ║  │                    │    │              (cur2krw, buying_price)║        │
│  ║  │                    │    │              + cur2krw > 100        ║        │
│  ║  ╚══════════════════════════════════════════════════════════════╝        │
│                                                                              │
│  ╔══════════════════════════════════════════════════════════════════╗        │
│  ║  경로 B: naver_commerce_order → G-code 직접 매핑                 ║        │
│  ║                                                                  ║        │
│  ║  naver_commerce_order.sellerProductCode                          ║        │
│  ║  = erp_product_info.goods_code (= G-code 전체)                  ║        │
│  ║  → 직접 매핑 (md_keychron_sheet 불필요)                          ║        │
│  ║  ⚠️ 단, 2026-01-02 이후 데이터 없음                              ║        │
│  ╚══════════════════════════════════════════════════════════════════╝        │
│                                                                              │
│  ╔══════════════════════════════════════════════════════════════════╗        │
│  ║  경로 C: 판매분석 대시보드 (수동입력 데이터)                       ║        │
│  ║                                                                  ║        │
│  ║  erp_customer_order_list                                         ║        │
│  ║  ├─ product_name → erp_customer_order_product_list.product_name ║        │
│  ║  ├─ product_id → erp_customer_order_product_list.product_id     ║        │
│  ║  └─ partner_name → erp_customer_order_partner_list.partner_name ║        │
│  ╚══════════════════════════════════════════════════════════════════╝        │
│                                                                              │
│  ⚠️ 경로 A 사용 시 주의사항:                                                │
│  1. COLLATE utf8mb4_unicode_ci 필수 (collation mismatch 방지)              │
│  2. md_keychron_sheet 1:N 관계 → 대표 G-code 중복 제거 필수                 │
│  3. erp_order_item.cur2krw > 100 필터 필수 (비정상 환율 제거)               │
│  4. Decimal → float 변환 필요 (Python pymysql 사용 시)                      │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 검증된 전체 JOIN SQL (경로 A)

```sql
-- ★ 네이버 실거래가 + G-code + 원가 통합 조회 (2026-02 검증 완료)
-- 1월 전체 매출·원가율 계산 예시

-- Step 1: 대표 G-code 매핑 (중복 제거)
WITH gcode_map AS (
    SELECT naver_product_code, MIN(tobe_product_code) AS gcode
    FROM md_keychron_sheet
    WHERE naver_product_code IS NOT NULL AND naver_product_code != ''
      AND tobe_product_code IS NOT NULL AND tobe_product_code != ''
    GROUP BY naver_product_code
)

-- Step 2: 주문 + G-code + 원가 결합
SELECT
    nso.product_name,
    gm.gcode AS g_code,
    SUM(nso.quantity) AS total_qty,
    SUM(nso.total_price) AS total_revenue,
    ROUND(AVG(nso.total_price / nso.quantity)) AS avg_selling_price,
    -- 원가 (별도 조회 권장, LATERAL JOIN 시 성능 이슈)
    MIN(nso.total_price / nso.quantity) AS min_price,
    MAX(nso.total_price / nso.quantity) AS max_price
FROM naver_smartstore_orders nso
LEFT JOIN gcode_map gm
    ON nso.product_no COLLATE utf8mb4_unicode_ci
     = gm.naver_product_code COLLATE utf8mb4_unicode_ci
WHERE nso.order_date >= '2026-01-01' AND nso.order_date < '2026-02-01'
  AND nso.product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY nso.product_name, gm.gcode
ORDER BY total_revenue DESC;
```

### 원가 배치 조회 (성능 최적화)

```sql
-- ★ G-code 목록으로 원가 일괄 조회 (LATERAL JOIN 대신 배치 방식)
-- Python에서 G-code 목록을 추출한 후 배치 조회

SELECT s.product_code, s.buying_price AS buying_krw,
       s.buying_price / oi.cur2krw AS buying_usd,
       oi.cur2krw AS exchange_rate,
       s.import_date
FROM erp_stock s
JOIN erp_order_item oi ON s.item_seq = oi.item_seq
WHERE s.product_code IN ('G-0029-0509-0001', 'G-0029-0404-0009', ...)
  AND s.buying_price > 0
  AND oi.cur2krw > 100
ORDER BY s.product_code, s.import_date DESC;
-- → Python에서 product_code별 첫 번째 행 = 최신 원가
```

---

## 7. 판매분석 페이지 API 구조

### 페이지: sales-analysis-prototype.html
> 데이터소스: `erp_customer_order_list` (수동입력, Layer 4)

| 탭 | API 엔드포인트 | 파라미터 | 반환 필드 |
|---|---|---|---|
| 월별 목표 매출 | `/api/erp/getMonthlySalesTargetData` | year, month | target, actual |
| 월별 전체 매출 | `/api/erp/getSalesSummary` | year, month, brand | on, off, total, count, ea |
| 일별 상세 매출 | `/api/erp/getSalesSummaryDaily` | startDate, endDate, brand | order_date, partner_name, type, count, ea, price |
| 모델별 비교 매출 | `/api/erp/getSalesModelList` | year | model_name, month, ea, price |

### 핵심 SQL CTE: `orderBaseDataCTE` (erp-sales.xml)
```sql
WITH order_base_data AS (
    -- Path A: product_name 매칭
    SELECT o.*, partner.type, partner.category,
           product.brand_name, product.model_name, product.layout_name
    FROM erp_customer_order_list o
    LEFT JOIN erp_customer_order_partner_list partner ON o.partner_name = partner.partner_name
    LEFT JOIN erp_customer_order_product_list product ON o.product_name = product.product_name
    WHERE EXISTS (SELECT 1 FROM erp_customer_order_product_list WHERE product_name = o.product_name)
    UNION ALL
    -- Path B: product_id fallback
    SELECT o.*, partner.type, partner.category,
           product.brand_name, product.model_name, product.layout_name
    FROM erp_customer_order_list o
    LEFT JOIN erp_customer_order_partner_list partner ON o.partner_name = partner.partner_name
    LEFT JOIN erp_customer_order_product_list product ON o.product_id = product.product_id
    WHERE NOT EXISTS (SELECT 1 FROM erp_customer_order_product_list WHERE product_name = o.product_name)
)
```

---

## 7-2. ★ 스마트스토어 상품분석 페이지 데이터 구조 (tbe.kr `?tab=ss-products`)

> 프론트엔드 모듈: `js/tabs/ss-products.js`
> 데이터소스: `naver_smartstore_orders` (1순위 실거래가)
> 최종 검증: 2026-03-04

### 전체 데이터 파이프라인

```
[tbe.kr 프론트엔드]                    [tbe.kr 백엔드]                     [MySQL DB]
                                                                          TBNWS_ADMIN
js/tabs/ss-products.js                  /api/smartstore/products/*         naver_smartstore_orders
   │                                        │                                │
   ├─ apiFetch('/api/smartstore/            ├─ /analysis                     ├─ product_perf 집계
   │    products/analysis')  ──────────────▶│   → product_perf[]             │   GROUP BY product_no
   │    → product_perf[]                    │   → product_daily[]            │
   │    → product_daily[]                   │   → weekly_summary[]           ├─ product_daily 집계
   │    → weekly_summary[]                  │   → concentration[]            │   GROUP BY stat_date,
   │    → concentration[]                   │   → total_revenue              │            product_no
   │                                        │                                │
   ├─ apiFetch('/api/smartstore/            ├─ /catalog?size=100             ├─ 스마트스토어 상품 API
   │    products/catalog')  ───────────────▶│   → products[]                 │   (네이버 커머스 연동)
   │                                        │   → total                      │
   ├─ apiFetch('/api/smartstore/            ├─ /data-freshness               │
   │    data-freshness')  ─────────────────▶│   → freshness 상태             │
   │                                        │                                │
   └─ fetchEvents()  ─────────────────────▶ calendar-events.js              │
       → 공휴일/이벤트 배열                    → 한국 공휴일 + 쇼핑 이벤트     │
```

### 핵심 SQL: product_perf (상품별 성과 집계)

```sql
-- 추정 쿼리 (naver_smartstore_orders 기반)
SELECT
    product_no,
    product_name,
    COUNT(*) AS order_cnt,
    SUM(quantity) AS quantity,
    SUM(total_price) AS revenue,
    SUM(CASE WHEN product_order_status IN ('CANCELED','RETURNED','EXCHANGED') THEN 1 ELSE 0 END) AS cancel_cnt,
    ROUND(SUM(CASE WHEN product_order_status IN ('CANCELED','RETURNED','EXCHANGED') THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS cancel_rate
FROM naver_smartstore_orders
WHERE account = %s
  AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY product_no, product_name
ORDER BY revenue DESC
```

### 핵심 SQL: product_daily (상품별 일별 데이터 — 판매추이 차트 원본)

```sql
-- 추정 쿼리 (naver_smartstore_orders 기반)
SELECT
    DATE(order_date) AS stat_date,
    product_no,
    product_name,
    SUM(total_price) AS revenue,
    COUNT(*) AS order_cnt
FROM naver_smartstore_orders
WHERE account = %s
  AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY DATE(order_date), product_no, product_name
ORDER BY stat_date, revenue DESC
```

### ★ product_no로 특정 상품 조회하는 방법

```sql
-- product_no = 네이버 상품번호 (예: '10310833443' = 키크론 K10 PRO SE 레트로)
-- ⚠️ 1:N 관계: 하나의 product_no에 여러 옵션(G-code) 매핑 가능 (최대 39개)

-- 1) 특정 상품 일별 매출 추이
SELECT
    DATE(order_date) AS stat_date,
    SUM(total_price) AS revenue,
    COUNT(*) AS order_cnt,
    SUM(quantity) AS total_qty
FROM naver_smartstore_orders
WHERE account = 'keychron'
  AND product_no = '10310833443'
  AND order_date >= '2026-01-01' AND order_date < '2026-03-04'
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY DATE(order_date)
ORDER BY stat_date;

-- 2) product_no → G-code 매핑 확인 (md_keychron_sheet 브릿지)
SELECT naver_product_code, tobe_product_code, internal_management_code
FROM md_keychron_sheet
WHERE naver_product_code = '10310833443';
-- → 39개 옵션(G-code) 반환

-- 3) product_no → 상품명 확인
SELECT DISTINCT product_name, product_no
FROM naver_smartstore_orders
WHERE product_no = '10310833443'
LIMIT 1;
-- → '키크론 K10 PRO SE 레트로 무선 기계식 키보드 사무용 저소음'
```

### 프론트엔드 차트 렌더링 로직 (sspDailyTrendChart)

```
[_renderDailyTrend(productDaily, top20, events)]
│
├─ 1. 날짜축 생성
│     allDates = [...new Set(productDaily.map(r => r.stat_date))].sort()
│     labels = allDates.map(d => "M/D" 형식)
│
├─ 2. 셀렉트 옵션 생성
│     <select id="sspProductSelect">
│       <option value="__ALL__">전체 Top 20 합계</option>
│       ${top20.map(p => <option value="${p.product_no}">${name} (${revenue})</option>)}
│     → value = product_no (문자열) 또는 '__ALL__'
│
├─ 3. 어노테이션 생성
│     annotations = buildAnnotations(events, allDates)
│     ├─ 공휴일: 빨간 점선 (rgba(186,5,23,0.22)) + 라벨 (신정, 설날, 삼일절 등)
│     └─ 이벤트: 파란 점선 (rgba(1,118,211,0.22)) + 라벨 (밸런타인데이 등)
│
├─ 4. 모드 분기
│     ├─ [__ALL__ 모드] 전체 Top 20 합계
│     │   ├─ 막대: 상품별 스택형 (y축 stacked=true)
│     │   ├─ 주문건수: 전 상품 합산 라인
│     │   └─ 7일 평균: 전체 매출 기준 이동평균
│     │
│     └─ [개별상품 모드] product_no 필터
│         ├─ 필터: productDaily.filter(r => String(r.product_no) === String(selectedNo))
│         ├─ 매출 막대: byDate[date].revenue (단일 바)
│         ├─ 주문건수 라인: byDate[date].order_cnt
│         └─ 7일 이동평균 라인: revenues[i-6..i] 평균
│
├─ 5. 7일 이동평균 계산
│     ma7 = revenues.map((_, i) => {
│         start = Math.max(0, i - 6);
│         slice = revenues.slice(start, i + 1);
│         return Math.round(slice.reduce((s, v) => s + v, 0) / slice.length);
│     });
│
└─ 6. Chart.js 설정
      type: 'bar' (mixed)
      datasets: [
        { label: "매출",        type: bar,  yAxisID: 'y',  color: rgba(1,118,211,0.5), order: 2 },
        { label: "주문건수",    type: line, yAxisID: 'y1', color: #2E8449, tension: 0.3, order: 1 },
        { label: "매출 7일 평균", type: line, yAxisID: 'y',  color: #FE9339, dash: [5,3], order: 0 }
      ]
      scales: { y: left/매출, y1: right/주문건수, x: category/날짜 }
```

### 네이버 상품번호(product_no) 주요 매핑 예시

| product_no | 상품명 | G-code(대표) | 옵션수 | 비고 |
|---|---|---|---|---|
| 10310833443 | 키크론 K10 PRO SE 레트로 무선 기계식 키보드 | G-0029-0405-0019 | 39 | 매출 1위 |
| 11868159568 | 키크론 K10 PRO MAX 무선 기계식 키보드 애플 | - | - | 매출 2위 |
| 9627735495 | 키크론 M6 무선 슬림 버티컬 마우스 인체공학 초경량 | - | - | 매출 3위 |

---

## 8. 정산 데이터 구조

### naver_commerce_order 기반 정산 CTE (erp-common.xml)
```sql
SELECT
    DATE(nc.paymentDate) AS order_date,
    SUM(nc.totalPaymentAmount) AS total_actual_amount,     -- 실결제 총액
    SUM(nc.totalProductAmount) AS total_product_amount,    -- 상품 총액
    SUM(nc.totalProductAmount) - SUM(nc.totalPaymentAmount) AS total_discount,
    SUM(nc.expectedSettlementAmount) AS total_settlement,  -- 정산 예정 총액
    SUM(nc.paymentCommission) AS total_commission,         -- 수수료 합계
    SUM(nc.deliveryFeeAmount) AS total_delivery_fee        -- 배송비 합계
FROM naver_commerce_order nc
WHERE nc.productOrderStatus IN ('DELIVERING','PURCHASE_DECIDED','DELIVERED','PAYED')
GROUP BY DATE(nc.paymentDate);
```

---

## 9. AI 시스템 질문-데이터 소스 매핑

| 질문 유형 | 데이터 소스 | 조인 경로 |
|---|---|---|
| "이 제품 실거래가는?" | naver_smartstore_orders | 경로 A (product_name LIKE) |
| "이 제품 원가는?" | erp_stock → erp_order_item | G-code로 직접 조회 |
| "이번달 마진율은?" | smartstore_orders + erp_stock | 경로 A 전체 |
| "네이버 정산 예상액은?" | naver_commerce_order | expectedSettlementAmount |
| "모델별 전체 매출?" | erp_customer_order_list | 경로 C (getSalesModelList) |
| "채널별 매출 비교?" | tbnws_sabangnet_order | mall_id 그룹핑 |
| "B2B 출고 마진?" | erp_order_recv_item | export_price - COGS |
| "할인율 높은 상품?" | naver_commerce_order | productDiscountAmount / totalProductAmount |

### 영업이익 계산 공식

```
매출총이익 = 실거래가(smartstore total_price/qty) - 원가VAT(buying_price × 1.1)
마진율(%) = (실거래가 - 원가VAT) / 실거래가 × 100
정산마진 = expectedSettlementAmount/qty - 원가VAT  (naver_commerce_order 한정)
전체원가율 = SUM(원가VAT × 수량) / SUM(매출) × 100
```

---

## 10. ★ 재고·발주 분석 데이터 체계

### 창고 체계 (2개 창고)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    창고 구조 (wms_warehouse)                                  │
│                                                                             │
│  ┌──────────────────────────────────────┐                                  │
│  │  GT (지티기어 창고)                     │                                  │
│  │  ├─ 상품코드 prefix: G-, P-, K-, Y-   │                                  │
│  │  ├─ 현황: 142,040건 / 1,337 제품      │                                  │
│  │  └─ 본사 직접 관리 & 출고              │                                  │
│  └──────────────────────────────────────┘                                  │
│                                                                             │
│  ┌──────────────────────────────────────┐                                  │
│  │  GJ (CJ 풀필먼트 창고, 곤지암)          │                                  │
│  │  ├─ 상품코드 prefix: F-               │  ⚠️ DB코드 'GJ' = CJ 풀필먼트    │
│  │  ├─ 현황: 41,965건 / 504 제품         │                                  │
│  │  ├─ 네이버 물량 대부분 여기서 3PL 출고  │                                  │
│  │  └─ F-0029-xxxx = G-0029-xxxx 동일상품│                                  │
│  └──────────────────────────────────────┘                                  │
│                                                                             │
│  ★ 동일 상품 = 코드 prefix만 다름: G-0029-0108-0040(GT) ↔ F-0029-0108-0040(GJ)│
│  ★ 한 제품이 두 창고에 동시 존재 (재고·판매 별도 집계)                        │
│  ★ 발주 분석 시 반드시 G-코드 + F-코드 합산 필요!                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 재고 데이터 소스 계층

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    재고 데이터 소스 (3계층)                                      │
│                                                                              │
│  [1순위] erp_stock_realtime (9,451행, 실시간)                                │
│  ├─ productCode = product_code                                              │
│  ├─ ea = 현재 재고                                                           │
│  ├─ beDueEa = 입고 예정                                                     │
│  ├─ oneWeekSalesEa / oneMonthSalesEa / threeMonthSalesEa = 판매량 내장      │
│  ├─ latestBuyingPrice = 최신 매입가 (KRW)                                   │
│  ├─ updatedAt = 갱신 시점 (⚠️ camelCase 컬럼명)                             │
│  └─ ⚠️ G-코드(GT) + F-코드(CJ) 별도 행 → 합산 필요!                         │
│                                                                              │
│  [2순위] erp_stock_summary (261,207행, 일별 스냅샷)                          │
│  ├─ product_code + summary_date = 복합 PK                                   │
│  ├─ ea = 현재 재고, be_due_ea = 입고 예정                                   │
│  ├─ one_week_sales_ea / one_month_sales_ea / three_month_sales_ea          │
│  ├─ 과거 트렌드 분석 가능 (날짜별 추이)                                      │
│  ├─ WHERE summary_date = (SELECT MAX(summary_date) FROM ...) 으로 최신 조회 │
│  └─ ⚠️ G-코드 + F-코드 별도 행 → 합산 필요!                                │
│                                                                              │
│  [3순위] erp_stock (184,005행 미출고, 개별 유닛 추적)                         │
│  ├─ 물리적 재고 단위 (stock_seq별 1개)                                       │
│  ├─ warehouse_tag = 'GT' 또는 'GJ' (★ 여기서 창고 구분)                     │
│  ├─ import_date / export_date = 입출고 이력                                  │
│  ├─ buying_price = 입고 시점 매입가 (KRW)                                   │
│  └─ item_seq → erp_order_item 연결 (원가 조회)                              │
│                                                                              │
│  ★ 재고 분석은 1순위(realtime) 또는 2순위(summary) 사용                      │
│  ★ 원가 분석은 3순위(stock) 사용 (item_seq JOIN 필요)                        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 안전재고 데이터

```sql
-- erp_product_info.safe_ea (6,312개 제품에 설정됨)
-- safe_ea > 0 이면 안전재고 기준 존재
-- 재고(ea) < safe_ea 이면 "안전재고 미달" 상태
SELECT product_code, product_name, safe_ea, sell_status
FROM erp_product_info
WHERE safe_ea > 0 AND sell_status = 'Y';
```

### 발주(PO) 데이터 구조

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    발주 시스템 (erp_purchase_order, 535행)                      │
│                                                                              │
│  ★ 하나의 테이블에서 order_seq 유무로 상태 구분                               │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │  임시 발주 (PO 탭)                                            │            │
│  │  WHERE order_seq IS NULL                                     │            │
│  │  현황: 101건, 18개 PO, 총 25,911개                            │            │
│  │  → 아직 실제 발주(erp_order)로 전환되지 않은 상태               │            │
│  │  → findAvailablePOrderItemList (MyBatis)                     │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                              │ updatePoSetOrderSeq()                        │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │  완료 발주 (발주현황 탭)                                       │            │
│  │  WHERE order_seq IS NOT NULL                                 │            │
│  │  현황: 434건, 34개 PO, 총 42,581개                            │            │
│  │  → erp_order에 연결됨 (order_seq = erp_order.order_seq)      │            │
│  │  → findPurchaseOrderAll (MyBatis)                            │            │
│  │  → 수정/삭제 불가 (locked)                                    │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                                                                              │
│  핵심 컬럼:                                                                 │
│  ├─ po_number: PO번호 (K260206 형식, K=키크론 + 날짜)                       │
│  ├─ product_code: G-code (= erp_product_info.product_code)                 │
│  ├─ quantity: 발주 수량                                                     │
│  ├─ partner_code: ptn0039 = 키크론 본사                                     │
│  ├─ order_seq: NULL=임시, 값있음=완료                                       │
│  └─ due_date: 입고 예정일 (대부분 NULL)                                     │
│                                                                              │
│  ⚠️ status 컬럼 없음 — order_seq NULL 여부가 유일한 상태 판별 기준           │
│  ⚠️ erp_purchase_order_backup (292행) = 백업 테이블, 구조 동일              │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### ★ 발주 필요 재고 분석 SQL (검증 완료, GT+CJ 합산)

```sql
-- ★ 핵심 쿼리: 발주 필요 재고 분석 (2026-02-28 검증 완료)
-- ⚠️ 반드시 G-코드(GT) + F-코드(CJ 풀필먼트) 합산해야 정확!
-- F-코드 = CONCAT('F', SUBSTRING(G-코드, 2))

SELECT
    g.productCode,
    p.brand_name, p.goods_name, p.option_name,
    g.ea AS gt_stock,                                    -- GT 창고 재고
    IFNULL(f.ea, 0) AS cj_stock,                         -- CJ 풀필먼트 재고
    g.ea + IFNULL(f.ea, 0) AS total_stock,               -- ★ 합산 재고
    g.beDueEa + IFNULL(f.beDueEa, 0) AS total_incoming,  -- 합산 입고예정
    IFNULL(p.safe_ea, 0) AS safe_ea,
    g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0) AS total_s1m,  -- 합산 월판매
    ROUND((g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0)) / 30.0, 1) AS daily_avg,
    CASE WHEN (g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0)) > 0
         THEN ROUND(
           (g.ea + IFNULL(f.ea, 0) + g.beDueEa + IFNULL(f.beDueEa, 0))
           / ((g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0)) / 30.0), 0)
         ELSE 9999
    END AS days_of_supply
FROM erp_stock_realtime g
JOIN erp_product_info p ON g.productCode = p.product_code
LEFT JOIN erp_stock_realtime f ON CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode
WHERE g.productCode LIKE 'G-%'
  AND ((g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0)) > 0
       OR g.threeMonthSalesEa + IFNULL(f.threeMonthSalesEa, 0) > 3)
  AND p.sell_status != 'N'
ORDER BY days_of_supply ASC;

-- 위험 등급 기준 (total_stock = GT + CJ 합산 기준):
-- 🔴 품절: total_stock = 0 AND daily_avg >= 0.3
-- 🟠 안전재고 미달: total_stock > 0 AND total_stock < safe_ea
-- 🟡 2주내 소진: days_of_supply < 14
-- 🔵 1개월내 소진: days_of_supply < 30

-- ⚠️ GT만 보면 품절로 보이지만 CJ에 재고 있는 경우 다수 존재!
-- ⚠️ 예: B6 Space Gray = GT:0 + CJ:104 → 실제 12일치 재고
```

```sql
-- ★ 이미 발주된 제품 확인 (중복 발주 방지)
SELECT product_code, SUM(quantity) AS ordered_qty,
       MAX(register_date) AS latest_po_date,
       GROUP_CONCAT(DISTINCT po_number) AS po_numbers
FROM erp_purchase_order
WHERE register_date >= DATE_SUB(CURDATE(), INTERVAL 60 DAY)
GROUP BY product_code;

-- ★ 현재 임시 발주 조회 (아직 입고 안 된 것)
SELECT po_number, product_code, quantity, register_date
FROM erp_purchase_order
WHERE order_seq IS NULL
ORDER BY register_date DESC;
```

### AI 질문-재고 데이터 매핑

| 질문 유형 | 데이터 소스 | 핵심 쿼리 |
|---|---|---|
| "재고 몇 개?" | erp_stock_realtime | G-코드 + F-코드 합산 (`g.ea + IFNULL(f.ea, 0)`) |
| "발주 필요한 재고?" | realtime + product_info | GT+CJ 합산 days_of_supply 분석 쿼리 |
| "지티창고 재고?" | erp_stock_realtime | `WHERE productCode LIKE 'G-%'` (GT만) |
| "CJ 풀필 재고?" | erp_stock_realtime | `WHERE productCode LIKE 'F-%'` (CJ만) |
| "이미 발주했어?" | erp_purchase_order | `WHERE product_code = 'G-xxxx'` |
| "임시 발주 목록?" | erp_purchase_order | `WHERE order_seq IS NULL` |
| "입고 예정?" | erp_stock_realtime.beDueEa | G+F 합산, 또는 erp_purchase_order 확인 |
| "판매 속도?" | realtime.oneMonthSalesEa | G+F 합산 후 daily_avg |
| "안전재고 미달?" | product_info.safe_ea vs realtime.ea | G+F 합산 total vs safe_ea |
| "재고 추이?" | erp_stock_summary | summary_date별 ea 추이 (G+F 합산) |

---

## 11. 데이터 갭 및 주의사항

### 존재하지 않는 테이블
| 테이블명 | 상태 |
|---|---|
| `naver_commerce_product_detail` | ❌ 존재하지 않음! 이전 문서 오류 |

### 데이터 품질 이슈
| 이슈 | 영향 | 해결방법 |
|---|---|---|
| erp_order_item.cur2krw = 1.00 | 원가 비정상 (87,365원이 $87,365로 계산) | `cur2krw > 100` 필터 적용 |
| md_keychron_sheet 1:N 관계 | 매출 중복 부풀림 (최대 39배) | naver_product_code별 GROUP BY 중복 제거 |
| Collation 불일치 | JOIN 실패 (utf8mb4_0900_ai_ci vs unicode_ci) | COLLATE utf8mb4_unicode_ci 명시 |
| Decimal × float 타입 에러 | Python pymysql에서 계산 실패 | float() 변환 후 계산 |
| 액세서리 가격 혼입 | 키보드(10만원대)와 키스킨(3,500원) 혼합 | NOT LIKE '%키스킨%' 등 필터 |

### 미매핑 영역
| 영역 | 현황 | 비고 |
|---|---|---|
| 신규 키크론 제품 (39개) | md_keychron_sheet 미등록 | K15 PRO SE ZMK, K8 PRO SE ZMK 등 |
| MOZA 제품 (56개) | md_keychron_sheet 미등록 | 레이싱 휠, 페달 등 |
| 기타 브랜드 (48개) | 파나텍, STRIX, 플레이시트 등 | |
| 물류비(택배비) | DB 미존재 | 판매자 실제 택배비 데이터 없음 |
| 광고비 배분 | DB 미존재 | ROAS 계산 불가 |

---

## 12. 주요 테이블 데이터 현황 (2026-02 기준)

| 테이블 | 행 수 | 기간 | 비고 |
|---|---|---|---|
| naver_smartstore_orders | 55,398 | 2023-12 ~ 현재 | keychron 53,939 + gtgear 1,459 |
| naver_commerce_order | 19,332 | 2025-01 ~ 2026-01 | 동기화 중단 |
| tbnws_sabangnet_order | 22,107 | 2024-10 ~ 현재 | 15개+ 채널 |
| md_keychron_sheet | 925 | - | 331 유니크 네이버코드, 681 유니크 G-code |
| md_keychron_sheet_attach | 649 | - | 복합 옵션 매핑 |
| erp_product_info | - | - | 상품 마스터 (핵심) |
| erp_stock | 2,363,960 | 2017-10 ~ 현재 | 개별 유닛 추적 |
| erp_order_item | 45,823 | - | 발주 품목 + 원가 |
| erp_discounted_products | 3,198 | 2024-10 ~ 현재 | 관리자 할인 설정 |
| erp_channel_price | 7,819 | - | 채널별 설정가 |
| naver_store_customer_inquiries | ~143/월 | 2024 ~ 현재 | 1:1 문의 (keychron/gtgear) |
| crm_as | ~560/월 접수 | 2024 ~ 현재 | AS 접수·처리 마스터 |
| crm_as_product | - | - | AS 제품별 진단·처리 상세 |
| naver_smartstore_qna | - | - | 상품 QnA (keychron/gtgear) |
| naver_ad_campaign_daily | - | 2024 ~ 현재 | SA 캠페인 일별 성과 |
| naver_ad_gfa_campaign_daily | - | 2024 ~ 현재 | GFA 디스플레이 일별 성과 |
| naver_ad_keyword_daily | - | 2024 ~ 현재 | SA 키워드 일별 성과 |
| naver_ad_bizmoney | - | - | 광고 비즈머니 잔액 |
| naver_smartstore_order_daily | - | - | 스토어 일별 매출 집계 |
| **AI_CHATBOT.naver_product** | **211** | **2026-01 ~ 현재** | **네이버 상품 정보 (가격/품절/태그)** |
| **AI_CHATBOT.products** | **160** | **-** | **제품 전체 스펙 (챗봇 핵심)** |

---

## 13. ★ CS(고객서비스) 데이터 구조 맵

### CS 전체 데이터 흐름도

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    CS 데이터 파이프라인 (2026-02 검증)                          │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │  [입구] 고객 접점 채널                                           │         │
│  │                                                                  │         │
│  │  1:1 문의 ──────▶ naver_store_customer_inquiries                │         │
│  │  (네이버 톡톡)      answered(0/1), category, processing_status  │         │
│  │                                                                  │         │
│  │  상품 QnA ──────▶ naver_smartstore_qna                          │         │
│  │  (상품 페이지)      answer(NULL=미답변), account(keychron/gtgear)│         │
│  │                                                                  │         │
│  │  주문 클레임 ───▶ naver_smartstore_orders.claim_type             │         │
│  │  (취소/반품/교환)   CANCEL, RETURN, EXCHANGE, ADMIN_CANCEL       │         │
│  │                                                                  │         │
│  │  사방넷 클레임 ─▶ tbnws_sabangnet_claim                         │         │
│  │  (멀티채널)         claim_type, claim_status                    │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                           │                                                  │
│                           ▼                                                  │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │  [처리] 내부 CRM 시스템                                          │         │
│  │                                                                  │         │
│  │  crm_as ─────────▶ AS 접수 마스터                                │         │
│  │  │  start_date        (접수일)                                   │         │
│  │  │  complete_date     (완료일, NULL=미완료)                       │         │
│  │  │  as_category       (AS유형)                                   │         │
│  │  │                                                               │         │
│  │  │  ★ 상태 판별 로직 (status 컬럼 없음!):                        │         │
│  │  │  complete_date IS NOT NULL → 완료                             │         │
│  │  │  start_date IS NOT NULL AND complete_date IS NULL → 진행중   │         │
│  │  │  둘 다 NULL → 대기                                            │         │
│  │  │                                                               │         │
│  │  └──▶ crm_as_product ─ AS 제품별 상세                            │         │
│  │       │  as_seq (FK → crm_as)                                   │         │
│  │       │  product_code (→ erp_product_info JOIN 시 상품명 조회)   │         │
│  │       │  process_type  (종결/회수중/입고 완료/수리 완료)          │         │
│  │       │  resolution_type (A/S/교환/반품/선조치)                   │         │
│  │       │  diagnose       (자유 텍스트 — 고장 패턴 분석 핵심)       │         │
│  │       │  ⚠️ product_name 컬럼 없음! erp_product_info JOIN 필수  │         │
│  │                                                                  │         │
│  │  crm_return_management ▶ 반품 처리 관리                          │         │
│  │  │  return_status, return_reason                                 │         │
│  │                                                                  │         │
│  │  naver_store_inquiry_issue ▶ 문의 이슈 트래킹                    │         │
│  │  │  inquiry_seq (FK), issue_type                                │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                              │
│  ★ CS 분석 4대 축:                                                          │
│  (1) 1:1 문의 답변율 + 미답변 건 관리                                        │
│  (2) 클레임(취소/반품/교환) 비율 + 문제 상품 식별                             │
│  (3) AS 접수·처리율 + 백로그 관리                                            │
│  (4) QnA 미답변 관리                                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### CS 핵심 데이터 소스 상세

#### (1) 1:1 문의: naver_store_customer_inquiries

```sql
-- 핵심 필드
inquiry_no              -- PK
account                 -- 스토어: keychron, gtgear
category                -- 문의 유형: 배송문의, 상품문의, 교환/환불 등
title                   -- 문의 제목
content                 -- 문의 내용
answered                -- ★ 답변여부: 0=미답변, 1=답변완료
processing_status       -- 처리상태: 접수, 처리중, 완료 등
agent_answer            -- 상담원 답변
ai_answer_generated     -- AI 자동 생성 답변 여부
created_date            -- 문의 작성일
answered_date           -- 답변일

-- ★ 미답변 건 조회 (핵심 모니터링 쿼리)
SELECT inquiry_no, account, category, title,
       created_date,
       DATEDIFF(CURDATE(), DATE(created_date)) AS days_waiting
FROM naver_store_customer_inquiries
WHERE answered = 0
ORDER BY created_date ASC;

-- ★ 월별 답변율 추이
SELECT DATE_FORMAT(created_date, '%Y-%m') AS month,
       COUNT(*) AS total,
       SUM(answered) AS answered,
       ROUND(SUM(answered)*100.0/COUNT(*), 1) AS answer_rate
FROM naver_store_customer_inquiries
WHERE account = 'keychron'
GROUP BY DATE_FORMAT(created_date, '%Y-%m')
ORDER BY month DESC;
```

#### (2) 클레임: naver_smartstore_orders.claim_type

```sql
-- ★ claim_type 값 의미
-- CANCEL = 취소, RETURN = 반품, EXCHANGE = 교환, ADMIN_CANCEL = 관리자취소

-- 클레임 현황 분석
SELECT claim_type, COUNT(*) AS cnt
FROM naver_smartstore_orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01'
  AND claim_type IS NOT NULL AND claim_type != ''
GROUP BY claim_type;

-- 반품 다발 상품 식별 (반품률 계산)
SELECT product_name,
       COUNT(*) AS total_orders,
       SUM(CASE WHEN claim_type = 'RETURN' THEN 1 ELSE 0 END) AS returns,
       ROUND(SUM(CASE WHEN claim_type = 'RETURN' THEN 1 ELSE 0 END)*100.0/COUNT(*), 1) AS ret_rate
FROM naver_smartstore_orders
WHERE order_date >= '2026-01-01'
GROUP BY product_name
HAVING total_orders >= 10
ORDER BY ret_rate DESC
LIMIT 20;

-- ⚠️ claim_type은 NULL이거나 빈 문자열이면 정상 주문
-- ⚠️ product_order_status와 함께 사용: CANCELED, RETURNED, EXCHANGED
```

#### (3) AS: crm_as + crm_as_product

```sql
-- ★ AS 상태 판별 로직 (status 컬럼 없음!)
-- complete_date IS NOT NULL → 처리 완료
-- start_date IS NOT NULL AND complete_date IS NULL → 진행 중
-- 둘 다 NULL → 대기 중

-- AS 처리율 분석
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END) AS completed,
    ROUND(SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END)*100.0/COUNT(*), 1) AS completion_rate
FROM crm_as
WHERE start_date >= '2026-02-01';

-- AS 고장 패턴 분석 (diagnose 텍스트)
-- ⚠️ crm_as_product에는 product_name 없음 → erp_product_info JOIN 필수!
SELECT p.diagnose, COUNT(*) AS cnt
FROM crm_as_product p
JOIN crm_as a ON p.as_seq = a.as_seq
WHERE a.start_date >= '2026-01-01'
  AND p.diagnose IS NOT NULL AND p.diagnose != ''
GROUP BY p.diagnose
ORDER BY cnt DESC LIMIT 20;

-- AS 처리 유형 분포
SELECT p.resolution_type, COUNT(*) AS cnt
FROM crm_as_product p
GROUP BY p.resolution_type
ORDER BY cnt DESC;
-- 예: A/S, 교환, 반품, 선조치

-- AS 제품별 건수 (erp_product_info JOIN)
SELECT pi.goods_name, pi.option_name, COUNT(*) AS as_cnt
FROM crm_as_product p
JOIN erp_product_info pi ON p.product_code = pi.product_code
JOIN crm_as a ON p.as_seq = a.as_seq
WHERE a.start_date >= '2026-01-01'
GROUP BY pi.goods_name, pi.option_name
ORDER BY as_cnt DESC LIMIT 20;
```

#### (4) QnA: naver_smartstore_qna

```sql
-- 미답변 QnA 조회
SELECT account, title, created_date
FROM naver_smartstore_qna
WHERE answer IS NULL OR answer = ''
ORDER BY created_date ASC;

-- 스토어별 미답변 집계
SELECT account, COUNT(*) AS unanswered
FROM naver_smartstore_qna
WHERE answer IS NULL OR answer = ''
GROUP BY account;
```

### CS 핵심 분석 패턴

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    CS 분석 최적 진행 순서                                       │
│                                                                              │
│  Step 1: 미답변/긴급 현황 파악                                               │
│  ├─ naver_store_customer_inquiries WHERE answered=0                         │
│  ├─ naver_smartstore_qna WHERE answer IS NULL                               │
│  └─ → 방치 기간(days_waiting) 기준 긴급도 판별                               │
│                                                                              │
│  Step 2: 클레임 비율 모니터링                                                │
│  ├─ naver_smartstore_orders.claim_type 집계                                 │
│  ├─ → 전체 대비 클레임 비율 (12% 이하면 정상)                                │
│  └─ → 반품 다발 상품 식별 (ret_rate > 10% 경고)                              │
│                                                                              │
│  Step 3: AS 백로그 확인                                                      │
│  ├─ crm_as: complete_date IS NULL 건수 (누적 백로그)                         │
│  ├─ → 이번 달 접수 vs 완료 비율 (처리율 > 70% 목표)                          │
│  └─ → crm_as_product.diagnose로 반복 불량 패턴 파악                          │
│                                                                              │
│  Step 4: 문제 상품 크로스 분석                                               │
│  ├─ 반품률 높은 상품 ↔ AS 다발 상품 ↔ QnA 다발 상품                         │
│  └─ → 동일 상품이 3개 채널에서 모두 이슈면 품질 문제                          │
│                                                                              │
│  ⚠️ 핵심 주의사항:                                                          │
│  • AS 상태는 complete_date 기준 (status 컬럼 없음!)                          │
│  • crm_as_product에 product_name 없음 → erp_product_info JOIN 필수          │
│  • claim_type은 NULL/빈문자열 = 정상 주문                                    │
│  • 답변율 추이가 하락하면 인력 부족 시그널                                    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### AI 질문-CS 데이터 매핑

| 질문 유형 | 데이터 소스 | 핵심 필드/로직 |
|---|---|---|
| "CS 이슈 뭐 있어?" | 4개 소스 전체 | 미답변 + 클레임율 + AS처리율 + QnA 순회 |
| "미답변 문의 몇 건?" | naver_store_customer_inquiries | `WHERE answered = 0` + days_waiting |
| "문의 답변율?" | naver_store_customer_inquiries | `SUM(answered)/COUNT(*)` 월별 GROUP BY |
| "반품 많은 상품?" | naver_smartstore_orders | `claim_type='RETURN'` 비율 계산 |
| "취소/클레임 현황?" | naver_smartstore_orders | `claim_type` IS NOT NULL 집계 |
| "AS 몇 건 처리?" | crm_as | `complete_date IS NOT NULL` 비율 |
| "AS 고장 패턴?" | crm_as_product | `diagnose` 텍스트 집계 + `resolution_type` |
| "AS 백로그?" | crm_as | `complete_date IS NULL` 전체 누적 |
| "QnA 미답변?" | naver_smartstore_qna | `WHERE answer IS NULL` 스토어별 |
| "문제 상품 Top10?" | orders + crm_as_product | 반품률 + AS건수 크로스 분석 |

---

## 14. ★ 광고 & 마케팅 데이터 구조 맵

### 광고 데이터 전체 흐름도

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    광고 데이터 파이프라인 (2026-02 검증)                        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │  [광고 플랫폼] 네이버 광고 시스템                                 │         │
│  │                                                                  │         │
│  │  SA (검색광고)                                                   │         │
│  │  ├── naver_ad_campaign_daily  ─ 캠페인 일별 성과                │         │
│  │  │   └─ sales_amt(비용), conv_amt(전환매출), ROAS 계산          │         │
│  │  │                                                               │         │
│  │  ├── naver_ad_keyword_daily   ─ 키워드 일별 성과                │         │
│  │  │   └─ ⚠️ 전환수 = ccnt (NOT conv_cnt!)                       │         │
│  │  │                                                               │         │
│  │  GFA (디스플레이 광고 / 성과형 디스플레이)                       │         │
│  │  ├── naver_ad_gfa_campaign_daily ─ GFA 캠페인 일별 성과        │         │
│  │  │   └─ total_cost(비용), purchase_conv_revenue(전환매출)       │         │
│  │  │   └─ campaign_name으로 퍼널 단계 분류 (TOF/MOF/BOF/RT)      │         │
│  │  │                                                               │         │
│  │  └── naver_ad_bizmoney ─ 광고 비즈머니 잔액                    │         │
│  │      └─ ⚠️ 잔액 = bizmoney (NOT balance!)                      │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                           │                                                  │
│                           ▼                                                  │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │  [매출 연동] 광고-매출 성과 분석                                   │         │
│  │                                                                  │         │
│  │  naver_smartstore_order_daily ─ 스토어 일별 매출 집계            │         │
│  │  ├─ ⚠️ 날짜 = stat_date (NOT date!)                            │         │
│  │  ├─ revenue (매출), order_cnt (주문수)                           │         │
│  │  ├─ account (keychron / gtgear)                                 │         │
│  │  └─ 광고비 대비 매출 추이 비교에 활용                             │         │
│  │                                                                  │         │
│  │  campaign_brand_map ─ 캠페인↔브랜드 매핑                        │         │
│  │  └─ campaign_name 패턴 → 브랜드 자동 분류                       │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                              │
│  ★ 광고 분석 핵심 지표:                                                     │
│  • ROAS(%) = 전환매출 / 광고비 × 100                                        │
│  • CPA = 광고비 / 전환수                                                     │
│  • CTR = 클릭수 / 노출수 × 100                                              │
│  • Binet & Field 60/40 Rule: Brand 60% : Activation 40%                    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 광고 핵심 데이터 소스 상세

#### (1) SA (검색광고): naver_ad_campaign_daily

```sql
-- 핵심 필드
campaign_name       -- 캠페인명
stat_date           -- 일자
impressions         -- 노출수
clicks              -- 클릭수
sales_amt           -- ★ 광고비 (소진액)
conv_amt            -- ★ 전환매출 (광고를 통한 매출)
account             -- keychron, gtgear

-- SA 월별 성과
SELECT
    account,
    SUM(sales_amt) AS spend,
    SUM(conv_amt) AS conv_revenue,
    ROUND(SUM(conv_amt)/NULLIF(SUM(sales_amt),0)*100, 1) AS roas_pct
FROM naver_ad_campaign_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
GROUP BY account;
```

#### (2) GFA (디스플레이 광고): naver_ad_gfa_campaign_daily

```sql
-- 핵심 필드
campaign_name             -- ★ 캠페인명 (퍼널 분류 키!)
stat_date                 -- 일자
impressions               -- 노출수
clicks                    -- 클릭수
total_cost                -- ★ 광고비
purchase_conv_revenue     -- ★ 전환매출
purchases                 -- 전환수 (구매)
account                   -- keychron, gtgear

-- ★ GFA 퍼널 분류 규칙 (campaign_name 기반)
-- TOF (Top of Funnel)    : campaign_name LIKE 'TOF%' OR LIKE '%인지%'
-- MOF (Middle of Funnel) : campaign_name LIKE 'MOF%' OR LIKE '%고려%'
-- BOF (Bottom of Funnel) : campaign_name LIKE 'BOF%' OR LIKE '%전환%'
-- ADVoost                : campaign_name LIKE 'ADVoost%' OR LIKE '%부스트%'
-- Retargeting            : campaign_name LIKE '%리타게팅%' OR LIKE '%RT%'

-- GFA 퍼널별 성과 분석
SELECT
    CASE
        WHEN campaign_name LIKE 'TOF%' OR campaign_name LIKE '%인지%' THEN 'TOF'
        WHEN campaign_name LIKE 'MOF%' OR campaign_name LIKE '%고려%' THEN 'MOF'
        WHEN campaign_name LIKE 'BOF%' OR campaign_name LIKE '%전환%' THEN 'BOF'
        WHEN campaign_name LIKE 'ADVoost%' OR campaign_name LIKE '%부스트%' THEN 'ADVoost'
        WHEN campaign_name LIKE '%리타게팅%' OR campaign_name LIKE '%RT%' THEN 'Retargeting'
        ELSE 'Other'
    END AS funnel_stage,
    SUM(total_cost) AS spend,
    SUM(purchase_conv_revenue) AS conv_revenue,
    ROUND(SUM(purchase_conv_revenue)/NULLIF(SUM(total_cost),0)*100, 1) AS roas_pct,
    ROUND(SUM(total_cost)/NULLIF(SUM(total_cost),0)*100, 1) AS budget_share
FROM naver_ad_gfa_campaign_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
  AND account = 'keychron'
GROUP BY funnel_stage
ORDER BY spend DESC;
```

#### (3) 키워드 분석: naver_ad_keyword_daily

```sql
-- 핵심 필드
keyword             -- 키워드
campaign_name       -- 캠페인명
stat_date           -- 일자
impressions         -- 노출수
clicks              -- 클릭수
sales_amt           -- 광고비
conv_amt            -- 전환매출
ccnt                -- ★ 전환수 (⚠️ conv_cnt 아님!)
account             -- keychron, gtgear

-- 전환 없는 낭비 키워드 식별
SELECT keyword,
       SUM(sales_amt) AS total_spend,
       SUM(ccnt) AS total_conv,
       SUM(clicks) AS total_clicks
FROM naver_ad_keyword_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
  AND account = 'keychron'
GROUP BY keyword
HAVING total_conv = 0 AND total_spend > 10000
ORDER BY total_spend DESC;

-- ROAS 높은 효율 키워드
-- ⚠️ conv_amt가 None일 수 있음 → IFNULL 처리 필수
SELECT keyword,
       SUM(sales_amt) AS spend,
       SUM(IFNULL(conv_amt, 0)) AS conv_revenue,
       ROUND(SUM(IFNULL(conv_amt,0))/NULLIF(SUM(sales_amt),0)*100, 1) AS roas_pct
FROM naver_ad_keyword_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
  AND account = 'keychron'
GROUP BY keyword
HAVING spend > 5000
ORDER BY roas_pct DESC LIMIT 20;
```

#### (4) 비즈머니: naver_ad_bizmoney

```sql
-- 핵심 필드
account             -- keychron, gtgear
bizmoney            -- ★ 잔액 (⚠️ balance 아님!)
updated_at          -- 갱신시점

-- 잔액 + 잔여일수 계산
-- 잔여일수 = 잔액 / 일평균 소진액
-- 일평균 소진액은 최근 7일 SA+GFA 합산으로 계산

-- Step 1: 현재 잔액
SELECT account, bizmoney FROM naver_ad_bizmoney;

-- Step 2: 일평균 소진액 (최근 7일)
SELECT
    COALESCE(sa.account, gfa.account) AS account,
    (IFNULL(sa.daily_spend, 0) + IFNULL(gfa.daily_spend, 0)) AS daily_avg_spend
FROM (
    SELECT account, ROUND(SUM(sales_amt)/7) AS daily_spend
    FROM naver_ad_campaign_daily
    WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
    GROUP BY account
) sa
LEFT JOIN (
    SELECT account, ROUND(SUM(total_cost)/7) AS daily_spend
    FROM naver_ad_gfa_campaign_daily
    WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
    GROUP BY account
) gfa ON sa.account = gfa.account;

-- Step 3: 잔여일수 = bizmoney / daily_avg_spend
-- ⚠️ Python에서 계산 권장 (SQL 단일 쿼리가 복잡)
```

#### (5) 매출-광고 연동: naver_smartstore_order_daily

```sql
-- 핵심 필드
stat_date           -- ★ 날짜 (⚠️ date 아님!)
account             -- keychron, gtgear
revenue             -- 매출
order_cnt           -- 주문수

-- 일별 매출 vs 광고비 비교 (광고효율 추이)
SELECT
    d.stat_date,
    d.revenue AS daily_revenue,
    IFNULL(sa.spend, 0) + IFNULL(gfa.spend, 0) AS daily_ad_spend,
    ROUND(d.revenue / NULLIF(IFNULL(sa.spend,0)+IFNULL(gfa.spend,0), 0), 1) AS revenue_per_adspend
FROM naver_smartstore_order_daily d
LEFT JOIN (
    SELECT stat_date, SUM(sales_amt) AS spend
    FROM naver_ad_campaign_daily WHERE account = 'keychron'
    GROUP BY stat_date
) sa ON d.stat_date = sa.stat_date
LEFT JOIN (
    SELECT stat_date, SUM(total_cost) AS spend
    FROM naver_ad_gfa_campaign_daily WHERE account = 'keychron'
    GROUP BY stat_date
) gfa ON d.stat_date = gfa.stat_date
WHERE d.account = 'keychron'
  AND d.stat_date >= '2026-02-01'
ORDER BY d.stat_date;
```

### ★ Binet & Field 60/40 프레임워크

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Binet & Field 광고 예산 배분 모델                            │
│                                                                              │
│  이론: Les Binet & Peter Field의 "The Long and the Short of It"             │
│  최적 배분: Brand Building 60% : Sales Activation 40%                       │
│                                                                              │
│  ┌────────────────────────────────────────────┐                             │
│  │  Brand Building (60% 목표)                   │                             │
│  │  = TOF (Top of Funnel) + MOF (Middle)       │                             │
│  │  → 장기 브랜드 인지도, 고려도 구축           │                             │
│  │  → ROAS 낮아도 OK (장기 투자)               │                             │
│  │                                              │                             │
│  │  현재: ~41.6% ← 목표 60% 대비 부족!        │                             │
│  └────────────────────────────────────────────┘                             │
│                                                                              │
│  ┌────────────────────────────────────────────┐                             │
│  │  Sales Activation (40% 목표)                 │                             │
│  │  = BOF + Retargeting + ADVoost              │                             │
│  │  → 단기 매출 직접 전환                       │                             │
│  │  → ROAS 높음 (직접 전환 목적)               │                             │
│  │                                              │                             │
│  │  현재: ~58.4% ← 목표 40% 대비 과다!        │                             │
│  └────────────────────────────────────────────┘                             │
│                                                                              │
│  ★ 분석 SQL:                                                                │
│  Brand% = (TOF+MOF cost) / Total GFA cost × 100                            │
│  Activation% = (BOF+RT+ADVoost cost) / Total GFA cost × 100               │
│                                                                              │
│  ★ 퍼널별 기대 ROAS 벤치마크:                                               │
│  ADVoost > Retargeting > TOF > BOF > MOF (실제 2월 데이터 기준)             │
│                                                                              │
│  ⚠️ campaign_name 패턴 매칭이 유일한 퍼널 분류 방법                          │
│  ⚠️ 새 캠페인 생성 시 네이밍 규칙이 바뀌면 분류 로직 업데이트 필요           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 광고 핵심 분석 패턴

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    광고 분석 최적 진행 순서                                     │
│                                                                              │
│  Step 1: 전체 성과 요약 (SA + GFA)                                           │
│  ├─ naver_ad_campaign_daily → SA 월별 spend/ROAS                           │
│  ├─ naver_ad_gfa_campaign_daily → GFA 월별 spend/ROAS                      │
│  └─ 합산: 총 광고비, 총 전환매출, 통합 ROAS                                  │
│                                                                              │
│  Step 2: GFA 퍼널 분석 (Binet & Field)                                      │
│  ├─ campaign_name CASE WHEN으로 퍼널 분류                                   │
│  ├─ Brand(TOF+MOF) vs Activation(BOF+RT+ADVoost) 비율 계산                 │
│  └─ 60/40 대비 현재 비율 괴리 확인                                            │
│                                                                              │
│  Step 3: 키워드 효율 분석                                                    │
│  ├─ 전환 0건 + 비용 발생 키워드 → 낭비 키워드 제거 후보                       │
│  ├─ ROAS 상위 키워드 → 예산 증액 후보                                        │
│  └─ ⚠️ ccnt(전환수) 사용, conv_cnt는 존재하지 않는 컬럼!                     │
│                                                                              │
│  Step 4: 비즈머니 잔여일수                                                   │
│  ├─ naver_ad_bizmoney.bizmoney (⚠️ balance 아님!)                          │
│  ├─ 최근 7일 일평균 소진액 계산                                              │
│  └─ 잔여일수 = 잔액 / 일평균 소진                                            │
│                                                                              │
│  Step 5: 매출-광고 상관관계                                                  │
│  ├─ naver_smartstore_order_daily.revenue (⚠️ stat_date 컬럼!)              │
│  ├─ 일별 매출 vs 광고비 추이 비교                                             │
│  └─ 광고 증액/감액 시점의 매출 변화 확인                                      │
│                                                                              │
│  ⚠️ 컬럼명 주의사항 정리:                                                   │
│  • naver_ad_keyword_daily 전환수 = ccnt (NOT conv_cnt)                     │
│  • naver_ad_bizmoney 잔액 = bizmoney (NOT balance)                         │
│  • naver_smartstore_order_daily 날짜 = stat_date (NOT date)                │
│  • conv_amt는 None/NULL 가능 → IFNULL 처리 필수                            │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### AI 질문-광고 데이터 매핑

| 질문 유형 | 데이터 소스 | 핵심 필드/로직 |
|---|---|---|
| "광고 성과 어때?" | campaign_daily + gfa_daily | SA+GFA spend/conv/ROAS 합산 |
| "SA 이번달 ROAS?" | naver_ad_campaign_daily | `SUM(conv_amt)/SUM(sales_amt)*100` |
| "GFA 퍼널 분석?" | naver_ad_gfa_campaign_daily | campaign_name CASE WHEN 분류 |
| "브랜드 vs 퍼포먼스 비율?" | naver_ad_gfa_campaign_daily | TOF+MOF vs BOF+RT+ADVoost 비율 |
| "낭비 키워드?" | naver_ad_keyword_daily | `ccnt=0 AND sales_amt>10000` |
| "ROAS 높은 키워드?" | naver_ad_keyword_daily | `IFNULL(conv_amt,0)` 사용 필수 |
| "비즈머니 며칠치?" | bizmoney + campaign + gfa | bizmoney / 일평균소진(7일) |
| "매출-광고 상관?" | order_daily + campaign + gfa | stat_date 기준 일별 조인 |
| "광고비 총액?" | campaign_daily + gfa_daily | `SUM(sales_amt) + SUM(total_cost)` |

### ★ GFA 캠페인 리네임 감지

네이버 GFA 플랫폼은 캠페인명 변경 시 DB에서 **구 캠페인 종료 + 신규 캠페인 시작**으로 기록됨.
"중단→신설"로 오인하면 예산 변동 분석 자체가 틀어짐.

```sql
-- 특정 기간에 마지막 데이터인 캠페인 = 리네임으로 사라진 구 캠페인
SELECT campaign_name, MAX(stat_date) AS last_date
FROM naver_ad_gfa_campaign_daily
GROUP BY campaign_name
HAVING last_date BETWEEN %s AND %s;

-- 같은 기간에 첫 데이터인 캠페인 = 리네임으로 생긴 신규 캠페인
SELECT campaign_name, MIN(stat_date) AS first_date
FROM naver_ad_gfa_campaign_daily
GROUP BY campaign_name
HAVING first_date BETWEEN %s AND %s;
```

> ⚠️ 두 결과의 campaign_name 패턴, result_type, 예산 규모를 대조하면 매핑 가능
> ⚠️ 이 한계는 네이버 GFA API 구조적 문제이므로 수동 매핑 테이블 관리 필요

### ★ 예산 변동 감지 & 파이프라인 Lag 분석

#### 예산 변동 감지

```sql
-- 캠페인별 최근 3일 vs 이전 3일 지출 비교 → 변동률 ±20% 이상 감지
SELECT campaign_name,
    AVG(CASE WHEN stat_date >= DATE_SUB(CURDATE(), INTERVAL 3 DAY)
             THEN total_cost END) AS recent_3d,
    AVG(CASE WHEN stat_date BETWEEN DATE_SUB(CURDATE(), INTERVAL 6 DAY)
             AND DATE_SUB(CURDATE(), INTERVAL 4 DAY)
             THEN total_cost END) AS prev_3d
FROM naver_ad_gfa_campaign_daily
WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL 6 DAY)
  AND account = 'keychron'
GROUP BY campaign_name;
-- Python에서: change_pct = (recent_3d - prev_3d) / prev_3d * 100
-- |change_pct| > 20% → 변동 감지
```

#### 파이프라인 Lag 분석 패턴

TOF(인지도) 캠페인 → ADVoost(전환) 리타게팅 풀 공급 구조.
TOF 노출 당일~1일 후 ADVoost 장바구니에 상관관계 존재 (0.55 수준).

```
분석 절차:
1. TOF 캠페인 일별 impressions 추출
2. ADVoost 일별 purchase_conv_revenue 추출
3. Python에서 1~3일 lag 상관계수 계산
4. ADVoost CartRate(장바구니건수/클릭수×100) 추이 모니터링
   - 17~22%: 정상
   - <15%: 파이프라인 약화 경고
```

> ⚠️ 파이프라인 효과는 GFA 어트리뷰션에 직접 안 잡힘 → 간접 증거로만 판단

### ★ 한계효율(Marginal ROI) 분석

#### 개념

기존 평균 ROAS가 아닌, **추가 투입 1원당 추가 회수**를 측정.
- 한계효율 < 평균효율 → **과투자 신호**
- 한계효율 > 평균효율 → **확장 여력 있음**

```
계산 방법 (Python):
marginal_roi = (post_cart_rev - pre_cart_rev) / (post_spend - pre_spend)

예: 광고비 60K→122K 증액 시
  추가 투입 = 62K, 추가 회수 = 1.5M → 한계 ROI = 24x
  평균 ROAS = 31x → 한계 < 평균 = 과투자 구간
```

#### 소액 크림스키밍 보정

캠페인 예산을 줄이면 ROAS가 올라가는 건 **자연스러운 현상**.
광고 플랫폼이 가장 전환 확률 높은 유저에게만 노출하기 때문.

| 예산 수준 | 기대 ROAS 패턴 | 이유 |
|-----------|---------------|------|
| 소액 (원래의 30% 이하) | ROAS **2~3배↑** | 크림스키밍 (최상위 유저만) |
| 원래 예산 | 기준 ROAS | 정상 |
| 증액 (원래의 150%+) | ROAS **30~50%↓** | 한계효율 체감 |

> ⚠️ ROAS 높다고 무조건 증액하면 안 됨 → 한계효율 확인 필수
> ⚠️ 최소 7일 안정 데이터 필요 (학습기간 3~5일 제외)

### ★ GFA 어트리뷰션 ≠ 실매출 (디커플링)

GFA 장바구니 매출(`purchase_conv_revenue`)과 SmartStore 실매출(`revenue`)은 다를 수 있음.

| 상황 | GFA 어트리뷰션 | 실제 매출 | 해석 |
|------|---------------|----------|------|
| BOF 캠페인 중단 | 장바구니 **급락** | 변동 적음 | 고객이 직접 방문으로 구매 |
| TOF 캠페인 증액 | 장바구니 소폭↑ | 더 많이↑ | 인지도→직접 구매 (어트리뷰션 미반영) |
| 광고 전체 축소 | 장바구니 하락 | 덜 하락 | 오가닉/직접 유입 존재 |

```sql
-- GFA 장바구니 vs SmartStore 실매출 일별 비교
SELECT g.stat_date,
    SUM(g.purchase_conv_revenue) AS gfa_cart_rev,
    s.revenue AS store_revenue,
    ROUND(s.revenue / NULLIF(SUM(g.purchase_conv_revenue), 0), 2) AS ratio
FROM naver_ad_gfa_campaign_daily g
JOIN naver_smartstore_order_daily s
    ON g.stat_date = s.stat_date AND s.account = 'keychron'
WHERE g.stat_date >= DATE_SUB(CURDATE(), INTERVAL 14 DAY)
  AND g.account = 'keychron'
GROUP BY g.stat_date, s.revenue
ORDER BY g.stat_date;
```

> ⚠️ GFA UTM 미연동으로 GFA 인지→직접 방문 경로 추적 불가
> ⚠️ "장바구니 매출 하락 = 매출 하락"으로 해석하면 안 됨 → 반드시 SS 실매출과 대조

### ★ 키크론 스토어 기준값 (Baseline, 2026-03 기준)

"정상 vs 비정상" 판단 기준. 분석 시 참조.

#### GFA 정상 범위

| 지표 | 정상 | 경고 |
|------|------|------|
| 총 광고비/일 | 700~800K | >900K 과지출, <600K 부족 |
| TOF 합계 (인지도 캠페인) | 150~200K | <100K 파이프라인 약화 |
| ADVoost Cart ROAS | 120~150x | <100x |
| ADVoost CartRate | 17~22% | <15% |
| 전체 효율 (SS매출/총광고비) | 28~35x | <25x 비효율 |

#### SA 정상 범위

| 지표 | 정상 | 비고 |
|------|------|------|
| 일 광고비 | 70~120K | 주말 급감 (13~30K) |
| ROAS | 8,000~12,000% | = 80~120x |

#### 대장주(K10 PRO SE) 기준값

| 지표 | 정상값 | 이상치 기준 |
|------|--------|-----------|
| 주중 일매출 | 7.0~7.5M | >9M 또는 <5M |
| 토요일 | 4.5~5.5M | — |
| 일요일 | 6.0~7.0M | — |
| 스토어 비중 | 23~25% | >28% (연휴 효과 의심), <20% (이상) |

#### 달력/계절 효과

| 이벤트 | 매출 변동 | 연휴 후 첫 출근 스파이크 |
|--------|----------|----------------------|
| 설날 (5일 연휴) | **-53%** | **+41%** |
| 삼일절 (2일 연휴) | -15~20% | **+32%** |
| 토요일 | -28% (주중 대비) | — |
| 일요일 | -11% (주중 대비) | — |

> ⚠️ 연휴 후 첫 출근일 데이터는 광고 효과 분석에서 **반드시 제외** (보복소비 왜곡)
> ⚠️ 비교 시 반드시 **같은 요일끼리** 비교 (목 vs 목, 토 vs 토)

---

## 15. ★ 제품 상세 정보 데이터 구조 (AI_CHATBOT DB)

> 챗봇 제품 추천·스펙 비교·상품 안내에 사용하는 핵심 데이터
> DB: `AI_CHATBOT` (222.122.42.221:3306)
> 최종 검증: 2026-03-05

### 전체 데이터 파이프라인

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    제품 정보 데이터 구조 (AI_CHATBOT DB)                        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │  [판매 정보] naver_product (211건)                               │         │
│  │                                                                  │         │
│  │  네이버 스마트스토어 상품 기준                                    │         │
│  │  ├─ product_id (PK) : 네이버 상품번호                            │         │
│  │  ├─ product_name    : 판매 상품명 (옵션 포함)                    │         │
│  │  ├─ group_product_no: 그룹번호 (같은 모델 묶기)                  │         │
│  │  ├─ sale_price / discounted_price / discount_rate               │         │
│  │  ├─ product_attributes: 네이버 속성 (파이프 구분 KV)            │         │
│  │  ├─ supplements     : 추가상품 JSON (팜레스트, 케이블 등)       │         │
│  │  ├─ seller_tags     : 셀러 태그 (검색·추천용)                   │         │
│  │  ├─ is_sold_out     : 품절 여부                                  │         │
│  │  └─ searchable_text : 통합 검색 텍스트                           │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                           │                                                  │
│              product_name / group_name 매칭                                  │
│                           │                                                  │
│  ┌────────────────────────▼───────────────────────────────────────┐         │
│  │  [기술 스펙] products (160건) — '제품 전체 스펙'                  │         │
│  │                                                                  │         │
│  │  챗봇 제품 추천·비교의 핵심 테이블                                │         │
│  │  ├─ product_name / product_name_synonyms(JSON): 제품명+별칭     │         │
│  │  ├─ keyboard_layout / keyboard_type: 배열·타입                   │         │
│  │  ├─ switch_options(JSON): 스위치 옵션                            │         │
│  │  ├─ connection_method: 유선/무선/유무선                           │         │
│  │  ├─ hot_swap_socket / rapid_trigger: 핫스왑·래피드트리거         │         │
│  │  ├─ polling_rate: 폴링레이트                                     │         │
│  │  ├─ battery_capacity / bluetooth_runtime: 배터리·사용시간        │         │
│  │  ├─ color(JSON) / color_details(JSON): 색상 정보                │         │
│  │  ├─ features(JSON) / tags(JSON): 기능·태그                      │         │
│  │  ├─ size / weight: 크기·무게                                     │         │
│  │  ├─ discontinued: 단종 여부                                      │         │
│  │  └─ release_date: 출시일                                         │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                              │
│  ★ 두 테이블 연계:                                                          │
│  • 직접 FK 없음 → product_name 기반 매칭                                    │
│  • naver_product.group_name ≈ products.product_name (모델 단위)             │
│  • naver_product: 가격/할인/품절/태그 (판매 관점)                            │
│  • products: 스위치/배열/배터리/폴링레이트 (기술 스펙 관점)                  │
│                                                                              │
│  ★ 기존 테이블과의 관계:                                                    │
│  • naver_product.product_id = naver_smartstore_orders.product_no            │
│  • md_keychron_sheet.naver_product_code = naver_product.product_id          │
│  • erp_product_info (TBNWS_ADMIN) ↔ products: 내부 상품코드 체계           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 핵심 SQL: 제품 정보 조회

```sql
-- ★ 제품 스펙 조회 (products)
SELECT product_name, keyboard_layout, connection_method,
       switch_options, polling_rate, hot_swap_socket, rapid_trigger,
       battery_capacity, bluetooth_runtime, price, discontinued
FROM AI_CHATBOT.products
WHERE discontinued = 0;

-- ★ 네이버 상품 가격·품절 조회 (naver_product)
SELECT product_id, product_name, group_name, option_name,
       sale_price, discounted_price, discount_rate, is_sold_out
FROM AI_CHATBOT.naver_product
WHERE is_sold_out = 0;

-- ★ 모델별 옵션 목록 (group_product_no로 그룹핑)
SELECT group_name, option_name, discounted_price, is_sold_out
FROM AI_CHATBOT.naver_product
WHERE group_product_no = %s
ORDER BY discounted_price;

-- ★ 추가상품(악세서리) 조회 — supplements JSON 파싱
-- supplements 예시: [{"name":"원목 팜레스트 | 호두나무 (PR3)","price":29000,"isSoldOut":false}, ...]
SELECT product_id, product_name, supplements
FROM AI_CHATBOT.naver_product
WHERE supplements IS NOT NULL AND supplements != '[]';

-- ★ 통합 검색 (searchable_text FULLTEXT 검색)
SELECT product_id, product_name, discounted_price, is_sold_out
FROM AI_CHATBOT.naver_product
WHERE searchable_text LIKE '%저소음%' AND searchable_text LIKE '%무선%'
  AND is_sold_out = 0;

-- ★ naver_product ↔ naver_smartstore_orders 연결 (판매실적 포함)
SELECT np.group_name, np.option_name, np.discounted_price, np.is_sold_out,
       COUNT(o.product_order_id) AS order_cnt,
       SUM(o.total_price) AS revenue
FROM AI_CHATBOT.naver_product np
LEFT JOIN TBNWS_ADMIN.naver_smartstore_orders o
    ON np.product_id = o.product_no
    AND o.product_order_status NOT IN ('CANCELED','RETURNED','EXCHANGED')
    AND o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
WHERE np.is_sold_out = 0
GROUP BY np.product_id
ORDER BY revenue DESC;
```

### AI 질문-제품 데이터 매핑

| 질문 유형 | 데이터 소스 | 핵심 필드/로직 |
|---|---|---|
| "이 제품 스펙 알려줘" | products | product_name 매칭 → 전체 스펙 반환 |
| "무선 키보드 추천해줘" | products | `connection_method LIKE '%무선%'` + features/tags |
| "핫스왑 되는 모델?" | products | `hot_swap_socket IS NOT NULL` |
| "래피드 트리거 지원?" | products | `rapid_trigger` 필드 확인 |
| "가격 얼마야?" | naver_product | `discounted_price` (할인가 우선) |
| "품절이야?" | naver_product | `is_sold_out = 1` |
| "이 모델 옵션 뭐있어?" | naver_product | `WHERE group_product_no = X` 옵션 나열 |
| "팜레스트 호환?" | naver_product | `supplements` JSON 파싱 |
| "저소음 축 있는 모델?" | naver_product + products | searchable_text 또는 switch_options |
| "단종 제품?" | products | `WHERE discontinued = 1` |
| "최근 출시 제품?" | products | `ORDER BY release_date DESC` |

### ⚠️ 핵심 주의사항

```
1. naver_product vs products 역할 구분
   • 가격/품절/할인 → naver_product (네이버 기준 실시간)
   • 기술 스펙/스위치/배터리 → products (상세 스펙)
   • 두 테이블 모두 조회해야 완전한 제품 정보 제공 가능

2. naver_product.product_attributes 파싱
   • 파이프(|) 구분 Key-Value 형태
   • 예: "연결방식: 유선 | 단자: USB Type-C | 키방식: 기계식"
   • 콜론(:) 앞 = 속성명, 뒤 = 값

3. naver_product.supplements JSON 구조
   • 배열 형태: [{"name":"...", "price":29000, "isSoldOut":false}, ...]
   • 추가상품(악세서리) 목록 — 팜레스트, 더스트커버, 케이블 등

4. products.product_name_synonyms 활용
   • 사용자가 비공식 명칭으로 질문 시 동의어 매칭에 활용
   • JSON 배열 형태

5. 크로스 DB 조인 시
   • naver_product는 AI_CHATBOT DB
   • naver_smartstore_orders는 TBNWS_ADMIN DB
   • product_id(AI_CHATBOT) = product_no(TBNWS_ADMIN) 매핑
```
