# ToBe Networks AI 질문-데이터 인덱스 맵

> 각 역할별 질문에 정확한 데이터를 뽑기 위한 완전한 매핑 문서
> 최종 업데이트: 2026-02-28

---

## 채널 분류 기준 (ON/OFF)

### ON 채널 (온라인 직접 판매) - `tbnws_sabangnet_order`의 온라인 컬럼
| 거래처명 | 유형 |
|----------|------|
| 브랜드스토어 | 자사몰 |
| Cafe24(신) 유튜브쇼핑 | 자사몰 |
| 스마트스토어 | 네이버 |
| 스마트스토어 (AIPER) | 네이버 |
| 쿠팡 로켓 | 쿠팡 직매입 |
| 쿠팡 | 쿠팡 마켓 |
| ESM지마켓/옥션 | G마켓/옥션 |
| 쇼피파이 | 해외몰 |
| 11번가 | 오픈마켓 |
| 롯데온/무신사/29CM/카카오 등 | 온라인 채널 |

### OFF 채널 (B2B/오프라인) - `erp_order_recv`의 오프라인 컬럼
| 거래처명 | 유형 |
|----------|------|
| 쿠팡 로켓 (지티) | 쿠팡 도매 |
| (주)밸류포인트 | 도매 |
| (주) 글로와이드 | B2B |
| (주) 영풍문고 | 리테일 |
| 하나커머즈 | B2B |
| 주식회사 이오샵 | B2B |
| (주)에스와이에스리테일 | B2B |
| 아산사회복지재단 | 기관 |
| 인스토어 | 오프라인 |
| 퍼스트몰 (b2b.gtgear.co.kr) | B2B몰 |
| 학교/기관/기업 | 대량구매 |

---

## 상품 식별 체계 (Product Identification)

### 상품코드(G-code) 구조

```sql
-- erp-product.xml:17 - product_code 생성 공식
CONCAT(category_code, '-', brand_code, '-', goods_code, '-', option_code)
-- 예: G-0029-0509-0001

-- erp-product.xml:19 - product_name 표시 공식
CONCAT('[', brand_name, '] ', goods_name, ' - ', option_name)
-- 예: [Keychron] 키크론 EX - (EX75-A6) EX75 HE 블랙 8K RGB 기계식 핫스왑 저소음 마그네틱스위치
```

```
G-code 구조: {category}-{brand}-{goods}-{option}
├── category_code: G(일반상품), F(풀필먼트)
├── brand_code: 0029(키크론), 0027(지티기어), ...
├── goods_code: 상품군 식별 (4자리)
└── option_code: 옵션 식별 (4자리, 색상/스위치 등)
```

### 이름 매핑 패턴 (사용자 호칭 → DB → 네이버)

```
[사용자가 부르는 이름]
"K10 PRO SE 핫스왑 미지원 레트로파스텔 저소음 바나나"
"EX75"

[erp_product_info 필드 매핑]
├── brand_name = "Keychron" (또는 "키크론")
├── goods_name = "키크론 K PRO SE 핫스왑 미지원 레트로파스텔" (상품군명)
├── option_name = "저소음 바나나" (옵션명)
├── product_code = "G-0029-XXXX-XXXX" (G-code)
└── goods_code = "XXXX" (상품군 코드)

[erp_customer_order_product_list 필드]
├── model_name = "키크론 K PRO SE" (모델 시리즈명, 판매분석용)
├── product_name = 전체 상품명 (주문 데이터 원본)
└── layout_name = 레이아웃 (풀사이즈, TKL, 75% 등)

[naver_commerce_product_detail 필드]
├── seller_product_code = "G-0029-XXXX-XXXX" (= G-code 전체)
├── product_name = 네이버에 등록된 상품명
└── option_code = 네이버 옵션 코드

[네이버 ↔ 내부 매핑 테이블]
md_keychron_sheet: naver_product_code → tobe_product_code (G-code)
md_keychron_sheet_attach: option_code → tobe_code (복합 옵션)
```

### 상품 검색 SQL

```sql
-- 사용자 호칭에서 상품 찾기
-- "K10 PRO SE 레트로파스텔 저소음 바나나" → product_code 찾기
SELECT product_code, brand_name, goods_name, option_name
FROM erp_product_info
WHERE goods_name LIKE '%K PRO SE%레트로파스텔%'
  AND option_name LIKE '%저소음%바나나%'
  AND sell_status = 'Y';

-- "EX75" → product_code 찾기
SELECT product_code, brand_name, goods_name, option_name
FROM erp_product_info
WHERE (goods_name LIKE '%EX75%' OR option_name LIKE '%EX75%')
  AND sell_status = 'Y';

-- G-code로 네이버 매핑 확인
SELECT tobe_product_code, naver_product_code, internal_management_code
FROM md_keychron_sheet
WHERE tobe_product_code = 'G-0029-0509-0001';
```

---

## 상품별 원가/판가 조회 패턴

### AI가 "원가/판가 알려줘" 질문에 답하는 방법

```
[Step 1] 상품 식별
상품명/모델명 → erp_product_info에서 goods_name, option_name LIKE 검색
→ product_code (예: G-0029-0509-0001), goods_code (예: 0509)

[Step 2] 원가 조회 (2가지 경로)
경로A: selectGoodsPriceList (erp-product.xml:547)
  erp_product_info → LATERAL JOIN erp_stock → erp_order_item
  최신 입고건 기준 (ORDER BY import_date DESC LIMIT 1)
  → buyingPrice (외화원가), buyingKRWPrice (원화환산원가)

경로B: selectOrderPredictionList (dev.xml:9)
  erp_order → erp_order_item → erp_product_info
  최근 1년 발주 기준 (ORDER BY order_date DESC, ROW_NUMBER = 1)
  → recentBuyingPriceRaw (외화원가), recentBuyingPrice (평균 환산원가)
  → recentBuyingCurrency (통화)

[Step 3] 판매가 조회 (4단계 정밀도 - 수정됨)
★ 1순위: naver_smartstore_orders.total_price/quantity → 실거래가 (현재 운영)
  2순위: naver_commerce_order.totalPaymentAmount/quantity → 실거래가 (상세, 동기화 중단)
  3순위: erp_discounted_products.price → 할인 설정가 (관리자 설정, ≠ 실거래가)
  4순위: erp_product_info.actual_price → 관리자 설정 기준가 (대시보드 "현재 판매가")
⚠️ 3순위/4순위는 관리자 설정값이며 실제 판매가가 아님!
⚠️ naver_commerce_product_detail 테이블은 존재하지 않음 (이전 문서 오류)

[Step 4] 마진 계산
매출총이익 = 1순위 실거래가 - (buyingPrice × cur2krw × 1.1 VAT)
정산마진 = expectedSettlementAmount - (buyingPrice × cur2krw × 1.1 VAT)  ※ naver_commerce_order 한정
```

### 핵심 가격 조회 SQL (selectGoodsPriceList)

```sql
-- erp-product.xml:547 - 상품별 원가/판가 전체 조회
SELECT
  product.product_code AS code,
  product.brand_name, product.goods_name, product.option_name,
  product.msrp,                              -- 권장소비자가 (정가)
  product.purchase_price AS purchasePrice,   -- 사입가 (관리자 설정)
  product.actual_price AS actualPrice,       -- 현재 판매가 (관리자 설정, ≠ 실거래가)
  product.actual_price_channel,              -- 채널 TAG가
  product.duty,                              -- 관세
  product.currency,                          -- 통화
  -- 최신 수입원가 (가장 최근 입고건 기준)
  stock_info.buyingPrice,                    -- 외화 원가 (예: $35)
  stock_info.buyingKRWPrice,                 -- 원화 환산 원가 (예: 56,753원)
  -- 할인가 (있는 경우)
  discount_info.discountedPrice
FROM erp_product_info product
LEFT JOIN LATERAL (
    SELECT o_i.buying_price AS buyingPrice,
           o_i.buying_price * o_i.cur2krw AS buyingKRWPrice
    FROM erp_stock stock
    JOIN erp_order_item o_i ON o_i.item_seq = stock.item_seq
    JOIN erp_order o ON o_i.order_seq = o.order_seq
    WHERE stock.product_code = product.product_code
      AND stock.import_date IS NOT NULL
      AND o_i.buying_price > 0
    ORDER BY stock.import_date DESC LIMIT 1
) stock_info ON true
LEFT JOIN LATERAL (
    SELECT price AS discountedPrice
    FROM erp_discounted_products dp
    WHERE dp.product_code = product.product_code
    ORDER BY seq DESC LIMIT 1
) discount_info ON true
WHERE product.goods_code = #{goodsCode}  -- 상품 필터
```

### 발주예측 대시보드 데이터 (selectOrderPredictionList)

```sql
-- dev.xml:9 - 상품별 재고/판매/원가 통합 조회
-- 반환 필드:
--   productCode, brandName, goodsName, optionName
--   currentStock, incomingStock (재고)
--   oneWeekSales, oneMonthSales, threeMonthSales (판매량)
--   b2cXxxSales, b2bXxxSales, rocketXxxSales (채널별 판매)
--   actualPrice, msrp (가격)
--   recentBuyingPrice (환산원가), recentBuyingPriceRaw (외화원가)
--   recentBuyingCurrency (통화)
--   leadTimeDays (리드타임)
--   lastOrderDate, lastOrderQty, lastOrderPrice (최근 발주)
--   lastImportDate, lastImportQty (최근 입고)

-- 원가 계산 로직 (dev.xml:161-175):
-- recentBuyingPrice = AVG(
--   CASE WHEN currency='KRW' THEN buying_price
--        ELSE buying_price × cur2krw END
-- ) → 입고 완료된 건들의 평균 환산 원가
-- recentBuyingPriceRaw = 최근 발주의 외화 원가 (ROW_NUMBER=1)

-- 환산원가(VAT포함) = buyingKRWPrice × 1.1
-- ⚠️ VAT 10%는 DB에 미반영, AI 계산 시 별도 적용 필요
```

### 실거래가 조회 (2개 소스)

```sql
-- ⚠️ naver_commerce_product_detail 테이블은 존재하지 않음! (이전 문서 오류)
-- 실제 네이버 실거래가 테이블은 아래 2개:

-- [1순위] naver_smartstore_orders (현재 운영, 전체 기간 커버)
SELECT
  product_name,
  COUNT(*) AS order_count,
  ROUND(AVG(total_price/quantity)) AS avg_price,
  MIN(total_price/quantity) AS min_price,
  MAX(total_price/quantity) AS max_price
FROM naver_smartstore_orders
WHERE product_name LIKE '%검색어%'  -- 예: '%eX75%', '%K10 PRO SE%'
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
  AND quantity > 0 AND total_price > 0
GROUP BY product_name
ORDER BY order_count DESC;

-- [2순위] naver_commerce_order (상세 필드, 2026-01-02 이후 동기화 중단)
SELECT
  sellerProductCode AS goods_code,
  COUNT(*) AS order_count,
  AVG(unitPrice) AS avg_unit_price,
  AVG(totalPaymentAmount / quantity) AS avg_paid_price,
  AVG(expectedSettlementAmount / quantity) AS avg_settle
FROM naver_commerce_order
WHERE sellerProductCode = 'G-0029-0509-0001'  -- G-code로 검색
  AND productOrderStatus IN ('DELIVERING','PURCHASE_DECIDED','DELIVERED','PAYED')
GROUP BY sellerProductCode;
```

---

## 영업이익 계산 공식 (Margin Calculation)

> 전체 데이터 구조, 테이블 스키마, 조인 키 매핑: [05_DATA_STRUCTURE.md](./05_DATA_STRUCTURE.md) 참조

### 판매 데이터 소스 (정확도순)

```
1. naver_smartstore_orders (네이버 API 자동동기화, 현재 운영) ← 실거래가 1순위
   - 필드: product_name, total_price, quantity, product_order_status, order_date, account
   - 식별: product_name LIKE 검색
   - 기간: 2023-12-21 ~ 현재

2. naver_commerce_order (네이버 API, 동기화 중단 2026-01-02) ← 상세 데이터 필요시
   - 필드: sellerProductCode, unitPrice, totalPaymentAmount, expectedSettlementAmount 등
   - 식별: sellerProductCode = G-code
   - 기간: 2025-01-13 ~ 2026-01-02

3. tbnws_sabangnet_order (사방넷 API 자동동기화) ← 전 채널 통합, 실결제가

4. erp_customer_order_list (수동 입력) ← 판매분석 대시보드용, 모델 단위 집계

⚠️ naver_commerce_product_detail 테이블은 존재하지 않음! (이전 문서 오류)
⚠️ 판매분석 페이지(sales-analysis-prototype.html)는 Layer 4 데이터 기반
⚠️ 실거래가 조회는 naver_smartstore_orders.total_price/quantity 사용
```

### 가격 체계 이해 (Price Hierarchy)

```
erp_product_info 가격 필드 (상품 마스터):
┌────────────────────┬──────────────────────────────────────────────┐
│ msrp               │ 권장소비자가 (제조사 정가, 고정)               │
│ actual_price       │ 실판가 (관리자 설정 기준 판매가)              │
│ fixed_price        │ 정가                                         │
│ purchase_price     │ 사입가                                       │
│ channel_price      │ 채널 상시가                                   │
│ actual_price_channel│ 채널 TAG가                                   │
│ actual_price_channel_uptag│ 채널 업택가                            │
│ channel_uptag_price│ 채널 업택 상시가                              │
└────────────────────┴──────────────────────────────────────────────┘

⚠️ 대시보드 "현재 판매가" = erp_product_info.actual_price
   이것은 관리자가 설정한 기준가이며, 네이버 실 거래가와 다름!
```

### 실제 거래가 데이터 소스

```
[수입원가 (COGS) — 검증 완료]
경로: erp_stock.product_code → erp_stock.item_seq → erp_order_item.item_seq
  erp_stock.buying_price     → KRW 환산 원가 (이미 환산됨)
  erp_order_item.cur2krw     → 적용 환율 (USD→KRW)
  USD 원가 = buying_price / cur2krw
→ 원가(VAT포함) = erp_stock.buying_price × 1.1
→ 예시: buying_price=56,753원, cur2krw=1,621.5 → USD $35 → VAT포함 62,428원
⚠️ cur2krw > 100 필터 필수 (비정상 환율 데이터 존재)
⚠️ ORDER BY import_date DESC LIMIT 1 → 최신 입고건 기준

[네이버 실거래가 - 1순위 (현재 운영)]
naver_smartstore_orders 테이블 (55,398건, 2023-12~현재):
  total_price / quantity → 실결제 단가
  product_no → md_keychron_sheet.naver_product_code → G-code 매핑
  ⚠️ COLLATE utf8mb4_unicode_ci 필수 (collation mismatch)

[네이버 정산 상세 - 2순위 (동기화 중단)]
naver_commerce_order 테이블 (19,332건, 2025-01~2026-01):
  sellerProductCode → G-code 직접 매핑 (md_keychron_sheet 불필요)
  unitPrice, totalPaymentAmount, expectedSettlementAmount 등 56필드
  ⚠️ 2026-01-02 이후 데이터 없음!

[사방넷 실거래가 - 전 채널 통합]
tbnws_sabangnet_order.pay_cost / p_ea → 건당 실결제가
→ 15개+ 채널 (지마켓, 29CM, 쿠팡, 11번가 등)
→ mall_id로 채널 구분

[B2B 출고가]
erp_order_recv_item.export_price → B2B 거래처별 출고가

[채널별 설정가 (관리자 설정, ≠ 실거래가)]
erp_channel_price: (product_code, partner_code, channel_code) → channel_price
erp_discounted_products: product_code → price (할인 설정가)
⚠️ 이것들은 관리자가 설정한 가격이며 실제 판매가와 다를 수 있음!
```

### 영업이익 계산 공식

```
[ON채널 (B2C) - 네이버 실거래 기준]
매출 = naver_smartstore_orders.total_price (실결제금액)
원가 = erp_stock.buying_price × 1.1 (VAT포함)
매출총이익 = 매출 - (수량 × 원가VAT)
마진율 = (매출 - 원가) / 매출 × 100

[ON채널 (B2C) - 정산가 기준 (naver_commerce_order 한정)]
정산액 = naver_commerce_order.expectedSettlementAmount
영업이익 = 정산액 - (수량 × 원가VAT) - 물류비
→ 수수료가 이미 차감된 금액이므로 더 정확

[ON채널 (B2C) - 사방넷 전 채널]
실수취액 = tbnws_sabangnet_order.pay_cost
매출총이익 = 실수취액 - (수량 × 원가VAT)

[OFF채널 (B2B)]
매출총이익 = 출고가(export_price) - 원가VAT
영업이익 = 매출총이익 - 물류비

[채널별 수수료율 참고]
네이버 스마트스토어: ~5.5%
쿠팡 로켓그로스: ~10%
쿠팡 로켓배송(직매입): 마진 협의
11번가/G마켓: ~12%
자사몰(브랜드스토어): ~3%
```

### 실전 마진 분석 SQL (검증 완료)

> 상세 데이터 구조 및 테이블 스키마: [05_DATA_STRUCTURE.md](./05_DATA_STRUCTURE.md) 참조

```sql
-- ★ 방법 1: naver_smartstore_orders + md_keychron_sheet + 원가 (현재 운영, 검증 완료)
-- 3-way JOIN: 실거래가 + G-code 매핑 + 수입원가

-- Step 1: 대표 G-code 매핑 (1:N 중복 제거)
WITH gcode_map AS (
    SELECT naver_product_code, MIN(tobe_product_code) AS gcode
    FROM md_keychron_sheet
    WHERE naver_product_code IS NOT NULL AND naver_product_code != ''
      AND tobe_product_code IS NOT NULL AND tobe_product_code != ''
    GROUP BY naver_product_code
)

-- Step 2: 실거래가 + G-code 결합
SELECT
    nso.product_name,
    gm.gcode AS g_code,
    COUNT(*) AS order_count,
    SUM(nso.quantity) AS total_qty,
    SUM(nso.total_price) AS total_revenue,
    ROUND(AVG(nso.total_price / nso.quantity)) AS avg_price,
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

-- Step 3: G-code별 원가 별도 배치 조회 (성능 최적화)
SELECT s.product_code, s.buying_price AS buying_krw,
       s.buying_price / oi.cur2krw AS buying_usd,
       oi.cur2krw, s.import_date
FROM erp_stock s
JOIN erp_order_item oi ON s.item_seq = oi.item_seq
WHERE s.product_code IN (/* G-code 목록 */)
  AND s.buying_price > 0 AND oi.cur2krw > 100
ORDER BY s.product_code, s.import_date DESC;
-- → Python에서 product_code별 첫 번째 행 = 최신 원가
```

```sql
-- ★ 방법 2: naver_commerce_order 기반 (정산 상세, 2026-01-02까지만)
-- sellerProductCode = G-code 직접 매핑 (md_keychron_sheet 불필요)
SELECT
  pi.product_name, pi.goods_code,
  pi.msrp,
  pi.actual_price as admin_set_price,
  COUNT(*) as order_count,
  SUM(nc.quantity) as total_qty,
  ROUND(AVG(nc.unitPrice)) as naver_list_price,
  ROUND(AVG(nc.totalPaymentAmount / nc.quantity)) as naver_paid_price,
  ROUND(AVG(nc.expectedSettlementAmount / nc.quantity)) as settle_price,
  -- 수입원가 (LATERAL JOIN)
  lat.buying_krw,
  ROUND(lat.buying_krw * 1.1) as cost_vat,
  -- 마진 (정산가 기준)
  ROUND(AVG(nc.expectedSettlementAmount / nc.quantity) - lat.buying_krw * 1.1) as margin_per_unit,
  ROUND((AVG(nc.expectedSettlementAmount / nc.quantity) - lat.buying_krw * 1.1)
    / NULLIF(AVG(nc.expectedSettlementAmount / nc.quantity), 0) * 100, 1) as margin_pct
FROM erp_product_info pi
JOIN naver_commerce_order nc ON pi.goods_code = nc.sellerProductCode
  AND nc.productOrderStatus IN ('DELIVERING', 'PURCHASE_DECIDED', 'DELIVERED', 'PAYED')
LEFT JOIN LATERAL (
    SELECT s.buying_price AS buying_krw
    FROM erp_stock s
    JOIN erp_order_item oi ON s.item_seq = oi.item_seq
    WHERE s.product_code = pi.product_code
      AND s.buying_price > 0 AND oi.cur2krw > 100
    ORDER BY s.import_date DESC LIMIT 1
) lat ON TRUE
WHERE pi.goods_code = 'G-0029-0509-0001'  -- 예: 키크론 EX75
GROUP BY pi.goods_code;
```

```sql
-- 방법 3: 사방넷 전 채널 통합 기반
SELECT
  pi.product_name, pi.product_code,
  s.mall_id as channel,
  ROUND(AVG(s.pay_cost / s.p_ea)) as real_selling_price,
  SUM(s.p_ea) as total_qty,
  SUM(s.pay_cost) as total_revenue
FROM tbnws_sabangnet_order s
JOIN erp_product_info pi ON s.model_no = pi.product_code
WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND s.order_status NOT IN ('취소')
GROUP BY pi.product_code, s.mall_id
ORDER BY total_revenue DESC;
```

```
[확정 데이터: 키크론 EX75 (G-0029-0509-0001)] (DB 실 조회 2026-02-28)
원가(외화): $35 USD
원가(원화환산): buyingKRWPrice = 50,470원 (cur2krw = 1,442, 입고일 2025-12-23)
원가(VAT포함): 55,517원 (50,470 × 1.1)
관리자설정가(actual_price): 119,000원
할인설정가(discountedPrice): 89,000원
네이버 실거래가(naver_smartstore_orders): 평균 67,500~74,500원 (434건, 2026-01~02)
→ 네이버 상품명: "키크론 익스트림 eX75 8K TMR 자석축 키보드 저소음 래피드 트리거 발로란트 키보드"
→ 마진(실거래 기준): 67,500 - 55,517 = 11,983원 (17.8%)
→ 주의: naver_commerce_order에는 0건 (동기화 2026-01-02 중단, EX75 첫판매 2026-01-08)

[확정 데이터: 키크론 K10 Pro SE 레트로파스텔 저소음바나나 핫스왑 미지원 (G-0029-0405-0044)] (DB 실 조회 2026-02-28)
원가(외화): $44 USD
원가(원화환산): buyingKRWPrice = 63,932원 (cur2krw = 1,453, 입고일 2026-02-10)
원가(VAT포함): 70,325원 (63,932 × 1.1)
관리자설정가(actual_price): 179,000원
관리자할인가(discountedPrice): 129,000원
네이버 실거래가(naver_smartstore_orders): 평균 123,534~125,960원
→ 네이버 상품명: "키크론 K10 PRO SE 레트로 무선 기계식 키보드 사무용 저소음" (3,779건)
→ 네이버 상품명(구): "키크론 K10 PRO SE 레트로 무선 기계식 키보드 핫스왑 미지원 저소음 바나나축" (2,895건)
→ naver_commerce_order에도 108건 있음 (sellerProductCode = G-0029-0405-0044)
→ 마진(실거래 기준): 123,534 - 70,325 = 53,209원 (43.1%)

참고: 동일 모델 핫스왑 지원 버전 (G-0029-0405-0047)
actual_price: 189,000원, discountedPrice: 149,000원, purchase_price: $46 USD
```

### 네이버 실거래가 조회 - 2개 테이블 주의
```
⚠️ 핵심: 네이버 판매 데이터가 2개 테이블에 분산되어 있음

[1] naver_commerce_order (상세, 중단됨)
- 기간: 2025-01-13 ~ 2026-01-02 (동기화 중단!)
- 건수: 19,332건
- 식별: sellerProductCode = G코드 (예: G-0029-0405-0044)
- 장점: unitPrice, totalPaymentAmount, expectedSettlementAmount, 할인내역 등 상세 필드
- 단점: 2026-01-02 이후 신상품 데이터 없음 (EX75 등)

[2] naver_smartstore_orders (간소, 현재 운영)
- 기간: 2023-12-21 ~ 현재 (2026-02-27)
- 건수: 55,398건
- 식별: product_name = 네이버 상품명 (LIKE 검색 필요)
- 장점: 최신 데이터, 전체 기간 커버
- 단점: total_price 하나만 있음, 정산액/할인내역 없음

→ AI 조회 로직:
  1순위: naver_smartstore_orders (전체 기간, product_name LIKE 검색)
  2순위: naver_commerce_order (상세 데이터 필요시, sellerProductCode로 검색)
  fallback: erp_discounted_products (관리자 설정 할인가)
  최종: erp_product_info.actual_price (관리자 설정 정가)
```

### DB 직접 조회 접근 방법
```
Host: 222.122.42.221:3306 (MySQL)
Database: TBNWS_ADMIN
접근: pymysql (python3) - mysql_native_password 호환
프로파일: dev 환경 사용
```

### 키크론 전 모델 네이버 실거래가 (naver_smartstore_orders 기준, 2026-02-28 조회)

```
⚠️ 아래 데이터는 naver_smartstore_orders 테이블에서 조회
⚠️ 액세서리(키스킨/키캡) 제외, 취소/반품/교환 제외
⚠️ 이전 세션의 모델 단위 분류에서 서브모델 단위로 세분화 (CV% 대폭 개선)

[가격 레인지 세분화 원칙]
- K10 PRO 계열 → K10 PRO MAX / K10 PRO SE2 / K10 PRO ZMK / K10 PRO SE 4분류
- K8 PRO 계열 → K8 PRO MAX / K8 PRO SE / K8 PRO(구) 3분류
- Q0 계열 → Q10+Q0 콤보세트 / Q0 PRO MAX 단품 / Q0 MAX 분리
- 레모키 → L1 / X3 별도 분류
- 액세서리(키스킨 3,000~5,900원)는 키보드와 분리

[서브모델별 평균 실거래가]
서브모델              거래건수   평균 실거래가   가격레인지            CV%
─────────────────────────────────────────────────────────────────────
Q10+Q0 콤보세트         18      406,799원    397,000~429,000원    3.3
Q1 HE                  15      299,314원    268,770~309,000원    4.8
Q5 PRO                  8      296,250원    259,480~309,000원    4.9
Q3 PRO                 64      281,530원    246,450~285,000원    3.0
Q1 PRO                 13      281,040원    259,470~285,000원    2.9
K2 HE                  48      185,770원    163,590~189,000원    3.7
K3 MAX                  8      168,054원    163,930~169,000원    0.9
K2 PRO                 14      165,586원    154,300~195,300원    8.9
Q0 PRO MAX            171      160,248원    128,750~179,000원    7.7
K3 PRO                 17      158,066원    148,010~165,800원    4.3
K1 PRO                 30      157,448원    145,310~165,800원    3.9
K10 PRO MAX          3,272      151,979원    104,360~164,000원    4.0
K8 PRO MAX             843      151,773원    104,860~159,800원    4.2
Q0 MAX                 10      148,872원    148,810~149,000원    0.1
K10 PRO ZMK           313      139,232원    123,310~194,900원    4.2
K10 PRO SE2          1,546      138,411원     96,030~149,000원    5.9
K8 PRO SE             155      127,377원     97,500~139,000원    8.0
K10 PRO SE           5,771      120,998원     83,300~149,000원    9.0
K8 PRO (구)             56       95,393원     66,930~139,000원   18.8
C2 PRO                832       81,512원     67,300~89,000원     5.9
EX75                  366       71,989원     62,600~82,000원     5.1
C1 PRO                 29       70,010원     44,100~79,000원    12.8
레모키 X3              145       51,705원     39,320~74,900원    13.9
B6 PRO               2,316       43,190원     33,740~47,500원    9.8
B1 PRO               2,322       40,226원     31,340~42,500원    5.5
레모키 L1                3      ~182,000원   181,890~255,000원   (건수부족)

[세분화 전후 CV% 비교 - 주요 개선 사례]
모델           세분화 전 CV%  →  세분화 후 CV%   원인
EX75              183.6%     →    5.1%          비키크론 제품(파나텍/로지텍) 혼입 제거
레모키              42.8%     →   13.9%/건수부족  L1(182K)과 X3(52K) 완전 다른 모델 분리
Q0 PRO MAX         40.0%     →    7.7%/3.3%     콤보세트(407K)와 단품(160K) 분리
C2 PRO             29.6%     →    5.9%          키스킨(4,500원) 제거
K10 PRO SE         26.3%     →    4.0~9.0%      MAX/SE2/ZMK/SE 4개 서브모델 분리
K8 PRO             25.4%     →    4.2~18.8%     MAX/SE/구버전 3개 서브모델 분리
B1 PRO             26.9%     →    5.5%          키스킨(3,500원) 제거
B6 PRO             24.2%     →    9.8%          키스킨(3,500원) + 한컴패키지(113K) 제거

[B6 PRO 특이사항]
- max 114,900원 = "[라운지전용] 키크론 B6 PRO 한글과컴퓨터 오피스 패키지" (4건)
- 실질 키보드 max: 47,500원

[K8 PRO 구버전 특이사항]
- CV 18.8%로 아직 높음 (56건 소량)
- 구성: K8 PRO 화이트 핫스왑(39건, 98K) + 타임딜(7건, 69K) + 라운지전용 알루미늄(5건, 137K)
- 단종 과도기 재고 처리로 가격 편차 큰 것으로 추정
```

### AI 실거래가 조회 SQL (서브모델 분류 포함)

```sql
-- naver_smartstore_orders에서 서브모델별 실거래가 조회
-- product_name 기반 CASE WHEN 분류 사용
SELECT
    CASE
        WHEN product_name LIKE '%eX75%' OR (product_name LIKE '%EX75%' AND product_name LIKE '%키크론%') THEN 'EX75'
        WHEN product_name LIKE '%K10 PRO MAX%' OR product_name LIKE '%K10 Pro Max%' THEN 'K10 PRO MAX'
        WHEN product_name LIKE '%K10 PRO SE2%' OR product_name LIKE '%K10 Pro SE2%' THEN 'K10 PRO SE2'
        WHEN product_name LIKE '%K10 PRO ZMK%' OR product_name LIKE '%K10 Pro ZMK%' THEN 'K10 PRO ZMK'
        WHEN product_name LIKE '%K10 PRO SE%' OR product_name LIKE '%K10 Pro SE%' THEN 'K10 PRO SE'
        WHEN product_name LIKE '%K8 PRO MAX%' OR product_name LIKE '%K8 Pro Max%' THEN 'K8 PRO MAX'
        WHEN product_name LIKE '%K8 PRO SE%' OR product_name LIKE '%K8 Pro SE%' THEN 'K8 PRO SE'
        WHEN product_name LIKE '%K8 PRO%' OR product_name LIKE '%K8 Pro%' THEN 'K8 PRO'
        WHEN product_name LIKE '%Q10%Q0 PRO MAX%' THEN 'Q10+Q0 콤보세트'
        WHEN product_name LIKE '%Q0 PRO MAX%' OR product_name LIKE '%Q0 Pro Max%' THEN 'Q0 PRO MAX'
        WHEN product_name LIKE '%Q0 Max%' THEN 'Q0 MAX'
        WHEN product_name LIKE '%레모키 L1%' OR product_name LIKE '%Lemokey L1%' THEN '레모키 L1'
        WHEN product_name LIKE '%레모키 X3%' OR product_name LIKE '%Lemokey X3%' THEN '레모키 X3'
        -- ... 기타 모델 추가 가능
        ELSE product_name
    END AS sub_model,
    COUNT(*) AS cnt,
    ROUND(AVG(total_price/quantity)) AS avg_price,
    MIN(total_price/quantity) AS min_price,
    MAX(total_price/quantity) AS max_price
FROM naver_smartstore_orders
WHERE product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
  AND quantity > 0 AND total_price > 0
  AND product_name NOT LIKE '%키스킨%'
  AND product_name NOT LIKE '%키캡%'
  AND product_name NOT LIKE '%실리콘 키스킨%'
GROUP BY sub_model
HAVING cnt >= 5
ORDER BY avg_price DESC;
```

---

## 브랜드 체계

| 브랜드 | category_code | brand_code 패턴 | 상품코드 예시 |
|--------|---------------|-----------------|---------------|
| 키크론 (Keychron) | G | 0029 | G-0029-XXXX-XXXX |
| 지티기어 (GTGear) | G | 0027 | G-0027-XXXX-XXXX |
| MOZA | G | 03XX | G-03XX-XXXX-XXXX |
| 캐논 | G | 02XX | G-02XX-XXXX-XXXX |
| 파나텍 | G | 04XX | G-04XX-XXXX-XXXX |
| AIPER | G | 01XX | G-01XX-XXXX-XXXX |
| 플레이시트/트랙레이서/STRIX | G | - | 레이싱 시뮬레이터 |
| 풀필먼트 상품 | F | - | F-XXXX-XXXX-XXXX |

---

## 3. 영업 담당 (국내 B2B)

### 🔎 매출 흐름

#### Q: "매출이 급격히 오른 모델 있어?"
```sql
-- 데이터 소스: erp_annual_model_sales + tbnws_sabangnet_order
-- 전월 대비 모델별 매출 증감률 계산
SELECT
  p.goods_name, p.product_code,
  SUM(CASE WHEN s.order_date BETWEEN '2026-02-01' AND '2026-02-28' THEN s.pay_cost END) as this_month,
  SUM(CASE WHEN s.order_date BETWEEN '2026-01-01' AND '2026-01-31' THEN s.pay_cost END) as last_month
FROM tbnws_sabangnet_order s
JOIN erp_product_info p ON s.model_no = p.product_code
WHERE s.order_status NOT IN ('취소')
GROUP BY p.goods_name
HAVING this_month > last_month * 1.3  -- 30% 이상 증가
ORDER BY (this_month - last_month) DESC
```
**관련 테이블**: `tbnws_sabangnet_order`, `erp_product_info`, `erp_annual_model_sales`
**키 컬럼**: `pay_cost`(매출), `order_date`(주문일), `model_no`(상품코드)

#### Q: "매출 급락한 제품/거래처는?"
```sql
-- 데이터 소스: tbnws_sabangnet_order (파트너별 매출 비교)
-- sales-dashboard-prototype.html의 "이상 감지 & 액션" 영역이 이 로직
SELECT
  mall_id as partner,
  SUM(CASE WHEN order_date BETWEEN '2026-02-01' AND '2026-02-28' THEN pay_cost END) as feb_sales,
  SUM(CASE WHEN order_date BETWEEN '2026-01-01' AND '2026-01-31' THEN pay_cost END) as jan_sales,
  ROUND((feb - jan) / jan * 100, 1) as change_pct
FROM tbnws_sabangnet_order
WHERE order_status NOT IN ('취소')
GROUP BY mall_id
HAVING change_pct < -20  -- 20% 이상 하락
ORDER BY change_pct ASC
```
**관련 테이블**: `tbnws_sabangnet_order`, `erp_order_recv` + `erp_order_recv_item` (B2B)
**추가 참조**: `erp_partner` (거래처 상세 정보)

#### Q: "신규 고객 중 재구매율 낮은 그룹은?"
```sql
-- B2B: erp_order_recv_biz의 최초 주문일 기준 신규 판별
SELECT
  b.biz_name, b.biz_code,
  MIN(r.recv_seq) as first_order,
  COUNT(r.recv_seq) as total_orders
FROM erp_order_recv_biz b
JOIN erp_order_recv r ON b.biz_code = r.biz_code
GROUP BY b.biz_code
HAVING first_order_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
  AND total_orders <= 1  -- 재구매 없음
```
**관련 테이블**: `erp_order_recv_biz`, `erp_order_recv`, `erp_order_recv_item`

---

### ⚠️ 위험 감지

#### Q: "이탈 위험 고객은 누구?"
```sql
-- 최근 3개월 주문 없는 기존 활성 거래처
SELECT b.biz_name, b.biz_code,
  MAX(r.due_date) as last_order_date,
  DATEDIFF(NOW(), MAX(r.due_date)) as days_since_last
FROM erp_order_recv_biz b
JOIN erp_order_recv r ON b.biz_code = r.biz_code
GROUP BY b.biz_code
HAVING days_since_last > 90
  AND COUNT(r.recv_seq) >= 3  -- 기존에 3건 이상 거래
ORDER BY days_since_last DESC
```
**관련 테이블**: `erp_order_recv_biz`, `erp_order_recv`
**추가 참조**: `erp_order_recv_memo` (최근 커뮤니케이션 이력)

#### Q: "할인율이 과도하게 높은 딜은?"
```sql
-- 판매가 vs MSRP 비교
SELECT
  ri.recv_seq, b.biz_name,
  p.product_name, p.msrp,
  ri.export_price,
  ROUND((1 - ri.export_price / p.msrp) * 100, 1) as discount_pct
FROM erp_order_recv_item ri
JOIN erp_order_recv r ON ri.recv_seq = r.recv_seq
JOIN erp_order_recv_biz b ON r.biz_code = b.biz_code
JOIN erp_product_info p ON ri.product_code = p.product_code
WHERE p.msrp > 0 AND ri.export_price > 0
  AND (1 - ri.export_price / p.msrp) > 0.3  -- 30% 이상 할인
ORDER BY discount_pct DESC
```
**관련 테이블**: `erp_order_recv_item`, `erp_product_info`, `erp_order_recv_biz`
**키 컬럼**: `msrp`(정가), `export_price`(출고가)

---

### 🔮 예측

#### Q: "다음 분기 매출 예측치는?"
```sql
-- 과거 연간 데이터 기반 시계열 예측
-- 데이터 소스: erp_annual_sales (2022~2026)
SELECT year, month, total_sales
FROM erp_annual_sales
WHERE year >= 2022
ORDER BY year, month
-- → LLM이 트렌드/계절성 분석 후 예측값 산출
-- → 참고: sales-dashboard 데이터 (2022: 186.3억, 2023: 187.7억, 2024: 180.8억, 2025: 234.4억)
```
**관련 테이블**: `erp_annual_sales`, `erp_annual_model_sales`
**예측 방법**: 가중평균 + 계절성 패턴 + YoY 성장률

#### Q: "수요 급증 예상 품목은?"
```sql
-- 최근 판매 속도 가속 + 검색 트렌드 상승 조합
-- orderForecast의 "판매 급증 285" 로직 활용
-- 1주 판매량 / 3달 평균 > 1.5배 = 수요 급증
SELECT p.product_name, p.product_code,
  weekly_sales, monthly_avg,
  weekly_sales / monthly_avg as acceleration
FROM (
  SELECT model_no,
    SUM(CASE WHEN order_date >= DATE_SUB(NOW(), INTERVAL 7 DAY) THEN p_ea END) as weekly_sales,
    SUM(CASE WHEN order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY) THEN p_ea END) / 12 as monthly_avg
  FROM tbnws_sabangnet_order
  WHERE order_status NOT IN ('취소')
  GROUP BY model_no
) sub
JOIN erp_product_info p ON sub.model_no = p.product_code
WHERE acceleration > 1.5
ORDER BY acceleration DESC
```
**관련 테이블**: `tbnws_sabangnet_order`, `erp_product_info`
**추가**: `naver_ranking_tracker_list` (검색 트렌드)

---

### 📦 재고 & 발주 관리

#### Q: "발주 챙겨야 하는 재고 알려줘"
```sql
-- ★ 검증 완료 (2026-02-28) — GT+CJ(풀필먼트) 합산 버전
-- ⚠️ 반드시 G-코드(GT창고) + F-코드(CJ풀필먼트) 합산해야 정확!
-- F-코드 = CONCAT('F', SUBSTRING(G-코드, 2))

SELECT
    g.productCode, p.brand_name, p.goods_name, p.option_name,
    g.ea AS gt_stock, IFNULL(f.ea, 0) AS cj_stock,
    g.ea + IFNULL(f.ea, 0) AS total_stock,
    g.beDueEa + IFNULL(f.beDueEa, 0) AS total_incoming,
    IFNULL(p.safe_ea, 0) AS safe_ea,
    g.oneMonthSalesEa + IFNULL(f.oneMonthSalesEa, 0) AS total_s1m,
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
HAVING daily_avg >= 0.3 AND days_of_supply < 30
ORDER BY
  CASE
    WHEN total_stock = 0 AND daily_avg >= 0.3 THEN 1          -- 🔴 품절
    WHEN total_stock > 0 AND total_stock < safe_ea THEN 2     -- 🟠 안전재고 미달
    WHEN days_of_supply < 14 THEN 3                            -- 🟡 2주내 소진
    WHEN days_of_supply < 30 THEN 4                            -- 🔵 1개월내 소진
  END,
  daily_avg DESC;

-- ★ 창고 체계 (wms_warehouse):
--   GT (지티기어 창고): G-, P-, K-, Y- prefix
--   GJ (CJ 풀필먼트, 곤지암): F- prefix → 네이버 물량 3PL 직접 출고
-- ⚠️ GT만 보면 품절이지만 CJ에 재고 있는 경우 다수 존재!
```
**관련 테이블**: `erp_stock_realtime` (G-코드 + F-코드), `erp_product_info`, `erp_purchase_order`
**키 컬럼**: G-코드 `ea` + F-코드 `ea` 합산, `oneMonthSalesEa` 합산, `safe_ea`, `beDueEa` 합산
**창고**: GT(지티기어, G-코드) + GJ(CJ풀필먼트, F-코드) — DB코드 'GJ' = CJ 풀필먼트

#### Q: "이미 발주된 건 확인해줘" / "임시 발주 목록?"
```sql
-- 전체 발주 현황 (임시 vs 완료 구분)
SELECT
    po_number, register_date, product_code, quantity,
    CASE WHEN order_seq IS NULL THEN '임시발주' ELSE '완료' END AS status
FROM erp_purchase_order
WHERE register_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
ORDER BY register_date DESC;

-- 임시 발주만 (아직 입고 안된 것)
SELECT po_number, product_code, quantity, register_date
FROM erp_purchase_order
WHERE order_seq IS NULL  -- ★ 임시발주 = order_seq IS NULL
ORDER BY register_date DESC;

-- 특정 제품의 발주 이력 확인
SELECT po_number, quantity, register_date,
       CASE WHEN order_seq IS NULL THEN '임시' ELSE '완료' END AS status
FROM erp_purchase_order
WHERE product_code = 'G-0029-0084-0068'
ORDER BY register_date DESC;
```
**관련 테이블**: `erp_purchase_order` (535행)
**핵심 로직**: `order_seq IS NULL` = 임시발주(PO탭), `order_seq IS NOT NULL` = 완료(발주현황탭)
**MyBatis**: `findAvailablePOrderItemList` (임시), `findPurchaseOrderAll` (전체)

#### Q: "잘 안나가다가 최근에 잘 나가는 모델?"
```sql
-- ★ 검증 완료 (2026-02-28)
-- 데이터 소스: naver_smartstore_orders (5개월 비교)
-- 이전 3개월 평균 vs 최근 2개월 평균 비교
SELECT sub_model,
    ROUND(AVG(CASE WHEN payment_date >= DATE_SUB(CURDATE(), INTERVAL 2 MONTH) THEN monthly_qty END)) AS recent_avg,
    ROUND(AVG(CASE WHEN payment_date < DATE_SUB(CURDATE(), INTERVAL 2 MONTH)
                    AND payment_date >= DATE_SUB(CURDATE(), INTERVAL 5 MONTH) THEN monthly_qty END)) AS prev_avg,
    -- growth_rate = (recent - prev) / prev * 100
    ...
FROM (서브모델 분류 CASE WHEN) naver_smartstore_orders
WHERE product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
  AND payment_date >= DATE_SUB(CURDATE(), INTERVAL 5 MONTH)
GROUP BY sub_model
HAVING recent_avg > prev_avg * 1.3;  -- 30% 이상 성장
```
**관련 테이블**: `naver_smartstore_orders`
**참고**: 서브모델 분류 CASE WHEN은 본 문서 "AI 실거래가 조회 SQL" 섹션 참조

#### Q: "재고 추이 보여줘" / "재고가 언제 바닥나?"
```sql
-- erp_stock_summary 일별 스냅샷으로 추이 분석
SELECT summary_date, ea, be_due_ea, one_month_sales_ea
FROM erp_stock_summary
WHERE product_code = 'G-0029-0509-0001'
  AND summary_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
ORDER BY summary_date;
-- → 일별 ea 감소 추이로 소진일 예측 가능
```
**관련 테이블**: `erp_stock_summary` (261,207행, 일별 스냅샷)

---

### 🎯 의사결정

#### Q: "마진 좋은데 영업 안 밀고 있는 제품은?"
```sql
-- 마진율 높지만 판매 수량 적은 상품
SELECT p.product_name,
  p.msrp, oi.buying_price, oi.cur2krw,
  (p.msrp - oi.buying_price * oi.cur2krw) / p.msrp * 100 as margin_pct,
  COALESCE(sales.total_ea, 0) as total_sold
FROM erp_product_info p
JOIN erp_order_item oi ON p.product_code = oi.product_code
LEFT JOIN (
  SELECT model_no, SUM(p_ea) as total_ea
  FROM tbnws_sabangnet_order
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
  GROUP BY model_no
) sales ON p.product_code = sales.model_no
WHERE p.sell_status = 'Y'
  AND margin_pct > 40  -- 마진 40% 이상
  AND total_sold < 10  -- 3개월 판매 10개 미만
ORDER BY margin_pct DESC
```
**관련 테이블**: `erp_product_info`, `erp_order_item`, `tbnws_sabangnet_order`
**키 컬럼**: `msrp`, `buying_price`, `cur2krw`(환율), `duty`(관세)

---

## 4. 세일즈 담당 (B2C 포함)

### 🔎 제품별 성과

#### Q: "매출 대비 광고비 비효율 모델은?"
```
-- 데이터 소스 2개 조합 필요:
-- 1) tbe.kr DB → SA/GFA 키워드별 광고비, 전환매출
-- 2) tbnws_sabangnet_order → 실제 매출
-- tbe.kr의 "비효율 키워드 TOP 10" 섹션 참조
-- ROAS < 100% 인 키워드/캠페인이 비효율
```
**tbe.kr 데이터**: SA 키워드별 소진금액, 전환매출, ROAS
**TBNWS DB**: `tbnws_sabangnet_order` (실제 매출), `naver_search_ad` (광고 데이터)

#### Q: "객단가 상승 추세 채널은?"
```sql
-- AOV(Average Order Value) 채널별 월별 추이
SELECT
  mall_id,
  DATE_FORMAT(order_date, '%Y-%m') as month,
  SUM(pay_cost) / COUNT(*) as aov
FROM tbnws_sabangnet_order
WHERE order_status NOT IN ('취소')
  AND order_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
GROUP BY mall_id, month
ORDER BY mall_id, month
-- → AOV가 연속 3개월 상승하는 채널 필터
```
**관련 테이블**: `tbnws_sabangnet_order`
**키 컬럼**: `pay_cost`(결제금액), `mall_id`(채널)

#### Q: "반품률 높은 SKU는?"
```sql
-- 반품 데이터: crm_return_management + erp_stock (return_flag='Y')
SELECT
  p.product_name, p.product_code,
  COUNT(CASE WHEN s.return_flag = 'Y' THEN 1 END) as returns,
  total_sold.ea as total_sales,
  ROUND(returns / total_sales * 100, 1) as return_rate
FROM erp_stock s
JOIN erp_product_info p ON s.product_code = p.product_code
LEFT JOIN (
  SELECT model_no, SUM(p_ea) as ea
  FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소')
  AND order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
  GROUP BY model_no
) total_sold ON p.product_code = total_sold.model_no
WHERE s.return_flag = 'Y'
GROUP BY p.product_code
HAVING return_rate > 5  -- 반품률 5% 이상
ORDER BY return_rate DESC
```
**관련 테이블**: `erp_stock`(return_flag), `crm_return_management`, `crm_return_product`
**추가**: `crm_as`(A/S 요청), `customer_refund`(환불)

---

### ⚠️ 이상징후

#### Q: "재고 과잉 가능 SKU는?"
```sql
-- inventory-analysis의 "체류 재고" 섹션 로직
-- 사장 재고: 90일간 판매 0건
-- 저회전: 12개월+ 소진 예상
SELECT p.product_name,
  stock.current_ea,
  stock.current_ea * p.msrp as stock_value,
  COALESCE(sales.weekly_ea, 0) as weekly_sales,
  CASE
    WHEN sales.weekly_ea = 0 THEN '사장재고'
    WHEN stock.current_ea / sales.weekly_ea > 52 THEN '저회전'
    WHEN stock.current_ea / sales.weekly_ea > 26 THEN '과잉'
    ELSE '정상'
  END as status
FROM erp_product_info p
JOIN (
  SELECT product_code, COUNT(*) as current_ea
  FROM erp_stock WHERE export_date IS NULL
  GROUP BY product_code
) stock ON p.product_code = stock.product_code
LEFT JOIN (
  SELECT model_no, SUM(p_ea) / 12 as weekly_ea  -- 3달 주간 평균
  FROM tbnws_sabangnet_order
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
  GROUP BY model_no
) sales ON p.product_code = sales.model_no
WHERE status IN ('사장재고', '저회전', '과잉')
ORDER BY stock_value DESC
```
**관련 테이블**: `erp_stock`, `erp_product_info`, `tbnws_sabangnet_order`
**참조**: inventory-analysis 대시보드 (체류 재고 24.4억: 사장 6.6억 + 저회전 17.7억)

#### Q: "가격 인하 없이도 잘 팔리는 모델은?"
```sql
-- 정가 판매 + 높은 판매량 = 가격 경쟁력 있는 제품
SELECT p.product_name, p.msrp,
  AVG(s.pay_cost / s.p_ea) as avg_selling_price,
  SUM(s.p_ea) as total_ea,
  (1 - avg_selling_price / p.msrp) as discount_ratio
FROM tbnws_sabangnet_order s
JOIN erp_product_info p ON s.model_no = p.product_code
WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
  AND s.order_status NOT IN ('취소')
  AND p.msrp > 0
GROUP BY p.product_code
HAVING discount_ratio < 0.05  -- 할인 5% 미만
  AND total_ea >= 50  -- 월 50개 이상
ORDER BY total_ea DESC
```
**관련 테이블**: `tbnws_sabangnet_order`, `erp_product_info`, `erp_discounted_products`

---

### 🔮 예측

#### Q: "시즌별 판매량 예측"
```
-- erp_annual_sales 4년 데이터 (2022~2025) 기반 계절성 분석
-- sales-analysis의 "년별 비교 매출" 탭 데이터:
-- 2022: 186.3억 | 2023: 187.7억 | 2024: 180.8억 | 2025: 234.4억
-- 월별 패턴: 1월(피크), 12월(피크), 6~7월(저점)
-- 계절성 지수 = 해당월 평균 / 연간 월평균
```
**관련 테이블**: `erp_annual_sales`, `erp_annual_model_sales`

#### Q: "단종 후보는?"
```sql
-- 판매 감소 추세 + 낮은 마진 + 재고 적음
SELECT p.product_name, p.product_code, p.sell_status,
  recent.ea as recent_3m,
  older.ea as older_3m,
  ROUND((recent.ea - older.ea) / older.ea * 100, 1) as trend
FROM erp_product_info p
LEFT JOIN (...) recent  -- 최근 3개월 판매
LEFT JOIN (...) older   -- 6~3개월 전 판매
WHERE p.sell_status = 'Y'
  AND trend < -50  -- 50% 이상 감소
  AND recent.ea < 10
ORDER BY trend ASC
```
**관련 테이블**: `erp_product_info`, `tbnws_sabangnet_order`

---

### 📉 판매 급감 이상 감지 (고급)

#### Q: "지난 2주간 판매 급감 SKU는?"
```sql
SELECT pi.product_name, pi.product_code,
  recent.ea as last_2wk,
  prev.ea as prev_2wk,
  ROUND((recent.ea - prev.ea) / prev.ea * 100, 1) as change_pct
FROM erp_product_info pi
JOIN (SELECT model_no, SUM(p_ea) as ea
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 14 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) recent ON pi.product_code = recent.model_no
JOIN (SELECT model_no, SUM(p_ea) as ea
      FROM tbnws_sabangnet_order
      WHERE order_date BETWEEN DATE_SUB(NOW(), INTERVAL 28 DAY) AND DATE_SUB(NOW(), INTERVAL 14 DAY)
        AND order_status NOT IN ('취소')
      GROUP BY model_no) prev ON pi.product_code = prev.model_no
WHERE prev.ea >= 10  -- 기존 판매량 유의미
  AND (recent.ea - prev.ea) / prev.ea < -0.3  -- 30% 이상 감소
ORDER BY change_pct ASC
```
**관련 테이블**: `tbnws_sabangnet_order`, `erp_product_info`

#### Q: "가격 변화 없이 수요 감소한 SKU는?"
```sql
-- 할인 이력 변화 없는데 판매 하락
SELECT pi.product_name,
  AVG(CASE WHEN s.order_date >= DATE_SUB(NOW(), INTERVAL 14 DAY) THEN s.pay_cost/s.p_ea END) as recent_asp,
  AVG(CASE WHEN s.order_date < DATE_SUB(NOW(), INTERVAL 14 DAY) THEN s.pay_cost/s.p_ea END) as prev_asp,
  SUM(CASE WHEN s.order_date >= DATE_SUB(NOW(), INTERVAL 14 DAY) THEN s.p_ea ELSE 0 END) as recent_qty,
  SUM(CASE WHEN s.order_date < DATE_SUB(NOW(), INTERVAL 14 DAY) THEN s.p_ea ELSE 0 END) as prev_qty
FROM tbnws_sabangnet_order s
JOIN erp_product_info pi ON s.model_no = pi.product_code
WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 28 DAY) AND s.order_status NOT IN ('취소')
GROUP BY pi.product_code
HAVING ABS(recent_asp - prev_asp) / prev_asp < 0.03  -- 가격 변화 3% 미만
  AND recent_qty < prev_qty * 0.7  -- 수량 30% 이상 감소
ORDER BY (recent_qty / prev_qty) ASC
```
**핵심**: 가격 동일 + 판매 하락 = 외부 요인(경쟁, 시즌, 트렌드) 시그널

#### Q: "리뷰 평점 하락과 판매 하락이 동시에 발생한 모델은?"
```sql
SELECT pi.product_name,
  review.recent_avg as recent_rating,
  review.older_avg as prev_rating,
  sales.recent_ea, sales.prev_ea,
  ROUND((sales.recent_ea / sales.prev_ea - 1) * 100, 1) as sales_change_pct,
  ROUND(review.recent_avg - review.older_avg, 2) as rating_change
FROM erp_product_info pi
JOIN (SELECT product_code,
        AVG(CASE WHEN review_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN score END) as recent_avg,
        AVG(CASE WHEN review_date < DATE_SUB(NOW(), INTERVAL 30 DAY) THEN score END) as older_avg
      FROM naver_store_review_item GROUP BY product_code) review
  ON pi.product_code = review.product_code
JOIN (SELECT model_no,
        SUM(CASE WHEN order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN p_ea ELSE 0 END) as recent_ea,
        SUM(CASE WHEN order_date BETWEEN DATE_SUB(NOW(), INTERVAL 60 DAY) AND DATE_SUB(NOW(), INTERVAL 30 DAY) THEN p_ea ELSE 0 END) as prev_ea
      FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소') GROUP BY model_no) sales
  ON pi.product_code = sales.model_no
WHERE review.recent_avg < review.older_avg - 0.2  -- 평점 0.2점 이상 하락
  AND sales.recent_ea < sales.prev_ea * 0.8  -- 판매 20% 이상 하락
ORDER BY rating_change ASC
```
**관련 테이블**: `naver_store_review_item`, `tbnws_sabangnet_order`

---

### 📈 재입고 후 회복 전략

#### Q: "재입고 후 초기 7일 예상 소화율은?"
```sql
-- 과거 동일 SKU 재입고 이력 기반 7일 소화 패턴
SELECT pi.product_name,
  hist.avg_7day_sellthrough,
  stock.available_ea as current_stock,
  ROUND(stock.available_ea * hist.avg_7day_sellthrough / 100) as expected_7day_sales
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
JOIN (-- 과거 재입고 후 7일 소화율 평균
      SELECT s.product_code,
        AVG(sold_7d.ea / restock.ea * 100) as avg_7day_sellthrough
      FROM (SELECT product_code, import_date, COUNT(*) as ea
            FROM erp_stock GROUP BY product_code, import_date) restock
      JOIN (SELECT model_no, order_date, SUM(p_ea) as ea
            FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소')
            GROUP BY model_no, order_date) sold_7d
        ON restock.product_code = sold_7d.model_no
        AND sold_7d.order_date BETWEEN restock.import_date AND DATE_ADD(restock.import_date, INTERVAL 7 DAY)
      GROUP BY s.product_code) hist ON pi.product_code = hist.product_code
```

#### Q: "재입고 후 프로모션 없이 자연 회복 가능한 SKU는?"
```sql
-- 과거 재입고 시 프로모션 없이도 정상 판매속도 회복한 SKU
SELECT pi.product_name,
  pre_oos.daily_avg as pre_oos_sales,
  post_restock.daily_avg as post_restock_sales,
  ROUND(post_restock.daily_avg / pre_oos.daily_avg * 100, 1) as recovery_rate,
  CASE
    WHEN post_restock.daily_avg >= pre_oos.daily_avg * 0.9 THEN '자연 회복 가능'
    WHEN post_restock.daily_avg >= pre_oos.daily_avg * 0.7 THEN '부분 프로모션 필요'
    ELSE '적극 프로모션 필요'
  END as recommendation
FROM erp_product_info pi
JOIN (/*OOS 직전 판매속도*/) pre_oos ON pi.product_code = pre_oos.model_no
JOIN (/*재입고 후 14일 판매속도*/) post_restock ON pi.product_code = post_restock.model_no
WHERE pre_oos.daily_avg > 1
ORDER BY recovery_rate DESC
```
**핵심**: 회복율 90%+ = 자연회복, 70~90% = 부분 프로모션, 70%- = 적극 프로모션

---

### 💰 가격 & 프로모션 의사결정 (고급)

#### Q: "현재 마진 30% 이상인데 판매 부진 SKU는?"
```sql
SELECT pi.product_name, pi.product_code,
  pi.msrp, oi.buying_price, oi.cur2krw,
  ROUND((pi.msrp - oi.buying_price * oi.cur2krw - COALESCE(pi.duty, 0)) / pi.msrp * 100, 1) as margin_pct,
  COALESCE(sales.monthly_ea, 0) as monthly_sales,
  CASE
    WHEN sales.monthly_ea < 5 THEN '심각 부진'
    WHEN sales.monthly_ea < 20 THEN '부진'
    ELSE '보통'
  END as sales_status
FROM erp_product_info pi
JOIN erp_order_item oi ON pi.product_code = oi.product_code
  AND oi.item_seq = (SELECT MAX(item_seq) FROM erp_order_item WHERE product_code = pi.product_code)
LEFT JOIN (SELECT model_no, SUM(p_ea) as monthly_ea
           FROM tbnws_sabangnet_order
           WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
           GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE pi.sell_status = 'Y' AND pi.msrp > 0
  AND (pi.msrp - oi.buying_price * oi.cur2krw) / pi.msrp > 0.30
  AND COALESCE(sales.monthly_ea, 0) < 20
ORDER BY margin_pct DESC
```

#### Q: "5% 할인 시 매출 증가 예상 폭은? / 10% 할인 시 이익총액은 오히려 감소하는 SKU는?"
```sql
-- 가격 탄력성 기반 시뮬레이션
-- 과거 할인 이벤트 전후 판매량 변화로 탄력성 추정
SELECT pi.product_name,
  pi.msrp, current_margin.margin_pct,
  sales.monthly_ea,
  sales.monthly_revenue,
  -- 5% 할인 시뮬레이션 (탄력성 1.5 가정)
  ROUND(sales.monthly_ea * 1.075) as est_qty_5pct_off,  -- 수량 7.5% 증가
  ROUND(sales.monthly_revenue * pi.msrp * 0.95 / pi.msrp * 1.075) as est_rev_5pct_off,
  -- 10% 할인 시 이익 비교
  ROUND(sales.monthly_ea * current_margin.margin_pct / 100 * pi.msrp) as current_profit,
  ROUND(sales.monthly_ea * 1.15 * (current_margin.margin_pct - 10) / 100 * pi.msrp) as profit_10pct_off,
  CASE
    WHEN sales.monthly_ea * 1.15 * (current_margin.margin_pct - 10) <
         sales.monthly_ea * current_margin.margin_pct
    THEN '10% 할인 시 이익 감소'
    ELSE '10% 할인 유효'
  END as discount_verdict
FROM erp_product_info pi
JOIN (/*마진율*/) current_margin ON pi.product_code = current_margin.product_code
JOIN (/*월간 판매*/) sales ON pi.product_code = sales.model_no
WHERE pi.sell_status = 'Y'
ORDER BY discount_verdict, margin_pct DESC
```
**핵심**: 마진율 < 할인율 × (1 + 수량증가율) 이면 할인 시 이익 감소

#### Q: "단종 전 재고 소진 전략 필요한 SKU는?"
```sql
SELECT pi.product_name, pi.product_code,
  stock.available_ea,
  stock.available_ea * pi.msrp as stock_value_at_msrp,
  sales.daily_avg,
  CASE
    WHEN sales.daily_avg = 0 THEN '판매 중단'
    ELSE ROUND(stock.available_ea / sales.daily_avg)
  END as days_to_clear,
  -- 단종 후보: 판매 감소 추세 + 후속 모델 존재 + 잔여 재고
  CASE
    WHEN sales.daily_avg < 0.5 AND stock.available_ea > 50 THEN '즉시 할인 소진'
    WHEN sales.daily_avg < 1 AND stock.available_ea > 100 THEN '프로모션 소진'
    ELSE '정상 판매 유지'
  END as clearance_strategy
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
LEFT JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg
           FROM tbnws_sabangnet_order
           WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
           GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE pi.sell_status = 'Y'
  AND (sales.daily_avg < 1 OR sales.daily_avg IS NULL)
  AND stock.available_ea > 20
ORDER BY stock_value_at_msrp DESC
```

---

### 🧠 시즌/시장 국면 판단

#### Q: "현재 판매 데이터 기준 시장은 상승기/정체기/하락기 중 어디?"
```sql
-- 4주 이동평균 추이로 국면 판단
SELECT week_label,
  weekly_revenue,
  AVG(weekly_revenue) OVER (ORDER BY week_num ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) as ma_4wk,
  LAG(weekly_revenue, 4) OVER (ORDER BY week_num) as same_week_last_month,
  CASE
    WHEN weekly_revenue > AVG(weekly_revenue) OVER (ORDER BY week_num ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) * 1.05
    THEN '상승기'
    WHEN weekly_revenue < AVG(weekly_revenue) OVER (ORDER BY week_num ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) * 0.95
    THEN '하락기'
    ELSE '정체기'
  END as market_phase
FROM (
  SELECT YEARWEEK(order_date) as week_num,
    CONCAT(YEAR(order_date), '-W', WEEK(order_date)) as week_label,
    SUM(pay_cost) as weekly_revenue
  FROM tbnws_sabangnet_order
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 16 WEEK) AND order_status NOT IN ('취소')
  GROUP BY YEARWEEK(order_date)
) weekly
ORDER BY week_num DESC
```
**판단 기준**: 4주MA 대비 현재 주 매출 → +5% 상승기, -5% 하락기

#### Q: "전년 동기 대비 수요 증감률은?"
```sql
SELECT
  DATE_FORMAT(s.order_date, '%Y-%m') as month,
  SUM(CASE WHEN YEAR(s.order_date) = YEAR(NOW()) THEN s.pay_cost END) as this_year,
  SUM(CASE WHEN YEAR(s.order_date) = YEAR(NOW())-1 THEN s.pay_cost END) as last_year,
  ROUND((SUM(CASE WHEN YEAR(s.order_date) = YEAR(NOW()) THEN s.pay_cost END) /
         SUM(CASE WHEN YEAR(s.order_date) = YEAR(NOW())-1 THEN s.pay_cost END) - 1) * 100, 1) as yoy_pct
FROM tbnws_sabangnet_order s
WHERE MONTH(s.order_date) = MONTH(NOW()) AND order_status NOT IN ('취소')
GROUP BY MONTH(s.order_date)
```
**보조 데이터**: `erp_annual_sales` (2022~2025 연간 매출 이력)

---

### 🎯 영업 전략 포트폴리오

#### Q: "상위 20% SKU가 매출 몇 % 차지해?"
```sql
-- 파레토 분석 (80/20 법칙)
SELECT
  COUNT(*) as total_skus,
  SUM(CASE WHEN rn <= total * 0.2 THEN revenue ELSE 0 END) as top20_revenue,
  SUM(revenue) as total_revenue,
  ROUND(SUM(CASE WHEN rn <= total * 0.2 THEN revenue ELSE 0 END) / SUM(revenue) * 100, 1) as top20_share_pct
FROM (
  SELECT model_no, SUM(pay_cost) as revenue,
    ROW_NUMBER() OVER (ORDER BY SUM(pay_cost) DESC) as rn,
    COUNT(*) OVER () as total
  FROM tbnws_sabangnet_order
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND order_status NOT IN ('취소')
  GROUP BY model_no
) ranked
```

#### Q: "마진은 낮은데 매출 비중 큰 SKU는?"
```sql
SELECT pi.product_name,
  sales.revenue, sales.revenue_share_pct,
  margin.margin_pct,
  CASE
    WHEN margin.margin_pct < 20 AND sales.revenue_share_pct > 5 THEN '고매출-저마진 위험'
    WHEN margin.margin_pct < 15 THEN '마진 개선 시급'
    ELSE '모니터링'
  END as alert
FROM erp_product_info pi
JOIN (SELECT model_no, SUM(pay_cost) as revenue,
        SUM(pay_cost) / (SELECT SUM(pay_cost) FROM tbnws_sabangnet_order
                         WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)) * 100 as revenue_share_pct
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
JOIN (/*마진율 계산*/) margin ON pi.product_code = margin.product_code
WHERE margin.margin_pct < 20 AND sales.revenue_share_pct > 3
ORDER BY sales.revenue_share_pct DESC
```

#### Q: "특정 채널 의존도 50% 넘는 SKU는?"
```sql
SELECT pi.product_name,
  s.mall_id as dominant_channel,
  channel_sales.channel_revenue,
  total_sales.total_revenue,
  ROUND(channel_sales.channel_revenue / total_sales.total_revenue * 100, 1) as channel_dependency_pct
FROM erp_product_info pi
JOIN (SELECT model_no, mall_id, SUM(pay_cost) as channel_revenue
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND order_status NOT IN ('취소')
      GROUP BY model_no, mall_id) channel_sales ON pi.product_code = channel_sales.model_no
JOIN (SELECT model_no, SUM(pay_cost) as total_revenue
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND order_status NOT IN ('취소')
      GROUP BY model_no) total_sales ON pi.product_code = total_sales.model_no
WHERE channel_sales.channel_revenue / total_sales.total_revenue > 0.5
ORDER BY channel_dependency_pct DESC
```
**핵심**: 단일 채널 의존도 50%+ = 채널 리스크 높음 → 분산 전략 필요

---

## 5. CS 담당

### 🔎 VOC 분석

#### Q: "클레임 급증 제품은?"
```sql
-- crm_as 테이블: A/S 접수 건수 추이
SELECT
  ap.product_code, p.product_name,
  COUNT(CASE WHEN a.regist_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN 1 END) as this_month,
  COUNT(CASE WHEN a.regist_date BETWEEN DATE_SUB(NOW(), INTERVAL 2 MONTH) AND DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN 1 END) as last_month
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN erp_product_info p ON ap.product_code = p.product_code
GROUP BY ap.product_code
HAVING this_month > last_month * 1.5  -- 50% 이상 증가
ORDER BY this_month DESC
```
**관련 테이블**: `crm_as`, `crm_as_product`, `crm_as_history`, `crm_as_memo`

#### Q: "동일 이슈 반복 발생 품목은?"
```sql
-- crm_as 내용 분석 + 상품별 그룹핑
-- crm_as_memo에서 키워드 추출
SELECT ap.product_code, p.product_name,
  COUNT(*) as issue_count,
  GROUP_CONCAT(DISTINCT am.memo_content SEPARATOR ' | ') as issues
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN crm_as_memo am ON a.seq = am.as_seq
JOIN erp_product_info p ON ap.product_code = p.product_code
WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
GROUP BY ap.product_code
HAVING issue_count >= 5
ORDER BY issue_count DESC
```
**관련 테이블**: `crm_as`, `crm_as_product`, `crm_as_memo`
**추가**: `naver_store_customer_inquiries` (네이버 문의), `naver_store_product_qna` (Q&A)

#### Q: "리뷰 평점 하락 제품은?"
```sql
-- naver_store_review_item 리뷰 평점 추이
SELECT product_code,
  AVG(CASE WHEN review_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN score END) as recent_avg,
  AVG(CASE WHEN review_date < DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN score END) as older_avg
FROM naver_store_review_item
GROUP BY product_code
HAVING recent_avg < older_avg - 0.3  -- 0.3점 이상 하락
```
**관련 테이블**: `naver_store_review_item`, `naver_store_review_photo`, `naver_store_review_crawler`

---

### ⚠️ 위기 감지

#### Q: "환불 증가율 높은 제품은?"
```sql
SELECT p.product_name,
  COUNT(CASE WHEN cr.regist_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN 1 END) as recent_refunds,
  COUNT(CASE WHEN cr.regist_date < DATE_SUB(NOW(), INTERVAL 1 MONTH) THEN 1 END) as older_refunds
FROM customer_refund cr
JOIN erp_product_info p ON cr.product_code = p.product_code
GROUP BY p.product_code
HAVING recent_refunds > older_refunds * 1.5
```
**관련 테이블**: `customer_refund`, `crm_refund`, `crm_return_management`, `crm_return_product`
**추가**: `tbnws_sabangnet_claim` (사방넷 클레임)

#### Q: "특정 로트 문제 가능성은?"
```sql
-- 동일 발주(order_seq) 입고분에서 A/S 비율 확인
SELECT o.order_seq, o.order_title,
  oi.product_code, p.product_name,
  oi.ea as order_qty,
  COUNT(DISTINCT a.seq) as as_count,
  ROUND(as_count / oi.ea * 100, 1) as defect_rate
FROM erp_order o
JOIN erp_order_item oi ON o.order_seq = oi.order_seq
JOIN erp_stock s ON oi.item_seq = s.item_seq
LEFT JOIN crm_as_product ap ON s.product_code = ap.product_code
LEFT JOIN crm_as a ON ap.as_seq = a.seq
GROUP BY o.order_seq, oi.product_code
HAVING defect_rate > 3  -- 3% 이상 불량
```
**관련 테이블**: `erp_order`, `erp_order_item`, `erp_stock`, `crm_as`, `crm_as_product`
**핵심 관계**: `erp_stock.item_seq → erp_order_item.item_seq` (로트 추적)

---

### 🔥 이슈 트렌드 분석 (고급)

#### Q: "요즘 가장 많은 이슈(AS/클레임 유형)가 뭐야?"
```sql
-- AS 유형별 Top 10 + 추이 분석
SELECT
  COALESCE(a.as_type, '미분류') as issue_type,
  pi.product_name,
  COUNT(*) as issue_count,
  COUNT(CASE WHEN a.regist_date >= DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 1 END) as last_7d,
  COUNT(CASE WHEN a.regist_date BETWEEN DATE_SUB(NOW(), INTERVAL 14 DAY)
                                    AND DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 1 END) as prev_7d,
  ROUND((last_7d - prev_7d) / NULLIF(prev_7d, 0) * 100, 1) as trend_pct,
  -- 키워드 빈도 분석 (AS 메모에서 추출)
  GROUP_CONCAT(DISTINCT
    CASE
      WHEN am.memo_content LIKE '%고장%' THEN '고장'
      WHEN am.memo_content LIKE '%불량%' THEN '불량'
      WHEN am.memo_content LIKE '%파손%' THEN '파손'
      WHEN am.memo_content LIKE '%교환%' THEN '교환요청'
      WHEN am.memo_content LIKE '%환불%' THEN '환불요청'
      WHEN am.memo_content LIKE '%배송%' THEN '배송이슈'
      WHEN am.memo_content LIKE '%누락%' THEN '구성품누락'
      WHEN am.memo_content LIKE '%소음%' THEN '소음'
      WHEN am.memo_content LIKE '%연결%' OR am.memo_content LIKE '%블루투스%' THEN '연결불량'
    END
  ) as keyword_tags
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN erp_product_info pi ON ap.product_code = pi.product_code
LEFT JOIN crm_as_memo am ON a.seq = am.as_seq
WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY ap.product_code
ORDER BY issue_count DESC
LIMIT 20
```
**파생 질문**: "이슈 유형별 평균 해결 시간은?" / "해결 못하고 장기 대기 중인 건은?"
```sql
-- 미해결 건 + 대기 기간
SELECT a.seq, pi.product_name,
  a.regist_date, DATEDIFF(NOW(), a.regist_date) as pending_days,
  ah.status as current_status,
  GROUP_CONCAT(am.memo_content ORDER BY am.regist_date DESC LIMIT 1) as last_memo
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN erp_product_info pi ON ap.product_code = pi.product_code
LEFT JOIN crm_as_history ah ON a.seq = ah.as_seq
LEFT JOIN crm_as_memo am ON a.seq = am.as_seq
WHERE ah.status NOT IN ('완료', '종료')
  AND DATEDIFF(NOW(), a.regist_date) > 7  -- 7일 이상 미해결
ORDER BY pending_days DESC
```

#### Q: "본사에 요청해서 해결해야 할 문제가 있어?"
```sql
-- 본사 대응 필요 건: 동일 모델 반복 이슈 (구조적 문제)
-- 기준: 동일 product_code에서 3건 이상 동일 유형 이슈 → 본사 품질 피드백 필요
SELECT pi.product_name, pi.product_code,
  pi.brand_code,
  COUNT(*) as total_issues,
  COUNT(DISTINCT CASE
    WHEN am.memo_content LIKE '%불량%' OR am.memo_content LIKE '%하자%' THEN a.seq
  END) as defect_issues,
  COUNT(DISTINCT CASE
    WHEN am.memo_content LIKE '%설명서%' OR am.memo_content LIKE '%매뉴얼%' THEN a.seq
  END) as manual_issues,
  -- 불량률 = AS건수 / 판매수량
  ROUND(COUNT(*) / NULLIF(sales.total_ea, 0) * 100, 2) as defect_rate_pct,
  CASE
    WHEN defect_rate_pct > 5 THEN '본사 긴급 보고 필요'
    WHEN defect_rate_pct > 2 THEN '본사 품질 피드백 필요'
    WHEN COUNT(DISTINCT CASE WHEN am.memo_content LIKE '%설명서%' THEN a.seq END) > 5
      THEN '매뉴얼/가이드 개선 요청'
    ELSE '모니터링'
  END as escalation_level,
  -- 구체적 요청 사항 제안
  CASE
    WHEN defect_rate_pct > 5 THEN CONCAT('품질 검수 강화 요청 (불량률 ', defect_rate_pct, '%)')
    WHEN manual_issues > 5 THEN '한글 매뉴얼/영상 가이드 제작 요청'
    WHEN COUNT(DISTINCT CASE WHEN am.memo_content LIKE '%펌웨어%' THEN a.seq END) > 3
      THEN '펌웨어 업데이트 요청'
  END as suggested_request
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN erp_product_info pi ON ap.product_code = pi.product_code
LEFT JOIN crm_as_memo am ON a.seq = am.as_seq
LEFT JOIN (SELECT model_no, SUM(p_ea) as total_ea
           FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소')
             AND order_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
           GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
GROUP BY pi.product_code
HAVING total_issues >= 5 OR defect_rate_pct > 2
ORDER BY defect_rate_pct DESC
```

#### Q: "문제가 가장 많은 모델은? (원인별 분류)"
```sql
-- 모델별 이슈 원인 분류 매트릭스
SELECT pi.product_name, pi.product_code,
  COUNT(*) as total_as,
  -- 원인별 분류 (AS 메모 키워드 분석)
  SUM(CASE WHEN am.memo_content REGEXP '불량|하자|결함|고장' THEN 1 ELSE 0 END) as hardware_defect,
  SUM(CASE WHEN am.memo_content REGEXP '연결|블루투스|무선|WiFi|페어링' THEN 1 ELSE 0 END) as connectivity,
  SUM(CASE WHEN am.memo_content REGEXP '펌웨어|업데이트|소프트웨어' THEN 1 ELSE 0 END) as firmware,
  SUM(CASE WHEN am.memo_content REGEXP '파손|깨짐|배송|포장' THEN 1 ELSE 0 END) as shipping_damage,
  SUM(CASE WHEN am.memo_content REGEXP '사용법|모르|어떻게|설정' THEN 1 ELSE 0 END) as user_guide,
  SUM(CASE WHEN am.memo_content REGEXP '누락|구성품|빠져' THEN 1 ELSE 0 END) as missing_parts,
  SUM(CASE WHEN am.memo_content REGEXP '소음|소리|진동' THEN 1 ELSE 0 END) as noise,
  -- 주요 원인 판정
  CASE
    WHEN hardware_defect > total_as * 0.4 THEN '하드웨어 품질 이슈'
    WHEN connectivity > total_as * 0.3 THEN '연결성 이슈'
    WHEN user_guide > total_as * 0.3 THEN '사용자 가이드 부족'
    WHEN shipping_damage > total_as * 0.2 THEN '포장/배송 이슈'
    ELSE '복합 이슈'
  END as primary_cause
FROM crm_as a
JOIN crm_as_product ap ON a.seq = ap.as_seq
JOIN erp_product_info pi ON ap.product_code = pi.product_code
LEFT JOIN crm_as_memo am ON a.seq = am.as_seq
WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
GROUP BY pi.product_code
HAVING total_as >= 3
ORDER BY total_as DESC
```
**파생 질문**: "이번 달 갑자기 이슈 급증한 모델은? (전월 대비 200%+)"

---

### 🔍 초기 하자 감지 (Early Defect Detection)

#### Q: "판매 초기인데 제품 하자가 있는 제품은?"
```sql
-- 출시 90일 이내 제품 중 AS 비율 이상 높은 건
SELECT pi.product_name, pi.product_code,
  MIN(s.order_date) as first_sale_date,
  DATEDIFF(NOW(), MIN(s.order_date)) as days_since_launch,
  sales.total_sold,
  as_data.as_count,
  ROUND(as_data.as_count / sales.total_sold * 100, 2) as early_defect_rate,
  -- 동일 카테고리 평균 대비
  ROUND(as_data.as_count / sales.total_sold * 100 / category_avg.avg_defect_rate, 1) as vs_category_avg,
  CASE
    WHEN as_data.as_count / sales.total_sold > 0.05 THEN '즉시 점검 필요 (불량률 5%+)'
    WHEN as_data.as_count / sales.total_sold > 0.03 THEN '주의 관찰 (불량률 3%+)'
    WHEN as_data.as_count / sales.total_sold > category_avg.avg_defect_rate * 2
      THEN '카테고리 평균 2배 초과'
    ELSE '정상 범위'
  END as alert_level
FROM erp_product_info pi
JOIN tbnws_sabangnet_order s ON pi.product_code = s.model_no AND s.order_status NOT IN ('취소')
JOIN (SELECT model_no, SUM(p_ea) as total_sold
      FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소') GROUP BY model_no) sales
  ON pi.product_code = sales.model_no
LEFT JOIN (SELECT ap.product_code, COUNT(*) as as_count
           FROM crm_as a JOIN crm_as_product ap ON a.seq = ap.as_seq
           GROUP BY ap.product_code) as_data ON pi.product_code = as_data.product_code
LEFT JOIN (-- 카테고리별 평균 불량률
  SELECT pi2.brand_code, AVG(ac / sl * 100) as avg_defect_rate
  FROM (/*브랜드별 AS건수*/) ac_tbl
  JOIN (/*브랜드별 판매수*/) sl_tbl ON ac_tbl.brand_code = sl_tbl.brand_code
  GROUP BY pi2.brand_code) category_avg ON pi.brand_code = category_avg.brand_code
WHERE DATEDIFF(NOW(), MIN(s.order_date)) <= 90  -- 출시 90일 이내
GROUP BY pi.product_code
HAVING total_sold >= 20  -- 최소 판매 20개 이상
ORDER BY early_defect_rate DESC
```
**파생 질문**: "특정 발주 로트에서만 불량 집중 발생하는 건 없어?"
→ `erp_stock.item_seq → erp_order_item.item_seq` 로트 추적

#### Q: "신제품 초기 리뷰에서 부정 키워드 패턴이 보이는 제품은?"
```sql
SELECT pi.product_name,
  COUNT(*) as total_reviews,
  AVG(r.score) as avg_score,
  SUM(CASE WHEN r.score <= 2 THEN 1 ELSE 0 END) as negative_reviews,
  ROUND(SUM(CASE WHEN r.score <= 2 THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as negative_pct,
  MIN(r.review_date) as first_review_date,
  DATEDIFF(NOW(), MIN(r.review_date)) as days_since_first_review
FROM naver_store_review_item r
JOIN erp_product_info pi ON r.product_code = pi.product_code
WHERE DATEDIFF(NOW(), r.review_date) <= 60  -- 최근 60일 리뷰
GROUP BY pi.product_code
HAVING total_reviews >= 5
  AND negative_pct > 20  -- 부정 리뷰 20% 초과
ORDER BY negative_pct DESC
```
**관련 테이블**: `naver_store_review_item`, `naver_store_product_qna`

---

### ⚙️ CS 프로세스 효율화

#### Q: "프로세스를 효율화해야 할 부분은? / 반복적으로 발생하는 동일 유형 문의는?"
```sql
-- 자동화/FAQ 후보: 동일 유형 문의 Top 10
SELECT
  keyword_group,
  COUNT(*) as frequency,
  AVG(DATEDIFF(ah.resolve_date, a.regist_date)) as avg_resolve_days,
  -- 자동화 가능 여부 판정
  CASE
    WHEN keyword_group IN ('사용법', '설정방법', '호환성문의') THEN 'FAQ/챗봇 자동화 가능'
    WHEN keyword_group IN ('배송조회', '교환접수', '환불접수') THEN 'RPA 자동화 가능'
    WHEN keyword_group IN ('부품요청') THEN '부품 자동발송 시스템 가능'
    ELSE '수동 대응 유지'
  END as automation_potential,
  -- 절감 효과 추정
  CASE
    WHEN keyword_group IN ('사용법', '설정방법') THEN CONCAT('월 ', COUNT(*), '건 × 15분 = ', ROUND(COUNT(*) * 15 / 60), '시간 절감 가능')
    ELSE '-'
  END as efficiency_gain
FROM (
  SELECT a.seq,
    CASE
      WHEN am.memo_content REGEXP '사용법|모르|어떻게|방법' THEN '사용법'
      WHEN am.memo_content REGEXP '설정|세팅|연결|페어링' THEN '설정방법'
      WHEN am.memo_content REGEXP '호환|맞|사용가능' THEN '호환성문의'
      WHEN am.memo_content REGEXP '배송|택배|언제' THEN '배송조회'
      WHEN am.memo_content REGEXP '교환|바꿔' THEN '교환접수'
      WHEN am.memo_content REGEXP '환불|취소|반품' THEN '환불접수'
      WHEN am.memo_content REGEXP '부품|키캡|케이블|어댑터' THEN '부품요청'
      WHEN am.memo_content REGEXP '불량|고장|안됨|작동' THEN 'AS접수'
      ELSE '기타'
    END as keyword_group
  FROM crm_as a
  LEFT JOIN crm_as_memo am ON a.seq = am.as_seq
  WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
) categorized
LEFT JOIN crm_as a ON categorized.seq = a.seq
LEFT JOIN crm_as_history ah ON a.seq = ah.as_seq
GROUP BY keyword_group
ORDER BY frequency DESC
```
**파생 질문**: "CS 담당자별 처리 건수와 평균 해결 시간은?"
```sql
SELECT m.name as cs_agent,
  COUNT(*) as total_handled,
  AVG(DATEDIFF(ah.resolve_date, a.regist_date)) as avg_resolve_days,
  SUM(CASE WHEN ah.status = '완료' THEN 1 ELSE 0 END) as resolved,
  SUM(CASE WHEN ah.status != '완료' THEN 1 ELSE 0 END) as pending
FROM crm_as a
JOIN member m ON a.regist_id = m.uid
LEFT JOIN crm_as_history ah ON a.seq = ah.as_seq
WHERE a.regist_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
GROUP BY a.regist_id
ORDER BY total_handled DESC
```

#### Q: "고객 불만을 효율적으로 해결하려면? (재구매율 영향 분석)"
```sql
-- AS 경험이 재구매에 미치는 영향
SELECT
  as_experience,
  COUNT(DISTINCT user_name) as customers,
  AVG(repurchase_flag) as repurchase_rate,
  AVG(avg_satisfaction) as avg_satisfaction
FROM (
  SELECT s.user_name,
    CASE
      WHEN as_count > 0 AND resolve_days <= 3 THEN '빠른 AS 해결 (3일 내)'
      WHEN as_count > 0 AND resolve_days <= 7 THEN 'AS 해결 (7일 내)'
      WHEN as_count > 0 AND resolve_days > 7 THEN '느린 AS 해결 (7일+)'
      WHEN as_count > 0 THEN 'AS 미해결'
      ELSE 'AS 경험 없음'
    END as as_experience,
    CASE WHEN COUNT(DISTINCT order_date) > 1 THEN 1 ELSE 0 END as repurchase_flag
  FROM tbnws_sabangnet_order s
  LEFT JOIN (SELECT ap.product_code, COUNT(*) as as_count,
               AVG(DATEDIFF(ah.resolve_date, a.regist_date)) as resolve_days
             FROM crm_as a
             JOIN crm_as_product ap ON a.seq = ap.as_seq
             LEFT JOIN crm_as_history ah ON a.seq = ah.as_seq
             GROUP BY ap.product_code) as_info ON s.model_no = as_info.product_code
  WHERE s.order_status NOT IN ('취소')
  GROUP BY s.user_name
) customer_analysis
GROUP BY as_experience
ORDER BY repurchase_rate DESC
```
**인사이트**: AS 3일 내 해결 고객의 재구매율 vs 미해결 고객 재구매율 비교
→ 빠른 해결이 재구매율에 미치는 영향 정량화

#### Q: "네이버 Q&A와 고객문의에서 자주 묻는 질문 패턴은?"
```sql
-- 네이버 상품 Q&A + 고객문의 통합 분석
SELECT
  question_type,
  COUNT(*) as frequency,
  CASE
    WHEN frequency > 20 THEN 'FAQ 등록 시급'
    WHEN frequency > 10 THEN 'FAQ 등록 권장'
    ELSE '모니터링'
  END as action
FROM (
  SELECT
    CASE
      WHEN content REGEXP '호환|맞|사용가능|되나요' THEN '호환성 문의'
      WHEN content REGEXP '배송|언제|도착' THEN '배송 문의'
      WHEN content REGEXP '차이|비교|추천' THEN '제품 비교/추천'
      WHEN content REGEXP '보증|워런티|수리' THEN '보증/수리 문의'
      WHEN content REGEXP 'A/S|AS|서비스' THEN 'AS 절차 문의'
      WHEN content REGEXP '할인|쿠폰|세일' THEN '프로모션 문의'
      ELSE '기타'
    END as question_type
  FROM naver_store_product_qna
  WHERE regist_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
  UNION ALL
  SELECT
    CASE
      WHEN content REGEXP '호환|맞|사용가능' THEN '호환성 문의'
      WHEN content REGEXP '배송|언제' THEN '배송 문의'
      ELSE '기타'
    END as question_type
  FROM naver_store_customer_inquiries
  WHERE regist_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
) all_inquiries
GROUP BY question_type
ORDER BY frequency DESC
```

---

## 6. 수출입 담당

### 🔎 공급망

#### Q: "발주 넣어야 할 품목은? / 긴급 발주 필요한 품목은?"
```
-- orderForecast 대시보드 데이터 직접 활용
-- 현재: 긴급 246개, 주의 87개
-- 가중치: 1주(50%) + 1개월(30%) + 3개월(20%)
-- 안전재고 = max(14일, LT * 0.5) 판매량
-- 긴급 = 가용재고 < 안전재고 AND 발주 미진행
```
**관련 테이블**: `erp_stock`, `erp_product_info`, `erp_order` + `erp_order_item`
**계산 로직**: orderForecast 페이지 참조 (커버리지 2.3x, 리드타임 기반)

#### Q: "리드타임 길어진 거래처는?"
```sql
-- 실제 입고일 - 발주일 비교
SELECT
  p.partner_name, p.partner_code,
  AVG(CASE WHEN o.order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
    THEN DATEDIFF(o.import_date, o.order_date) END) as recent_lt,
  AVG(CASE WHEN o.order_date < DATE_SUB(NOW(), INTERVAL 3 MONTH)
    THEN DATEDIFF(o.import_date, o.order_date) END) as older_lt
FROM erp_order o
JOIN erp_partner p ON o.partner_code = p.partner_code
WHERE o.import_date IS NOT NULL
GROUP BY p.partner_code
HAVING recent_lt > older_lt * 1.2  -- 20% 이상 증가
```
**관련 테이블**: `erp_order`, `erp_partner`
**키 컬럼**: `order_date`(발주일), `import_date`(입고일), `partner_code`(공급처)

---

### ⚠️ 리스크

#### Q: "환율 변동 손실 가능성은?"
```sql
-- 발주 시점 환율 vs 현재 환율 비교
SELECT
  o.order_seq, o.order_title,
  oi.currency, oi.cur2krw as order_rate,
  ce.exchange_rate as current_rate,
  oi.buying_price * oi.ea * (ce.exchange_rate - oi.cur2krw) as exchange_loss
FROM erp_order o
JOIN erp_order_item oi ON o.order_seq = oi.order_seq
JOIN currency_exchange_rate ce ON oi.currency = ce.currency_code
WHERE o.arr_flag != 'Y'  -- 아직 미도착
  AND ce.rate_date = (SELECT MAX(rate_date) FROM currency_exchange_rate)
  AND exchange_loss > 0
ORDER BY exchange_loss DESC
```
**관련 테이블**: `erp_order_item`, `currency_exchange_rate`
**키 컬럼**: `cur2krw`(발주 시 환율), `buying_price`(외화 매입가)

#### Q: "재고 소진 시점은?"
```sql
-- orderForecast 로직: 가용재고 / 일평균 판매량
SELECT p.product_name,
  stock.available,
  sales.daily_avg,
  ROUND(stock.available / sales.daily_avg) as days_remaining,
  DATE_ADD(NOW(), INTERVAL ROUND(stock.available / sales.daily_avg) DAY) as depletion_date
FROM erp_product_info p
JOIN (가용재고 서브쿼리) stock
JOIN (일평균 판매량 서브쿼리) sales
WHERE p.sell_status = 'Y'
ORDER BY days_remaining ASC
```
**관련 테이블**: `erp_stock`, `tbnws_sabangnet_order`, `erp_product_info`

---

### 🎯 전략

#### Q: "공급처 다변화 필요한 제품은?"
```sql
-- 단일 공급처 의존 상품
SELECT p.product_name,
  COUNT(DISTINCT o.partner_code) as supplier_count,
  GROUP_CONCAT(DISTINCT pt.partner_name) as suppliers,
  SUM(oi.ea) as total_qty
FROM erp_order_item oi
JOIN erp_order o ON oi.order_seq = o.order_seq
JOIN erp_partner pt ON o.partner_code = pt.partner_code
JOIN erp_product_info p ON oi.product_code = p.product_code
WHERE o.order_date >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
GROUP BY oi.product_code
HAVING supplier_count = 1 AND total_qty > 500  -- 단일 공급처 + 대량
```
**관련 테이블**: `erp_order`, `erp_order_item`, `erp_partner`, `erp_product_info`

---

### 🔥 리드타임 & 시즌 변수 반영

#### Q: "춘절 직전인데, PO 기준으로 실제 입고 예정일이 언제로 밀릴 가능성이 있어?"
```sql
-- 미도착 PO 중 중국 공급처 발주건 + 춘절 기간 겹침 여부
SELECT o.order_seq, o.order_title, pt.partner_name,
  o.order_date, o.import_date as planned_arrival,
  DATEDIFF(o.import_date, o.order_date) as planned_lt,
  -- 춘절 기간(1월말~2월초) 겹침 시 +14일 지연 추정
  CASE
    WHEN o.import_date BETWEEN '2026-01-25' AND '2026-02-15'
      THEN DATE_ADD(o.import_date, INTERVAL 14 DAY)
    WHEN o.import_date BETWEEN '2026-02-16' AND '2026-02-28'
      THEN DATE_ADD(o.import_date, INTERVAL 7 DAY)
    ELSE o.import_date
  END as estimated_arrival,
  GROUP_CONCAT(DISTINCT pi.product_name) as products
FROM erp_order o
JOIN erp_partner pt ON o.partner_code = pt.partner_code
JOIN erp_order_item oi ON o.order_seq = oi.order_seq
JOIN erp_product_info pi ON oi.product_code = pi.product_code
WHERE o.arr_flag != 'Y' AND o.dep_flag != 'Y'  -- 미출항/미도착
GROUP BY o.order_seq
ORDER BY o.import_date ASC
```
**관련 테이블**: `erp_order`, `erp_partner`, `erp_order_item`
**핵심 로직**: 공급처 국가별 공휴일 캘린더 + 과거 같은 시즌 실제 LT 편차 참조
**필요 보강**: 공급처별 국가 컬럼(erp_partner), 시즌 지연 이력 테이블

#### Q: "춘절 + 선적 지연 2주 가정 시, 재고 끊길 SKU는?"
```sql
-- 현재 가용재고 vs (지연된 입고일까지의 예상 소진량)
SELECT pi.product_name, pi.product_code,
  stock.available_ea,
  sales.daily_avg,
  ROUND(stock.available_ea / sales.daily_avg) as days_coverage,
  delayed.estimated_arrival,
  DATEDIFF(delayed.estimated_arrival, NOW()) as days_until_arrival,
  CASE
    WHEN stock.available_ea / sales.daily_avg < DATEDIFF(delayed.estimated_arrival, NOW())
    THEN 'OOS 예상'
    ELSE '커버 가능'
  END as risk_status,
  DATE_ADD(NOW(), INTERVAL ROUND(stock.available_ea / sales.daily_avg) DAY) as stockout_date
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
JOIN (SELECT model_no, SUM(p_ea)/DATEDIFF(NOW(), DATE_SUB(NOW(), INTERVAL 30 DAY)) as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales
  ON pi.product_code = sales.model_no
LEFT JOIN (SELECT oi.product_code, DATE_ADD(MAX(o.import_date), INTERVAL 14 DAY) as estimated_arrival
           FROM erp_order o JOIN erp_order_item oi ON o.order_seq = oi.order_seq
           WHERE o.arr_flag != 'Y' GROUP BY oi.product_code) delayed
  ON pi.product_code = delayed.product_code
WHERE stock.available_ea / sales.daily_avg < DATEDIFF(delayed.estimated_arrival, NOW())
ORDER BY days_coverage ASC
```
**관련 테이블**: `erp_stock`, `tbnws_sabangnet_order`, `erp_order`, `erp_order_item`
**핵심 계산**: 가용재고일수 < 지연 입고까지 남은 일수 → OOS 위험

#### Q: "항공 전환 시 비용 대비 손실 방어 가능한 SKU는?"
```sql
-- 항공 전환으로 절약되는 일수 × 일일 매출이익 > 항공 추가 운임
-- 해상→항공 전환 시 LT 단축 가정: 30일 → 7일 (23일 절약)
SELECT pi.product_name, pi.product_code,
  sales.daily_revenue,
  sales.daily_margin,
  23 as days_saved,  -- 해상→항공 전환 시
  sales.daily_margin * 23 as saved_margin,
  oi.ea * 5 as estimated_air_premium,  -- 개당 $5 항공 추가 가정
  CASE
    WHEN sales.daily_margin * 23 > oi.ea * 5 * oi.cur2krw
    THEN '항공 전환 유리'
    ELSE '해상 유지'
  END as recommendation
FROM erp_order_item oi
JOIN erp_order o ON oi.order_seq = o.order_seq
JOIN erp_product_info pi ON oi.product_code = pi.product_code
JOIN (SELECT model_no,
        SUM(pay_cost)/30 as daily_revenue,
        SUM(pay_cost - p_ea * pi2.purchase_price)/30 as daily_margin
      FROM tbnws_sabangnet_order s
      JOIN erp_product_info pi2 ON s.model_no = pi2.product_code
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE o.arr_flag != 'Y' AND o.dep_flag != 'Y'
ORDER BY saved_margin DESC
```
**핵심 판단**: 절약 마진 > 항공 추가비용이면 전환 유리

#### Q: "리드타임 변동성이 가장 큰 거래처는?"
```sql
SELECT pt.partner_name, pt.partner_code,
  COUNT(o.order_seq) as total_orders,
  ROUND(AVG(DATEDIFF(o.import_date, o.order_date))) as avg_lt,
  ROUND(MIN(DATEDIFF(o.import_date, o.order_date))) as min_lt,
  ROUND(MAX(DATEDIFF(o.import_date, o.order_date))) as max_lt,
  ROUND(STDDEV(DATEDIFF(o.import_date, o.order_date)), 1) as lt_stddev,
  ROUND(STDDEV(DATEDIFF(o.import_date, o.order_date))
    / AVG(DATEDIFF(o.import_date, o.order_date)) * 100, 1) as cv_pct
FROM erp_order o
JOIN erp_partner pt ON o.partner_code = pt.partner_code
WHERE o.import_date IS NOT NULL AND o.order_date IS NOT NULL
  AND o.order_date >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
GROUP BY pt.partner_code
HAVING total_orders >= 3
ORDER BY lt_stddev DESC
```
**관련 테이블**: `erp_order`, `erp_partner`
**핵심 지표**: CV%(변동계수) = 표준편차/평균 × 100 → 높을수록 불안정

---

### 📦 재고 단절 이후 수요 왜곡 대응

#### Q: "지난달 OOS로 판매 끊겼던 SKU 중, 입고 후 리바운드 예상 물량은?"
```sql
-- Step 1: OOS 기간 식별 (일별 가용재고 = 0인 구간)
-- Step 2: OOS 직전 판매속도 기준 리바운드 예상
SELECT pi.product_name, pi.product_code,
  pre_oos.daily_avg as pre_oos_daily_sales,
  stock.available_ea as current_stock,
  oos.oos_days,
  -- 리바운드 공식: OOS일수 × 직전 일평균 × 1.3 (보복소비 계수)
  ROUND(oos.oos_days * pre_oos.daily_avg * 1.3) as rebound_estimate_4wk
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
JOIN (-- OOS 기간: 가용재고 0이었던 날수 (최근 60일)
      SELECT product_code,
        SUM(CASE WHEN daily_stock = 0 THEN 1 ELSE 0 END) as oos_days
      FROM daily_stock_snapshot  -- 일별 스냅샷 필요
      WHERE snapshot_date >= DATE_SUB(NOW(), INTERVAL 60 DAY)
      GROUP BY product_code
      HAVING oos_days >= 7) oos ON pi.product_code = oos.product_code
JOIN (-- OOS 직전 30일 판매속도
      SELECT model_no, SUM(p_ea)/30 as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date BETWEEN DATE_SUB(NOW(), INTERVAL 90 DAY)
                          AND DATE_SUB(NOW(), INTERVAL 60 DAY)
      GROUP BY model_no) pre_oos ON pi.product_code = pre_oos.model_no
ORDER BY rebound_estimate_4wk DESC
```
**핵심 로직**: OOS 직전 판매속도 × OOS 일수 × 보복소비 계수(1.3)
**필요 보강**: `daily_stock_snapshot` 테이블 또는 erp_stock에서 일별 가용재고 재구성

#### Q: "품절 기간 동안 유입은 있었는데 구매 못한 수요 추정치는?"
```
-- 직접 데이터: tbe.kr 또는 GA(Google Analytics) 트래픽 데이터 필요
-- 간접 추정: OOS 직전 전환율 × OOS 기간 유입수
-- 대안 로직: OOS 직전 일평균 판매량 × OOS 일수 = 누적 미충족 수요
-- 예: 일평균 15개 × OOS 30일 = 450개 미충족 수요
```
**데이터 소스**: tbe.kr (유입 트래픽) + `tbnws_sabangnet_order` (전환율 역산)
**간접 추정**: OOS 직전 30일 일평균 × OOS 일수

#### Q: "입고 직후 4주간 소화속도는 정상 대비 몇 % 증가 예상?"
```sql
-- 과거 OOS→재입고 사례에서 4주 소화속도 패턴 분석
SELECT pi.product_name,
  normal.weekly_avg as normal_weekly,
  post_restock.weekly_avg as post_restock_weekly,
  ROUND((post_restock.weekly_avg / normal.weekly_avg - 1) * 100, 1) as surge_pct
FROM erp_product_info pi
JOIN (-- 정상 기간 주간 판매
      SELECT model_no, SUM(p_ea) / COUNT(DISTINCT YEARWEEK(order_date)) as weekly_avg
      FROM tbnws_sabangnet_order
      WHERE order_status NOT IN ('취소')
        AND order_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
      GROUP BY model_no) normal ON pi.product_code = normal.model_no
JOIN (-- 재입고 직후 4주 판매 (입고 이벤트 식별 필요)
      SELECT s.product_code as model_no,
        SUM(so.p_ea) / 4 as weekly_avg
      FROM erp_stock s
      JOIN tbnws_sabangnet_order so ON s.product_code = so.model_no
        AND so.order_date BETWEEN s.import_date AND DATE_ADD(s.import_date, INTERVAL 28 DAY)
      WHERE s.import_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
      GROUP BY s.product_code) post_restock ON pi.product_code = post_restock.model_no
WHERE post_restock.weekly_avg > normal.weekly_avg
ORDER BY surge_pct DESC
```
**핵심 지표**: 재입고 후 4주 주간 평균 / 정상 주간 평균 × 100

#### Q: "품절 후 재입고된 모델 중 과발주 위험 있는 SKU는?"
```sql
-- 과거 패턴: OOS 후 과잉 발주 → 6개월 후 재고 과잉된 사례
SELECT pi.product_name,
  oi.ea as ordered_qty,
  sales.monthly_avg * 6 as six_month_demand,
  ROUND(oi.ea / (sales.monthly_avg * 6) * 100) as order_vs_demand_pct,
  CASE
    WHEN oi.ea > sales.monthly_avg * 9 THEN '과발주 위험 높음'
    WHEN oi.ea > sales.monthly_avg * 6 THEN '과발주 주의'
    ELSE '적정'
  END as risk_level
FROM erp_order_item oi
JOIN erp_order o ON oi.order_seq = o.order_seq
JOIN erp_product_info pi ON oi.product_code = pi.product_code
JOIN (SELECT model_no, SUM(p_ea)/6 as monthly_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
        AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE o.arr_flag != 'Y'
  AND oi.ea > sales.monthly_avg * 6
ORDER BY order_vs_demand_pct DESC
```
**핵심 판단**: 발주량 > 6개월 수요의 150% → 과발주 위험

---

### 🔄 카니발라이제이션(Cannibalization) 분석

#### Q: "A 모델 품절 시 B 모델 매출 상승률은?"
```sql
-- 같은 goods_code(상품군) 내 다른 option의 대체 효과 분석
-- Step 1: A모델 OOS 기간 식별
-- Step 2: 같은 기간 B모델(동일 goods_code) 판매량 변화 측정
SELECT
  a_model.product_name as model_a,
  b_model.product_name as model_b,
  b_normal.weekly_avg as b_normal_sales,
  b_during_oos.weekly_avg as b_oos_period_sales,
  ROUND((b_during_oos.weekly_avg / b_normal.weekly_avg - 1) * 100, 1) as cannibalization_pct
FROM erp_product_info a_model
JOIN erp_product_info b_model
  ON a_model.goods_code = b_model.goods_code
  AND a_model.brand_code = b_model.brand_code
  AND a_model.product_code != b_model.product_code
-- B모델 정상 기간 vs A모델 OOS 기간 판매 비교
CROSS JOIN LATERAL (...) b_normal
CROSS JOIN LATERAL (...) b_during_oos
WHERE cannibalization_pct > 10
ORDER BY cannibalization_pct DESC
```
**핵심 로직**: 동일 goods_code 내 옵션간 대체율 측정
**필요 데이터**: 일별 SKU별 재고 유무 + 판매량 시계열

#### Q: "A/B 동시 보유 재고가 최소 몇 주 유지되어야 카니발 최소화?"
```
-- 분석 방법:
-- 1) A+B 동시 재고 보유 기간의 각각 판매량 측정
-- 2) A만 보유, B만 보유 기간의 판매량 측정
-- 3) 합산 매출 최대화 되는 최소 동시 재고 기간 산출
-- 경험적 기준: 동일 상품군 내 2개 이상 옵션은 최소 4주 동시 재고 권장
```
**산출 방식**: A+B 합산 매출이 정상 대비 95% 이상 유지되는 최소 재고 기간

#### Q: "A 재고 충분, B만 부족하면 매출 손실 추정치는?"
```sql
-- B 단독 매출 + A←B 카니발 효과 역산
SELECT
  pi_b.product_name as model_b,
  b_sales.daily_avg as b_daily_sales,
  b_sales.daily_revenue as b_daily_revenue,
  -- 카니발 역효과: B 품절 시 A로 대체되지 않는 비율 (이탈율)
  ROUND(b_sales.daily_revenue * 0.3, 0) as estimated_daily_loss,
  -- 가정: 고객의 30%는 대체 구매 안 함 (이탈)
  ROUND(b_sales.daily_revenue * 0.3 * 30, 0) as monthly_loss_estimate
FROM erp_product_info pi_b
JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg, SUM(pay_cost)/30 as daily_revenue
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
      GROUP BY model_no) b_sales ON pi_b.product_code = b_sales.model_no
WHERE pi_b.product_code IN (/* B모델 코드 */)
```
**핵심 가정**: 대체 불가 이탈율 약 30% (카테고리별 상이)

#### Q: "두 모델 합산 기준 안전재고는?"
```sql
-- 합산 안전재고 = 합산 일평균 × 안전일수 × 변동계수 보정
SELECT
  pi.goods_code, g.goods_name,
  SUM(sales.daily_avg) as combined_daily,
  ROUND(SUM(sales.daily_avg) * 14 * 1.2) as combined_safety_stock,
  SUM(stock.available_ea) as combined_current_stock
FROM erp_product_info pi
JOIN erp_goods g ON pi.goods_code = g.goods_code AND pi.brand_code = g.brand_code
JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL
      GROUP BY product_code) stock ON pi.product_code = stock.product_code
GROUP BY pi.goods_code
```
**안전재고 공식**: 합산 일평균 × 14일 × 1.2(변동계수)

---

### 📊 발주 타이밍 시뮬레이션

#### Q: "이번 주 발주 vs 다음 주 발주 시 재고 회전일 차이는?"
```sql
-- 시뮬레이션: 발주 시점에 따른 입고 시점 차이 → 재고 커버리지 영향
SELECT pi.product_name, pi.product_code,
  stock.available_ea,
  sales.daily_avg,
  ROUND(stock.available_ea / sales.daily_avg) as current_days_coverage,
  -- 이번 주 발주: 현재 LT 후 입고
  avg_lt.days as avg_lead_time,
  ROUND((stock.available_ea + incoming.ea) / sales.daily_avg) as coverage_if_order_now,
  -- 다음 주 발주: LT + 7일 후 입고
  ROUND((stock.available_ea - sales.daily_avg * 7 + incoming.ea) / sales.daily_avg) as coverage_if_order_next_week
FROM erp_product_info pi
JOIN (/*가용재고*/) stock ON pi.product_code = stock.product_code
JOIN (/*일평균판매*/) sales ON pi.product_code = sales.model_no
JOIN (/*평균LT*/) avg_lt ON pi.product_code = avg_lt.product_code
JOIN (/*발주예정수량*/) incoming ON pi.product_code = incoming.product_code
ORDER BY current_days_coverage ASC
```

#### Q: "MOQ 충족 위해 묶어야 할 SKU는?"
```sql
-- 동일 공급처(partner_code) 내 발주 필요 SKU 그룹핑
SELECT pt.partner_name,
  GROUP_CONCAT(pi.product_name ORDER BY urgency DESC SEPARATOR ', ') as bundle_skus,
  SUM(needed.qty) as total_qty,
  SUM(needed.qty * oi.buying_price) as total_amount
FROM erp_product_info pi
JOIN erp_order_item oi ON pi.product_code = oi.product_code
JOIN erp_order o ON oi.order_seq = o.order_seq
JOIN erp_partner pt ON o.partner_code = pt.partner_code
JOIN (-- 발주 필요 수량: 안전재고 - 가용재고
      SELECT product_code,
        GREATEST(pi2.safe_ea - COALESCE(s.available, 0), 0) as qty,
        CASE WHEN s.available < pi2.safe_ea * 0.5 THEN 'HIGH' ELSE 'MEDIUM' END as urgency
      FROM erp_product_info pi2
      LEFT JOIN (SELECT product_code, COUNT(*) as available
                 FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) s
        ON pi2.product_code = s.product_code
      WHERE pi2.sell_status = 'Y') needed ON pi.product_code = needed.product_code
WHERE needed.qty > 0
GROUP BY pt.partner_code
ORDER BY total_amount DESC
```
**핵심 로직**: 공급처별 묶음 → MOQ 달성 + 운송비 최적화

#### Q: "환율 상승 5% 가정 시 지금 발주가 유리해?"
```sql
SELECT pi.product_name,
  oi.buying_price, oi.currency, oi.ea,
  ce.exchange_rate as current_rate,
  ce.exchange_rate * 1.05 as projected_rate,
  oi.buying_price * oi.ea * ce.exchange_rate as cost_now,
  oi.buying_price * oi.ea * ce.exchange_rate * 1.05 as cost_later,
  ROUND(oi.buying_price * oi.ea * ce.exchange_rate * 0.05) as additional_cost,
  -- 대기 비용 vs 환율 추가비용 비교
  CASE
    WHEN oi.buying_price * oi.ea * ce.exchange_rate * 0.05 > stock_holding_cost
    THEN '선발주 유리'
    ELSE '대기 가능'
  END as recommendation
FROM erp_order_item oi
JOIN erp_product_info pi ON oi.product_code = pi.product_code
JOIN currency_exchange_rate ce ON oi.currency = ce.currency_code
  AND ce.rate_date = (SELECT MAX(rate_date) FROM currency_exchange_rate)
WHERE oi.currency != 'KRW'
ORDER BY additional_cost DESC
```
**관련 테이블**: `erp_order_item`, `currency_exchange_rate`, `erp_product_info`

---

### 🚨 위기 감지 고급

#### Q: "현재 재고 기준 3주 내 매출 타격 가능 SKU는?"
```sql
SELECT pi.product_name, pi.product_code,
  stock.available_ea,
  sales.daily_avg,
  ROUND(stock.available_ea / sales.daily_avg) as days_remaining,
  ROUND(sales.daily_avg * pi.msrp) as daily_revenue_at_risk,
  -- 입고 예정 여부
  COALESCE(incoming.expected_date, '미정') as next_incoming
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
LEFT JOIN (SELECT oi.product_code, MIN(o.import_date) as expected_date
           FROM erp_order o JOIN erp_order_item oi ON o.order_seq = oi.order_seq
           WHERE o.arr_flag != 'Y' GROUP BY oi.product_code) incoming
  ON pi.product_code = incoming.product_code
WHERE stock.available_ea / sales.daily_avg <= 21  -- 3주 이내
  AND sales.daily_avg > 0
ORDER BY days_remaining ASC
```

#### Q: "안전재고 계산이 실제 판매속도와 가장 괴리 큰 SKU는?"
```sql
SELECT pi.product_name, pi.product_code,
  pi.safe_ea as set_safety_stock,
  ROUND(sales.daily_avg * 14 * 1.2) as calculated_safety_stock,
  ABS(pi.safe_ea - ROUND(sales.daily_avg * 14 * 1.2)) as gap,
  CASE
    WHEN pi.safe_ea > sales.daily_avg * 14 * 2 THEN '과다 설정'
    WHEN pi.safe_ea < sales.daily_avg * 7 THEN '과소 설정'
    ELSE '적정'
  END as assessment
FROM erp_product_info pi
JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
WHERE pi.safe_ea > 0 AND sales.daily_avg > 0
ORDER BY gap DESC
```
**핵심**: `erp_product_info.safe_ea` 설정값 vs 실제 판매속도 기반 계산값 비교

#### Q: "최근 3개월 평균 대비 재고회전 급격히 느려진 SKU는?"
```sql
SELECT pi.product_name,
  recent.turnover_days as recent_90d_turnover,
  older.turnover_days as prev_90d_turnover,
  ROUND((recent.turnover_days / older.turnover_days - 1) * 100, 1) as slowdown_pct
FROM erp_product_info pi
JOIN (SELECT model_no,
        AVG(stock_ea) / NULLIF(SUM(p_ea)/90, 0) as turnover_days
      FROM tbnws_sabangnet_order s
      -- 최근 90일 회전일
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
      GROUP BY model_no) recent ON pi.product_code = recent.model_no
JOIN (SELECT model_no,
        AVG(stock_ea) / NULLIF(SUM(p_ea)/90, 0) as turnover_days
      FROM tbnws_sabangnet_order s
      WHERE order_date BETWEEN DATE_SUB(NOW(), INTERVAL 180 DAY)
                          AND DATE_SUB(NOW(), INTERVAL 90 DAY)
      GROUP BY model_no) older ON pi.product_code = older.model_no
WHERE recent.turnover_days > older.turnover_days * 1.5  -- 50% 이상 느려짐
ORDER BY slowdown_pct DESC
```
**핵심 지표**: 재고회전일수 = 평균재고 / 일평균판매

---

## 7. 마케팅 담당

### 🔎 퍼포먼스

#### Q: "ROAS 급락 채널은?"
```
-- tbe.kr 별도 DB 데이터
-- 현재 tbe.kr에서 확인 가능한 지표:
-- 구매 ROAS: 2,262%
-- SA 광고비: 448,481원 → SA전환: 49,791,900원
-- GFA 광고비: 4,782,108원 → GFA전환: 68,503,990원
-- 일별 ROAS 추이 차트 참조
```
**데이터 소스**: tbe.kr 별도 DB (네이버 광고 API)
**보조 데이터**: `naver_search_ad` (TBNWS_ADMIN DB)

#### Q: "광고 의존도 높은 제품은?"
```
-- tbe.kr 전환매출 vs admin DB 실제매출 비교
-- tbe.kr: 전환매출 118,295,890원
-- admin: 스마트스토어 실제매출 190,578,230원
-- 비율: 전환매출/실제매출 = 62% (나머지 38%가 자연유입)
-- 비율 높은 상품 = 광고 의존도 높음
```
**데이터 소스**: tbe.kr (광고 전환) + `tbnws_sabangnet_order` (실제 매출)

#### Q: "광고비 20% 증액 시 매출 예측?"
```
-- tbe.kr 기간별 데이터 기반 한계 ROAS 분석
-- 현재 ROAS 2,262% → 광고비 5.2M → 매출 118M
-- 한계 ROAS = 추가 광고비 대비 추가 전환율
-- 키워드 스코어링 데이터 (tbe.kr 스코어링 메뉴) 참조
```
**데이터 소스**: tbe.kr 스코어링/통합분석

---

### 📊 광고 효율 이상 감지 (고급)

#### Q: "ROAS 20% 이상 하락 캠페인은?"
```
-- tbe.kr 별도 DB: 캠페인/키워드별 일별 ROAS 추이
-- 비교 기간: 최근 7일 vs 이전 7일
-- ROAS = 전환매출 / 광고비 × 100
-- 하락 기준: (최근ROAS - 이전ROAS) / 이전ROAS < -0.2
```
**데이터 소스**: tbe.kr DB (SA/GFA 키워드별 소진금액, 전환매출)
**보조**: `naver_search_ad` (TBNWS_ADMIN DB)

#### Q: "클릭률은 유지되는데 전환율만 급락한 SKU는?"
```
-- tbe.kr 데이터: CTR(클릭률) vs CVR(전환율) 분리 분석
-- CTR 변동 < 5% AND CVR 변동 > -20% = 상품페이지/가격/재고 문제
-- 원인 추정 로직:
--   1) 재고 부족(OOS) → erp_stock 확인
--   2) 가격 인상 → erp_product_info.msrp 변경 이력
--   3) 리뷰 하락 → naver_store_review_item 평점 추이
--   4) 경쟁사 프로모션 → 외부 데이터 필요
```
**크로스 참조**: `erp_stock`(재고), `naver_store_review_item`(리뷰), `erp_product_info`(가격)

#### Q: "CAC 상승 추세 4주 이상 지속 채널은?"
```
-- CAC = 광고비 / 신규 고객 수
-- tbe.kr DB: 채널별 주간 CAC 추이
-- 4주 연속 상승 = 채널 효율 저하 신호
-- 판단: CAC > LTV(고객생애가치)면 채널 축소 검토
```
**파생 질문**: "CAC가 첫 구매 마진을 초과하는 채널은?"
```sql
-- 첫 구매 평균 마진 vs CAC 비교
SELECT mall_id,
  AVG(s.pay_cost) as avg_order_value,
  AVG(s.pay_cost) * 0.3 as estimated_first_margin,  -- 마진율 30% 가정
  -- tbe.kr에서 채널별 CAC 조회
  CASE
    WHEN estimated_first_margin < cac THEN 'CAC > 마진: 축소 검토'
    ELSE '유지'
  END as verdict
FROM tbnws_sabangnet_order s
WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
GROUP BY mall_id
```

#### Q: "광고비 줄여도 매출 유지 가능한 SKU는?"
```sql
-- 자연 유입 비중이 높은 SKU = 광고 의존도 낮음
-- 판단: 광고 전환매출 / 전체 매출 < 30% = 광고 축소 가능
SELECT pi.product_name,
  sales.total_revenue,
  -- tbe.kr에서 해당 SKU 광고 전환매출 조회
  ROUND(ad_conversion / sales.total_revenue * 100, 1) as ad_dependency_pct,
  CASE
    WHEN ad_dependency_pct < 20 THEN '광고 축소 가능'
    WHEN ad_dependency_pct < 40 THEN '점진적 축소 테스트'
    ELSE '광고 유지 필요'
  END as recommendation
FROM erp_product_info pi
JOIN (SELECT model_no, SUM(pay_cost) as total_revenue
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
ORDER BY ad_dependency_pct ASC
```
**파생 질문**: "브랜드 키워드 vs 일반 키워드 전환 비중은?"
→ 브랜드 키워드 비중 높으면 자연 인지도 강함, 광고 축소 가능

---

### 🧨 위험 감지 (고급)

#### Q: "브랜드 검색량 하락 시작 시점은?"
```
-- tbe.kr: 브랜드 키워드 일별 검색량 추이
-- TBNWS_ADMIN: naver_ranking_tracker_list 키워드 순위 추이
-- 하락 시작점 = 7일 이동평균이 30일 이동평균 아래로 교차하는 시점 (데드크로스)
```
**관련 테이블**: `naver_ranking_tracker_list`, tbe.kr DB
**파생 질문**: "검색량 하락이 매출 하락으로 전이되기까지 평균 며칠?"
→ 검색량 선행지표 분석 (보통 검색량 하락 → 7~14일 후 매출 하락)

#### Q: "광고 끊으면 즉시 매출 하락 예상 SKU는?"
```sql
-- 광고 의존도 70%+ = 광고 중단 시 즉시 타격
-- 자연 검색 순위 낮은 SKU = 광고 없으면 노출 자체가 안 됨
SELECT pi.product_name,
  nrt.current_rank,
  CASE
    WHEN nrt.current_rank > 50 AND ad_dependency > 0.7 THEN '즉시 타격'
    WHEN nrt.current_rank > 30 AND ad_dependency > 0.5 THEN '2주 내 타격'
    ELSE '버퍼 있음'
  END as risk_if_ad_stop
FROM erp_product_info pi
LEFT JOIN naver_ranking_tracker_list nrt ON pi.product_code = nrt.product_code
WHERE ad_dependency > 0.5
ORDER BY ad_dependency DESC
```
**파생 질문**: "광고 OFF 테스트를 해볼 만한 안전한 SKU는?"
→ 자연 검색 순위 10위 이내 + 리뷰 100개+ + 평점 4.5+

#### Q: "SNS 언급량 감소 제품은?"
```
-- 외부 데이터 필요: 소셜 모니터링 API (인스타그램, 유튜브, 블로그)
-- 간접 지표: 네이버 블로그/카페 검색량 추이 (tbe.kr or 네이버 API)
-- 내부 지표: naver_store_review_item 리뷰 작성 빈도 추이
```
**대안 측정**:
```sql
-- 리뷰 작성 빈도로 관심도 간접 측정
SELECT pi.product_name,
  COUNT(CASE WHEN r.review_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as recent_reviews,
  COUNT(CASE WHEN r.review_date BETWEEN DATE_SUB(NOW(), INTERVAL 60 DAY)
                                    AND DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as prev_reviews,
  ROUND((recent_reviews - prev_reviews) / NULLIF(prev_reviews, 0) * 100, 1) as review_trend_pct
FROM naver_store_review_item r
JOIN erp_product_info pi ON r.product_code = pi.product_code
GROUP BY pi.product_code
HAVING prev_reviews >= 5
ORDER BY review_trend_pct ASC
```

---

### 🎯 프로모션 설계 (고급)

#### Q: "할인 없이도 구매율 유지되는 SKU는?"
```sql
-- 정가 판매 비중 높고 판매량 안정적
SELECT pi.product_name,
  AVG(s.pay_cost / s.p_ea) as avg_selling_price,
  pi.msrp,
  ROUND((1 - AVG(s.pay_cost / s.p_ea) / pi.msrp) * 100, 1) as avg_discount_pct,
  COUNT(DISTINCT YEARWEEK(s.order_date)) as active_weeks,
  STDDEV(weekly_sales.ea) / AVG(weekly_sales.ea) as sales_stability_cv
FROM tbnws_sabangnet_order s
JOIN erp_product_info pi ON s.model_no = pi.product_code
JOIN (SELECT model_no, YEARWEEK(order_date) as wk, SUM(p_ea) as ea
      FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소')
      GROUP BY model_no, YEARWEEK(order_date)) weekly_sales
  ON s.model_no = weekly_sales.model_no
WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND s.order_status NOT IN ('취소')
GROUP BY pi.product_code
HAVING avg_discount_pct < 5  -- 평균 할인 5% 미만
  AND active_weeks >= 10  -- 10주 이상 꾸준히 판매
  AND sales_stability_cv < 0.3  -- 변동계수 30% 미만 (안정적)
ORDER BY sales_stability_cv ASC
```
**핵심**: 가격 비탄력적 SKU = 프리미엄 전략 유지 가능

#### Q: "1+1이 유리한 SKU vs %할인이 유리한 SKU는?"
```sql
-- 판단 기준:
-- 1+1 유리: 객단가 높음 + 재구매율 낮음 + 재고 과잉
-- %할인 유리: 객단가 낮음 + 가격 민감 채널 + 소량 재고
SELECT pi.product_name,
  pi.msrp,
  stock.available_ea,
  sales.avg_order_qty,
  CASE
    WHEN pi.msrp > 100000 AND stock.available_ea > sales.monthly_ea * 6
      THEN '1+1 권장 (재고 소진 + 체험 확대)'
    WHEN pi.msrp < 50000 AND stock.available_ea < sales.monthly_ea * 3
      THEN '% 할인 권장 (소량 재고, 빠른 회전)'
    WHEN pi.msrp BETWEEN 50000 AND 100000
      THEN '번들 세트 권장'
    ELSE '프로모션 불필요'
  END as promo_type
FROM erp_product_info pi
JOIN (/*재고*/) stock ON pi.product_code = stock.product_code
JOIN (/*판매 데이터*/) sales ON pi.product_code = sales.model_no
WHERE pi.sell_status = 'Y'
```
**파생 질문**: "세트 묶음 판매 시 객단가 상승 기대 SKU는?"
→ 보완재 관계(키보드+키캡, 레이싱휠+스탠드) 상품 조합 분석

#### Q: "쿠폰 적용 시 객단가 하락 위험 SKU는?"
```sql
-- 단가 낮은 상품에 정률 쿠폰 적용 시 객단가 역효과
SELECT pi.product_name, pi.msrp,
  CASE
    WHEN pi.msrp < 30000 THEN '쿠폰 적용 시 마진 잠식 위험'
    WHEN pi.msrp BETWEEN 30000 AND 80000 THEN '정액 쿠폰 권장'
    ELSE '정률 쿠폰 가능'
  END as coupon_strategy,
  -- 과거 쿠폰 사용 시 객단가 변화
  coupon_orders.avg_aov as coupon_aov,
  normal_orders.avg_aov as normal_aov
FROM erp_product_info pi
LEFT JOIN (SELECT model_no, AVG(pay_cost) as avg_aov
           FROM tbnws_sabangnet_order WHERE pay_cost < msrp * 0.9  -- 할인 주문
           GROUP BY model_no) coupon_orders ON pi.product_code = coupon_orders.model_no
LEFT JOIN (SELECT model_no, AVG(pay_cost) as avg_aov
           FROM tbnws_sabangnet_order WHERE pay_cost >= msrp * 0.9  -- 정가 주문
           GROUP BY model_no) normal_orders ON pi.product_code = normal_orders.model_no
```

---

### 🔮 시장 국면 판단 (고급)

#### Q: "현재는 성장기 / 교체기 / 수요 감소기 중 어디?"
```sql
-- 복합 지표 기반 시장 국면 판정
-- 1) 매출 추이 (4주 이동평균 방향)
-- 2) 신규 고객 비율 변화
-- 3) 객단가 변화
-- 4) 카테고리별 판매 분산도
SELECT
  'market_phase' as metric,
  CASE
    WHEN revenue_trend > 0 AND new_customer_ratio > 0.3 AND aov_trend > 0
      THEN '성장기 (Growing)'
    WHEN revenue_trend > 0 AND new_customer_ratio < 0.2 AND aov_trend < 0
      THEN '교체기 (Mature/Replacement)'
    WHEN revenue_trend < 0
      THEN '수요 감소기 (Declining)'
    ELSE '정체기 (Stagnant)'
  END as phase,
  revenue_trend, new_customer_ratio, aov_trend
FROM (
  SELECT
    (SUM(CASE WHEN order_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK) THEN pay_cost END)
     / SUM(CASE WHEN order_date BETWEEN DATE_SUB(NOW(), INTERVAL 8 WEEK)
                                    AND DATE_SUB(NOW(), INTERVAL 4 WEEK) THEN pay_cost END) - 1) as revenue_trend,
    COUNT(DISTINCT CASE WHEN first_order >= DATE_SUB(NOW(), INTERVAL 4 WEEK) THEN user_name END)
      / COUNT(DISTINCT user_name) as new_customer_ratio,
    (AVG(CASE WHEN order_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK) THEN pay_cost END)
     / AVG(CASE WHEN order_date < DATE_SUB(NOW(), INTERVAL 4 WEEK) THEN pay_cost END) - 1) as aov_trend
  FROM tbnws_sabangnet_order s
  LEFT JOIN (SELECT user_name, MIN(order_date) as first_order
             FROM tbnws_sabangnet_order GROUP BY user_name) fo ON s.user_name = fo.user_name
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 8 WEEK) AND order_status NOT IN ('취소')
)
```
**판정 매트릭스**:
| 매출추이 | 신규고객비율 | 객단가추이 | 국면 |
|---------|------------|----------|------|
| ↑ | 높음(30%+) | ↑ | 성장기 |
| ↑ | 낮음(<20%) | ↓ | 교체기 |
| ↓ | - | - | 수요 감소기 |
| → | → | → | 정체기 |

#### Q: "신제품 출시 효과 평균 유지 기간은?"
```sql
-- 신제품 출시 후 주간 판매량 피크 대비 50% 이하로 떨어지는 시점
SELECT pi.product_name,
  MIN(s.order_date) as launch_date,
  peak.peak_week,
  peak.peak_weekly_sales,
  decline.weeks_to_half as effect_duration_weeks
FROM erp_product_info pi
JOIN tbnws_sabangnet_order s ON pi.product_code = s.model_no
JOIN (-- 주간 피크 판매량
      SELECT model_no, YEARWEEK(order_date) as peak_week, SUM(p_ea) as peak_weekly_sales
      FROM tbnws_sabangnet_order WHERE order_status NOT IN ('취소')
      GROUP BY model_no, YEARWEEK(order_date)
      ORDER BY peak_weekly_sales DESC LIMIT 1) peak ON pi.product_code = peak.model_no
-- 피크 대비 50% 이하로 떨어진 최초 주차
CROSS JOIN LATERAL (
  SELECT COUNT(*) as weeks_to_half
  FROM (SELECT YEARWEEK(order_date) as wk, SUM(p_ea) as weekly_ea
        FROM tbnws_sabangnet_order
        WHERE model_no = pi.product_code AND order_status NOT IN ('취소')
        GROUP BY YEARWEEK(order_date)) w
  WHERE w.wk <= (SELECT MIN(wk) FROM (...) WHERE weekly_ea < peak.peak_weekly_sales * 0.5)
) decline
GROUP BY pi.product_code
```
**파생 질문**: "출시 3개월 후에도 성장 유지 중인 제품은?"
→ 롱런 히트 상품 식별 → 장기 재고 확보 전략

---

## 8. C레벨 통합 전략 질문

> 경영진이 요구하는 크로스 도메인 인사이트. 단일 테이블로 답할 수 없는 복합 분석.

### 💡 수요 본질 분석

#### Q: "지금 매출이 유지되는 이유가 '진짜 수요'야 아니면 '프로모션 착시'야?"
```sql
-- 복합 판정 로직:
-- 1) 할인 판매 비중 추이
-- 2) 정가 판매 수량 추이
-- 3) 광고 의존 매출 비중
-- 4) 자연 검색/유입 비중
SELECT
  '매출 품질 분석' as report,
  total.revenue as total_revenue,
  full_price.revenue as full_price_revenue,
  ROUND(full_price.revenue / total.revenue * 100, 1) as full_price_share_pct,
  discounted.revenue as discounted_revenue,
  ROUND(discounted.revenue / total.revenue * 100, 1) as discount_share_pct,
  -- 판정
  CASE
    WHEN full_price.revenue / total.revenue > 0.7 THEN '건강한 수요 (정가 70%+)'
    WHEN full_price.revenue / total.revenue > 0.5 THEN '혼합 (프로모션 의존 시작)'
    ELSE '프로모션 착시 위험 (할인 50%+)'
  END as demand_quality
FROM (SELECT SUM(pay_cost) as revenue FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND order_status NOT IN ('취소')) total
CROSS JOIN (-- 정가 판매 (실판매가 / MSRP > 0.95)
  SELECT SUM(s.pay_cost) as revenue
  FROM tbnws_sabangnet_order s
  JOIN erp_product_info pi ON s.model_no = pi.product_code
  WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND s.order_status NOT IN ('취소')
    AND s.pay_cost / s.p_ea >= pi.msrp * 0.95) full_price
CROSS JOIN (SELECT SUM(s.pay_cost) as revenue
  FROM tbnws_sabangnet_order s
  JOIN erp_product_info pi ON s.model_no = pi.product_code
  WHERE s.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND s.order_status NOT IN ('취소')
    AND s.pay_cost / s.p_ea < pi.msrp * 0.95) discounted
```
**추가 확인**: tbe.kr 광고 전환매출 비중 → 전체의 60%+ 이면 광고 착시

#### Q: "현재 재고 구조는 성장 대비 준비된 구조야, 방어형 구조야?"
```sql
-- 재고 포트폴리오 분석
SELECT
  SUM(CASE WHEN growth = 'star' THEN stock_value END) as star_stock,
  SUM(CASE WHEN growth = 'growing' THEN stock_value END) as growing_stock,
  SUM(CASE WHEN growth = 'stable' THEN stock_value END) as stable_stock,
  SUM(CASE WHEN growth = 'declining' THEN stock_value END) as declining_stock,
  SUM(CASE WHEN growth = 'dead' THEN stock_value END) as dead_stock,
  SUM(stock_value) as total_stock_value,
  CASE
    WHEN (star_stock + growing_stock) / SUM(stock_value) > 0.5 THEN '성장 준비형'
    WHEN (stable_stock) / SUM(stock_value) > 0.5 THEN '안정 유지형'
    WHEN (declining_stock + dead_stock) / SUM(stock_value) > 0.3 THEN '방어형 (부실 재고 30%+)'
    ELSE '균형형'
  END as portfolio_assessment
FROM (
  SELECT pi.product_code,
    stock.ea * pi.msrp as stock_value,
    CASE
      WHEN trend.growth_rate > 0.2 AND sales.monthly > 50 THEN 'star'
      WHEN trend.growth_rate > 0 THEN 'growing'
      WHEN trend.growth_rate BETWEEN -0.1 AND 0 THEN 'stable'
      WHEN trend.growth_rate < -0.1 AND sales.monthly > 0 THEN 'declining'
      ELSE 'dead'
    END as growth
  FROM erp_product_info pi
  JOIN (/*가용재고수량*/) stock ON pi.product_code = stock.product_code
  JOIN (/*월간판매*/) sales ON pi.product_code = sales.model_no
  JOIN (/*성장률*/) trend ON pi.product_code = trend.model_no
) portfolio
```
**판정 기준**:
| 포트폴리오 유형 | 성장+스타 비중 | 부실(하락+사장) 비중 | 평가 |
|---------------|-------------|-------------------|------|
| 성장 준비형 | 50%+ | <15% | 공격적 확장 가능 |
| 방어형 | <20% | 30%+ | 재고 구조조정 필요 |
| 균형형 | 30~50% | 15~30% | 안정적 |

#### Q: "우리가 과발주하고 있는 SKU는?"
```sql
SELECT pi.product_name,
  stock.available_ea,
  sales.daily_avg,
  ROUND(stock.available_ea / NULLIF(sales.daily_avg, 0)) as days_coverage,
  incoming.incoming_ea,
  ROUND((stock.available_ea + COALESCE(incoming.incoming_ea, 0))
    / NULLIF(sales.daily_avg, 0)) as total_coverage_days,
  CASE
    WHEN total_coverage_days > 180 THEN '심각한 과발주 (6개월+)'
    WHEN total_coverage_days > 120 THEN '과발주 (4개월+)'
    WHEN total_coverage_days > 90 THEN '주의'
    ELSE '적정'
  END as overorder_risk
FROM erp_product_info pi
JOIN (SELECT product_code, COUNT(*) as available_ea
      FROM erp_stock WHERE export_date IS NULL GROUP BY product_code) stock
  ON pi.product_code = stock.product_code
JOIN (SELECT model_no, SUM(p_ea)/30 as daily_avg
      FROM tbnws_sabangnet_order
      WHERE order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY) AND order_status NOT IN ('취소')
      GROUP BY model_no) sales ON pi.product_code = sales.model_no
LEFT JOIN (SELECT oi.product_code, SUM(oi.ea) as incoming_ea
           FROM erp_order_item oi JOIN erp_order o ON oi.order_seq = o.order_seq
           WHERE o.arr_flag != 'Y' GROUP BY oi.product_code) incoming
  ON pi.product_code = incoming.product_code
WHERE total_coverage_days > 90
ORDER BY total_coverage_days DESC
```

#### Q: "우리가 과소평가한 SKU는?"
```sql
-- 판매 성장세 + 재고 부족 + 안전재고 미달
SELECT pi.product_name,
  trend.growth_rate_pct,
  sales.daily_avg,
  stock.available_ea,
  ROUND(stock.available_ea / sales.daily_avg) as days_remaining,
  pi.safe_ea,
  CASE
    WHEN trend.growth_rate_pct > 30 AND stock.available_ea < pi.safe_ea
      THEN '급성장 + 재고 부족: 긴급 증량 발주'
    WHEN trend.growth_rate_pct > 15 AND stock.available_ea < pi.safe_ea * 2
      THEN '성장 중 + 재고 주의: 발주 증량 검토'
    ELSE '모니터링'
  END as assessment
FROM erp_product_info pi
JOIN (/*성장률*/) trend ON pi.product_code = trend.model_no
JOIN (/*일평균판매*/) sales ON pi.product_code = sales.model_no
JOIN (/*가용재고*/) stock ON pi.product_code = stock.product_code
WHERE trend.growth_rate_pct > 15
  AND stock.available_ea < pi.safe_ea * 2
ORDER BY trend.growth_rate_pct DESC
```

#### Q: "현재 가장 위험한 SKU 5개와 그 이유는?"
```sql
-- 종합 위험 스코어링 (0~100)
SELECT pi.product_name, pi.product_code,
  -- 위험 점수 산출 (각 요소 가중합)
  ROUND(
    COALESCE(stockout_risk, 0) * 30 +     -- 재고 고갈 위험 (30%)
    COALESCE(margin_risk, 0) * 20 +        -- 마진 하락 위험 (20%)
    COALESCE(demand_decline_risk, 0) * 25 + -- 수요 하락 위험 (25%)
    COALESCE(quality_risk, 0) * 15 +        -- 품질 리스크 (15%)
    COALESCE(channel_risk, 0) * 10          -- 채널 집중 위험 (10%)
  ) as total_risk_score,
  -- 위험 상세
  CONCAT_WS(', ',
    CASE WHEN stockout_risk > 0.7 THEN '재고고갈임박' END,
    CASE WHEN margin_risk > 0.7 THEN '마진악화' END,
    CASE WHEN demand_decline_risk > 0.7 THEN '수요급감' END,
    CASE WHEN quality_risk > 0.7 THEN '품질이슈' END,
    CASE WHEN channel_risk > 0.7 THEN '채널편중' END
  ) as risk_reasons
FROM erp_product_info pi
-- 각 위험 요소별 서브쿼리 조인
LEFT JOIN (/*재고고갈위험: 가용일수 < 14일이면 1.0*/) stockout ON ...
LEFT JOIN (/*마진위험: 마진율 < 15%이면 1.0*/) margin ON ...
LEFT JOIN (/*수요하락: 전월대비 -30%이면 1.0*/) demand ON ...
LEFT JOIN (/*품질위험: AS건수 상위 10%이면 1.0*/) quality ON ...
LEFT JOIN (/*채널위험: 단일채널 70%+이면 1.0*/) channel ON ...
ORDER BY total_risk_score DESC
LIMIT 5
```
**위험 가중치**: 재고고갈(30%) > 수요하락(25%) > 마진(20%) > 품질(15%) > 채널(10%)

#### Q: "3개월 후 가장 큰 리스크는 '재고 부족'이야, '재고 과잉'이야?"
```sql
-- 시나리오 분석: 현재 추세 유지 가정
SELECT
  shortage.sku_count as shortage_risk_skus,
  shortage.total_revenue_at_risk as shortage_revenue_risk,
  surplus.sku_count as surplus_risk_skus,
  surplus.total_stock_value_at_risk as surplus_value_risk,
  CASE
    WHEN shortage.total_revenue_at_risk > surplus.total_stock_value_at_risk
    THEN '최대 리스크: 재고 부족 (매출 손실 ' || shortage.total_revenue_at_risk || '원)'
    ELSE '최대 리스크: 재고 과잉 (자금 묶임 ' || surplus.total_stock_value_at_risk || '원)'
  END as primary_risk
FROM (-- 3개월 내 품절 예상 SKU
  SELECT COUNT(*) as sku_count,
    SUM(daily_avg * msrp * 90) as total_revenue_at_risk
  FROM (/*재고일수 < 90일 + 입고예정 없음*/) shortage_skus
) shortage
CROSS JOIN (-- 3개월 후에도 재고 잔여 과잉 SKU
  SELECT COUNT(*) as sku_count,
    SUM(excess_ea * msrp) as total_stock_value_at_risk
  FROM (/*재고 - 90일 판매예상 > 안전재고 × 3*/) surplus_skus
) surplus
```

#### Q: "시장이 꺾이기 시작했다면 가장 먼저 신호가 나타날 지표는?"
```
-- 선행 지표 우선순위 (빠른 순):
-- 1위: 브랜드 검색량 감소 (선행 7~14일) → tbe.kr/네이버 트렌드
-- 2위: 신규 고객 비율 감소 (선행 14~21일)
-- 3위: 장바구니 이탈률 증가 → GA/tbe.kr 데이터
-- 4위: 전환율 하락 (CTR 유지, CVR 하락)
-- 5위: 객단가 하락 (할인 없이도 저가 제품으로 이동)
-- 6위: 매출 감소 (후행 지표)

-- 현재 선행지표 대시보드
SELECT
  '브랜드 검색량' as indicator,
  search_trend.change_pct as current_change,
  CASE WHEN search_trend.change_pct < -10 THEN '경고' ELSE '정상' END as status,
  1 as priority
UNION ALL
SELECT '신규 고객 비율',
  new_customer.ratio_change,
  CASE WHEN new_customer.ratio_change < -5 THEN '경고' ELSE '정상' END, 2
UNION ALL
SELECT '전환율(CVR)',
  cvr.change_pct,
  CASE WHEN cvr.change_pct < -15 THEN '경고' ELSE '정상' END, 3
UNION ALL
SELECT '객단가(AOV)',
  aov.change_pct,
  CASE WHEN aov.change_pct < -10 THEN '경고' ELSE '정상' END, 4
ORDER BY priority
```
**선행 지표 체계**:
| 순위 | 지표 | 선행 기간 | 데이터 소스 |
|------|------|----------|-----------|
| 1 | 브랜드 검색량 | 7~14일 | tbe.kr, 네이버 트렌드 |
| 2 | 신규 고객 비율 | 14~21일 | `tbnws_sabangnet_order` |
| 3 | 전환율(CVR) | 7~14일 | tbe.kr |
| 4 | 객단가(AOV) | 즉시 | `tbnws_sabangnet_order` |
| 5 | 매출 | 후행 | `tbnws_sabangnet_order` |

---

## 크로스 도메인 질문 매핑

### 복합 질문 → 복수 데이터 소스 조합

| 질문 | 필요 데이터 소스 | 분석 유형 |
|------|-----------------|----------|
| "매출 좋은데 재고 부족한 제품" | `tbnws_sabangnet_order` + `erp_stock` | 재고-판매 갭 |
| "광고비 많이 쓰는데 재고 없는 제품" | tbe.kr + `erp_stock` | 마케팅-재고 미스매치 |
| "A/S 많은 제품의 매출 영향" | `crm_as` + `tbnws_sabangnet_order` | 품질-매출 상관 |
| "환율 오를때 발주 타이밍" | `currency_exchange_rate` + `erp_order` | 환율-발주 최적화 |
| "B2B 매출 하락이 전체 매출에 미치는 영향" | `erp_order_recv` + `erp_annual_sales` | 채널 영향도 |
| "마케팅 중단 시 매출 타격 큰 SKU" | tbe.kr + `tbnws_sabangnet_order` | 광고 의존도 |
| "OOS 시 대체품(카니발) 효과" | `erp_stock` + `tbnws_sabangnet_order` + `erp_product_info` | 카니발라이제이션 |
| "춘절 지연 시 매출 손실 예측" | `erp_order` + `erp_stock` + `tbnws_sabangnet_order` | 시즌 시뮬레이션 |
| "항공 전환 ROI 분석" | `erp_order_item`(비용) + `tbnws_sabangnet_order`(매출) | 비용-편익 |
| "프로모션 착시 vs 진짜 수요 판별" | `tbnws_sabangnet_order` + `erp_product_info`(MSRP) + tbe.kr | 수요 품질 |
| "재고 포트폴리오 건강도" | `erp_stock` + `tbnws_sabangnet_order` + `erp_order_item` | 포트폴리오 |
| "SKU별 종합 위험 스코어" | 전체 DB (재고+판매+CS+마진+채널) | 리스크 스코어링 |
| "시장 선행지표 대시보드" | tbe.kr(검색) + `tbnws_sabangnet_order`(매출/고객) | 시장 국면 |
| "가격 탄력성 기반 할인 시뮬레이션" | `tbnws_sabangnet_order` + `erp_product_info` + `erp_discounted_products` | 가격 전략 |
| "리뷰 하락 → 판매 하락 상관분석" | `naver_store_review_item` + `tbnws_sabangnet_order` | VOC-매출 |
| "발주 묶음(MOQ) 최적화" | `erp_order_item` + `erp_partner` + `erp_product_info` | 발주 최적화 |

### 분석 난이도별 분류

| 난이도 | 설명 | 예시 |
|--------|------|------|
| **Level 1** (직접 조회) | 단일 테이블 집계 | 매출 추이, 재고 현황, AS 건수 |
| **Level 2** (크로스 조인) | 2~3 테이블 조합 | 마진 분석, 재고회전율, 채널별 매출 |
| **Level 3** (시계열 분석) | 기간 비교 + 트렌드 | 판매 급감 감지, 시즌 패턴, YoY 비교 |
| **Level 4** (시뮬레이션) | What-if 분석 | 환율 변동 시나리오, 할인 시뮬레이션, 발주 타이밍 |
| **Level 5** (예측/판정) | 복합 지표 + 판단 | 시장 국면, 수요 품질, 종합 위험 스코어 |

### 필요 보강 데이터

| 데이터 | 현재 상태 | 활용처 |
|--------|----------|--------|
| tbe.kr 광고 데이터 API | 별도 DB, 연동 필요 | ROAS, CAC, 키워드 분석 |
| 일별 재고 스냅샷 | 미존재, 구축 필요 | OOS 기간 정확 식별 |
| 경쟁사 가격 데이터 | 외부 크롤링 필요 | 경쟁 가격 영향 분석 |
| Google Analytics | 별도 연동 필요 | 유입 트래픽, 전환율 |
| 소셜 모니터링 | 외부 API 필요 | SNS 언급량, 브랜드 인지도 |
| 공급처별 국가/공휴일 | erp_partner 보강 필요 | 시즌 리드타임 예측 |
| MOQ 정보 | erp_partner 또는 별도 테이블 | 발주 묶음 최적화 |

---

## 5. CS 담당 (고객서비스)

### 🎧 고객 문의 관리

#### Q: "CS 이슈 현황 알려줘" / "CS 쪽에는 어떤 이슈가 있어?"
```
분석 방법 (4개 데이터 소스 순차 조회):
1. naver_store_customer_inquiries → 1:1 문의 미답변 현황
2. naver_smartstore_qna → 상품 QnA 미답변 현황
3. naver_smartstore_orders (claim_type) → 취소/반품/교환 클레임 추이
4. crm_as + crm_as_product → AS 접수/처리율/진단 패턴
```

#### Q: "미답변 문의 있어?" / "답변 안 한 문의 목록"
```sql
-- ★ 검증 완료 (2026-02-28)
-- 1:1 미답변 문의 (긴급도 높은 순)
SELECT inquiry_no, brand_channel, category, title,
       inquiry_registration_date_time, product_name,
       DATEDIFF(NOW(), inquiry_registration_date_time) AS days_waiting
FROM naver_store_customer_inquiries
WHERE answered = 0
ORDER BY inquiry_registration_date_time ASC;  -- 오래된 것부터

-- 상품 QnA 미답변
SELECT account, question_date, product_name,
       SUBSTRING(question_content, 1, 80) AS q_preview
FROM naver_smartstore_qna
WHERE answered = 0
ORDER BY question_date DESC;
```
**관련 테이블**: `naver_store_customer_inquiries`, `naver_smartstore_qna`
**키 컬럼**: `answered` (0=미답변, 1=답변완료), `category` (상품/교환/반품/배송/기타/환불)

#### Q: "문의 답변율 어떻게 돼?" / "CS 응답률"
```sql
-- ★ 검증 완료 (2026-02-28) — 월별 답변율 추이
-- ⚠️ DATE_FORMAT 사용 시 Python에서 %%Y-%%m 이스케이핑 필요
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN answered = 1 THEN 1 ELSE 0 END) AS answered_cnt,
    SUM(CASE WHEN answered = 0 THEN 1 ELSE 0 END) AS unanswered_cnt,
    ROUND(SUM(CASE WHEN answered = 1 THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS answer_rate
FROM naver_store_customer_inquiries
WHERE inquiry_registration_date_time >= '2026-02-01'
  AND inquiry_registration_date_time < '2026-03-01';

-- 카테고리별 답변율
SELECT category,
    COUNT(*) AS total,
    SUM(CASE WHEN answered = 1 THEN 1 ELSE 0 END) AS answered_cnt,
    SUM(CASE WHEN answered = 0 THEN 1 ELSE 0 END) AS unanswered_cnt
FROM naver_store_customer_inquiries
WHERE inquiry_registration_date_time >= '2026-02-01'
GROUP BY category
ORDER BY total DESC;
```

---

### 📦 클레임 분석 (반품/교환/취소)

#### Q: "반품/취소 현황 어때?" / "클레임 추이 보여줘"
```sql
-- ★ 검증 완료 (2026-02-28) — naver_smartstore_orders.claim_type 활용
-- ⚠️ 월별 조회 시 Python에서 start/end 변수로 분리 (DATE_FORMAT % 이스케이핑 문제)
SELECT
    SUM(CASE WHEN claim_type = 'CANCEL' THEN 1 ELSE 0 END) AS cancel_cnt,
    SUM(CASE WHEN claim_type = 'RETURN' THEN 1 ELSE 0 END) AS return_cnt,
    SUM(CASE WHEN claim_type = 'EXCHANGE' THEN 1 ELSE 0 END) AS exchange_cnt,
    SUM(CASE WHEN claim_type = 'ADMIN_CANCEL' THEN 1 ELSE 0 END) AS admin_cancel,
    COUNT(CASE WHEN claim_type IS NOT NULL AND claim_type != '' THEN 1 END) AS claim_total,
    COUNT(*) AS total_orders,
    ROUND(COUNT(CASE WHEN claim_type IS NOT NULL AND claim_type != '' THEN 1 END)
          / COUNT(*) * 100, 1) AS claim_rate
FROM naver_smartstore_orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01';
```
**관련 테이블**: `naver_smartstore_orders`
**키 컬럼**: `claim_type` (CANCEL/RETURN/EXCHANGE/ADMIN_CANCEL), `order_date`
**벤치마크**: 클레임률 12~13%가 정상 범위

#### Q: "반품 많은 상품은?" / "클레임 다발 제품"
```sql
-- ★ 검증 완료 (2026-02-28)
SELECT product_name,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN claim_type IN ('RETURN','EXCHANGE') THEN 1 ELSE 0 END) AS ret_exch,
    SUM(CASE WHEN claim_type = 'CANCEL' THEN 1 ELSE 0 END) AS cancels,
    ROUND(SUM(CASE WHEN claim_type IN ('RETURN','EXCHANGE') THEN 1 ELSE 0 END)
          / COUNT(*) * 100, 1) AS ret_rate
FROM naver_smartstore_orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01'
GROUP BY product_name
HAVING total_orders >= 20
ORDER BY ret_rate DESC
LIMIT 15;
```
**관련 테이블**: `naver_smartstore_orders`
**벤치마크**: 반품+교환율 10% 이상이면 상품 품질/상세페이지 점검 필요

---

### 🔧 AS 관리

#### Q: "AS 현황 어때?" / "AS 처리율"
```sql
-- ★ 검증 완료 (2026-02-28) — 처리 상태는 start_date/complete_date 조합
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN complete_date IS NULL AND start_date IS NOT NULL THEN 1 ELSE 0 END) AS in_progress,
    SUM(CASE WHEN complete_date IS NULL AND start_date IS NULL THEN 1 ELSE 0 END) AS pending,
    ROUND(SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS completion_rate
FROM crm_as
WHERE regist_date >= '2026-02-01' AND regist_date < '2026-03-01';

-- 전체 미처리 백로그
SELECT
    SUM(CASE WHEN complete_date IS NULL AND start_date IS NULL THEN 1 ELSE 0 END) AS pending,
    SUM(CASE WHEN complete_date IS NULL AND start_date IS NOT NULL THEN 1 ELSE 0 END) AS in_progress
FROM crm_as;
```
**관련 테이블**: `crm_as`
**키 로직**: `complete_date IS NOT NULL`=완료, `start_date IS NOT NULL`=진행중, 둘 다 NULL=대기

#### Q: "AS 고장 패턴은?" / "어떤 고장이 많아?"
```sql
-- ★ 검증 완료 (2026-02-28) — crm_as + crm_as_product JOIN
-- AS 처리 유형 분포
SELECT p.process_type, COUNT(*) AS cnt
FROM crm_as_product p
JOIN crm_as a ON p.as_seq = a.seq
WHERE a.regist_date >= '2026-02-01' AND a.regist_date < '2026-03-01'
GROUP BY p.process_type
ORDER BY cnt DESC;

-- AS 해결 유형 분포 (A/S수리 vs 교환 vs 반품 vs 선조치)
SELECT p.resolution_type, COUNT(*) AS cnt
FROM crm_as_product p
JOIN crm_as a ON p.as_seq = a.seq
WHERE a.regist_date >= '2026-02-01' AND a.regist_date < '2026-03-01'
GROUP BY p.resolution_type
ORDER BY cnt DESC;

-- 진단 내용 샘플 (고장 패턴 파악)
SELECT p.diagnose
FROM crm_as_product p
JOIN crm_as a ON p.as_seq = a.seq
WHERE a.regist_date >= '2026-02-01'
  AND p.diagnose IS NOT NULL AND p.diagnose != ''
ORDER BY a.regist_date DESC LIMIT 15;
```
**관련 테이블**: `crm_as`, `crm_as_product`
**키 컬럼**: `process_type` (종결/회수중/입고 완료/수리 완료), `resolution_type` (A/S/교환/반품/선조치), `diagnose`
**빈출 고장**: 메인보드 불량, 특정키 입력 씹힘, 펌웨어 이슈, 스위치 이탈

---

## 6. 마케팅 담당 (광고 & 퍼포먼스)

### 📊 광고 성과 분석

#### Q: "광고 현황 분석해줘" / "광고 성과 어때?"
```
분석 방법 (종합 분석 = 4단계):
1. SA 성과: naver_ad_campaign_daily → 광고비(sales_amt), 전환매출(conv_amt), ROAS
2. GFA 성과: naver_ad_gfa_campaign_daily → 광고비(total_cost), 전환매출(purchase_conv_revenue)
3. 퍼널 분석: GFA campaign_name CASE WHEN으로 TOF/MOF/BOF/ADVoost/리타게팅 분류
4. 매출 연동: naver_smartstore_order_daily → stat_date, account, revenue, order_cnt

⚠️ 컬럼명 주의:
- SA: sales_amt(광고비), conv_amt(전환매출), ccnt(전환수), clk_cnt(클릭수)
- GFA: total_cost(광고비), purchase_conv_revenue(전환매출), purchases(전환수), clicks(클릭수)
- 키워드: ccnt (NOT conv_cnt)
- 비즈머니: bizmoney (NOT balance)
- 매출일별: stat_date (NOT date)
```

#### Q: "SA/GFA 월별 광고비, ROAS 추이"
```sql
-- ★ 검증 완료 (2026-02-28) — 월별 루프로 조회 (DATE_FORMAT % 이스케이핑 문제 우회)
-- Python에서 months 리스트 순회하며 start/end 파라미터 전달

-- SA 월별
SELECT IFNULL(SUM(sales_amt),0) AS sa_spend,
       IFNULL(SUM(conv_amt),0) AS sa_conv
FROM naver_ad_campaign_daily
WHERE stat_date >= %s AND stat_date < %s;

-- GFA 월별
SELECT IFNULL(SUM(total_cost),0) AS gfa_spend,
       IFNULL(SUM(purchase_conv_revenue),0) AS gfa_conv
FROM naver_ad_gfa_campaign_daily
WHERE stat_date >= %s AND stat_date < %s;

-- ROAS 계산
-- SA ROAS = ROUND(sa_conv / NULLIF(sa_spend, 0) * 100)
-- GFA ROAS = ROUND(gfa_conv / NULLIF(gfa_spend, 0) * 100)
```
**관련 테이블**: `naver_ad_campaign_daily` (SA), `naver_ad_gfa_campaign_daily` (GFA)

#### Q: "GFA 퍼널 분석해줘" / "TOF MOF BOF 성과"
```sql
-- ★ 검증 완료 (2026-02-28) — Binet & Field 60/40 Framework 적용
SELECT
    CASE
        WHEN campaign_name LIKE 'TOF%%' THEN 'TOF (인지/트래픽)'
        WHEN campaign_name LIKE 'MOF%%' THEN 'MOF (고려)'
        WHEN campaign_name LIKE 'BOF%%' THEN 'BOF (전환)'
        WHEN campaign_name LIKE '%%리타게팅%%' OR campaign_name LIKE '%%리타겟%%' THEN 'Retargeting'
        WHEN campaign_name LIKE '%%ADVoost%%' OR campaign_name LIKE '%%advoost%%' THEN 'ADVoost'
        ELSE 'Other'
    END AS funnel_stage,
    COUNT(DISTINCT campaign_name) AS campaign_cnt,
    SUM(total_cost) AS total_spend,
    SUM(impressions) AS total_impressions,
    SUM(clicks) AS total_clicks,
    ROUND(SUM(clicks)/NULLIF(SUM(impressions),0)*100, 2) AS ctr,
    SUM(purchases) AS total_purchases,
    SUM(purchase_conv_revenue) AS total_revenue,
    ROUND(SUM(purchase_conv_revenue)/NULLIF(SUM(total_cost),0)*100, 0) AS roas
FROM naver_ad_gfa_campaign_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
GROUP BY funnel_stage
ORDER BY total_spend DESC;

-- Binet & Field 비율 확인 (Brand 60% : Activation 40% 이상적)
-- Brand Building = TOF + MOF
-- Activation = BOF + Retargeting
-- ADVoost, Other = 별도 분류
```
**관련 테이블**: `naver_ad_gfa_campaign_daily`
**Binet & Field Framework**: Brand Building(TOF+MOF) 60% : Activation(BOF+RT) 40% 이상적
**현재 비율 (2월)**: Brand 41.6% : Activation 58.4% → Brand 비중 확대 필요

---

### 🔑 키워드 분석

#### Q: "전환 없는 키워드는?" / "낭비 키워드 알려줘"
```sql
-- ★ 검증 완료 (2026-02-28)
-- ⚠️ 전환수 컬럼: ccnt (NOT conv_cnt)
SELECT keyword,
    SUM(sales_amt) AS spend,
    SUM(clk_cnt) AS clicks,
    SUM(ccnt) AS conversions,
    ROUND(SUM(sales_amt)/NULLIF(SUM(clk_cnt),0), 0) AS cpc
FROM naver_ad_keyword_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
GROUP BY keyword
HAVING conversions = 0 AND spend > 5000
ORDER BY spend DESC;
```
**관련 테이블**: `naver_ad_keyword_daily`
**키 컬럼**: `keyword`, `sales_amt`(광고비), `ccnt`(전환수 ⚠️), `clk_cnt`(클릭수)

#### Q: "ROAS 높은 키워드는?" / "성과 좋은 키워드"
```sql
-- ★ 검증 완료 (2026-02-28) — None 값 처리 필요
SELECT keyword,
    SUM(sales_amt) AS spend,
    SUM(clk_cnt) AS clicks,
    SUM(ccnt) AS conversions,
    SUM(conv_amt) AS conv_revenue,
    ROUND(SUM(conv_amt)/NULLIF(SUM(sales_amt),0)*100, 0) AS roas
FROM naver_ad_keyword_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
GROUP BY keyword
HAVING conversions > 0
ORDER BY conv_revenue DESC
LIMIT 15;
```

---

### 💰 비즈머니 & 예산

#### Q: "비즈머니 잔액 얼마?" / "광고비 언제 떨어져?"
```sql
-- ★ 검증 완료 (2026-02-28) — 최신 잔액 조회
-- ⚠️ 컬럼명: bizmoney (NOT balance)
SELECT account, bizmoney, collected_at
FROM naver_ad_bizmoney
WHERE (account, collected_at) IN (
    SELECT account, MAX(collected_at) FROM naver_ad_bizmoney GROUP BY account
);

-- 일 평균 소진액 (잔여일수 계산용)
SELECT SUM(sales_amt) / COUNT(DISTINCT stat_date) AS sa_daily
FROM naver_ad_campaign_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01';

SELECT SUM(total_cost) / COUNT(DISTINCT stat_date) AS gfa_daily
FROM naver_ad_gfa_campaign_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01';

-- 잔여일수 = bizmoney / (sa_daily + gfa_daily)
```
**관련 테이블**: `naver_ad_bizmoney`, `naver_ad_campaign_daily`, `naver_ad_gfa_campaign_daily`

---

### 📈 매출-광고 연동

#### Q: "광고비 대비 매출 비율은?" / "스마트스토어 매출 추이"
```sql
-- ★ 검증 완료 (2026-02-28)
-- ⚠️ naver_smartstore_order_daily 컬럼: stat_date (NOT date), revenue, order_cnt, account
SELECT account, SUM(revenue) AS total_revenue, SUM(order_cnt) AS total_orders
FROM naver_smartstore_order_daily
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-03-01'
GROUP BY account
ORDER BY account;
```
**관련 테이블**: `naver_smartstore_order_daily`
**키 컬럼**: `stat_date`, `account` (keychron/gtgear), `revenue`, `order_cnt`

### AI 질문-광고/CS 데이터 매핑

| 질문 유형 | 데이터 소스 | 핵심 쿼리 |
|---|---|---|
| "광고 성과 어때?" | campaign_daily + gfa_campaign_daily | SA+GFA 합산 ROAS |
| "퍼널별 성과?" | gfa_campaign_daily | campaign_name CASE WHEN 퍼널 분류 |
| "키워드 낭비?" | keyword_daily | `ccnt = 0 AND sales_amt > 5000` |
| "비즈머니 잔액?" | naver_ad_bizmoney | `MAX(collected_at)` 최신 조회 |
| "미답변 문의?" | customer_inquiries | `answered = 0` |
| "클레임 현황?" | smartstore_orders | `claim_type` 집계 |
| "AS 처리율?" | crm_as | `complete_date IS NOT NULL` 비율 |
| "AS 고장 패턴?" | crm_as_product | `diagnose` 텍스트 + `resolution_type` |
| "반품 많은 상품?" | smartstore_orders | `claim_type IN ('RETURN','EXCHANGE')` 비율 |
| "캠페인 변경됐어?" | gfa_campaign_daily | MIN/MAX stat_date 비교 (리네임 감지) |
| "예산 변동 영향?" | gfa_campaign_daily + order_daily | 3일 vs 3일 비교 + lag 분석 |
| "한계효율 분석?" | gfa_campaign_daily | delta_cart_rev / delta_spend |
| "장바구니 vs 실매출?" | gfa_campaign_daily + order_daily | purchase_conv_revenue vs revenue |
| "대장주 분석?" | smartstore_orders + order_daily | 상품 비중 추이 + 클렌징 |

---

### 🔄 캠페인 구조 변경 감지

#### Q: "캠페인 변경된 거 있어?" / "중단된 캠페인?" / "새로 만든 광고?"
```
분석 방법:
1. 최근 N일 내 마지막/첫 데이터가 있는 캠페인 조회
2. 종료 캠페인과 시작 캠페인의 패턴(퍼널, result_type, 예산규모) 대조
3. 패턴 일치 → 리네임, 불일치 → 실제 중단/신설
```
```sql
-- 최근 3일 내 종료된 캠페인 (리네임 또는 실제 중단)
SELECT campaign_name, MAX(stat_date) AS last_date
FROM naver_ad_gfa_campaign_daily
WHERE account = 'keychron'
GROUP BY campaign_name
HAVING last_date BETWEEN DATE_SUB(CURDATE(), INTERVAL 5 DAY) AND DATE_SUB(CURDATE(), INTERVAL 2 DAY);

-- 같은 기간 신설된 캠페인 (리네임 또는 실제 신설)
SELECT campaign_name, MIN(stat_date) AS first_date
FROM naver_ad_gfa_campaign_daily
WHERE account = 'keychron'
GROUP BY campaign_name
HAVING first_date BETWEEN DATE_SUB(CURDATE(), INTERVAL 5 DAY) AND DATE_SUB(CURDATE(), INTERVAL 2 DAY);
```
**관련 테이블**: `naver_ad_gfa_campaign_daily`
**⚠️ 네이버 GFA 플랫폼**: 캠페인명 변경 = DB에서 구 캠페인 종료 + 신규 시작으로 기록됨
**판별**: 종료/시작 시점, result_type, 일평균 지출이 유사하면 리네임

---

### 📊 예산 변동 영향 분석

#### Q: "예산 변동 영향 분석해줘" / "광고비 바뀐 효과?"
```
분석 방법 (3단계):
1. 캠페인별 최근 3일 vs 이전 3일 지출 비교 → 변동률 ±20% 이상 감지
2. 변동 감지된 캠페인의 변경 전후 ROAS, 장바구니 건수 비교
3. 파이프라인 lag 확인: TOF 변동 → ADVoost CartRate 1~2일 후 변화
```
```sql
-- 캠페인별 예산 변동 감지
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
-- Python: change_pct = (recent_3d - prev_3d) / prev_3d * 100
-- |change_pct| > 20% → 변동 감지
```
**관련 테이블**: `naver_ad_gfa_campaign_daily`, `naver_smartstore_order_daily`
**⚠️ 소액 크림스키밍 주의**: 감액 후 ROAS 상승은 자연 현상 (한계효율 확인 필요)
**파이프라인 Lag**: TOF 노출 → ADVoost 장바구니 0~1일 lag 상관계수 0.55 수준

---

### 📈 한계효율(Marginal ROI) 분석

#### Q: "한계효율 분석?" / "예산 늘려도 될까?" / "과투자인지 확인"
```
분석 방법:
1. 예산 변경 전후 구간의 광고비, 장바구니매출 일평균 계산
2. 한계ROI = (후 매출 - 전 매출) / (후 광고비 - 전 광고비)
3. 한계ROI vs 평균ROAS 비교
   - 한계 > 평균: 확장 여력 있음
   - 한계 < 평균: 과투자 구간

소액 크림스키밍 기준:
| 예산 수준           | 기대 ROAS      |
|---------------------|---------------|
| 소액 (원래의 30%↓)  | 2~3배 상승     |
| 원래 예산           | 기준 ROAS      |
| 증액 (원래의 150%↑) | 30~50% 하락    |
```
**관련 테이블**: `naver_ad_gfa_campaign_daily`
**키 공식**: `marginal_roi = delta_cart_rev / delta_spend`
**⚠️ 최소 7일 안정 데이터 필요 (학습기간 3~5일 제외)**
**⚠️ 감액 후 ROAS↑는 크림스키밍 → 재증액 시 원래 수준 회귀 예상**

---

### 🔀 GFA 어트리뷰션 vs 실매출 괴리

#### Q: "장바구니 매출 vs 실매출 차이?" / "GFA 어트리뷰션 정확해?"
```
분석 방법:
1. GFA purchase_conv_revenue(장바구니 기반) vs SmartStore revenue(실제 결제) 일별 비교
2. 괴리 발생 시 원인:
   - BOF 감액/중단 → 장바구니↓ but 실매출 유지 = 직접방문 전환
   - TOF 증액 → 장바구니 소폭↑ but 실매출 더↑ = 인지도→직접구매
3. 총 광고 효율 = SS 실매출 / (SA + GFA 광고비)로 보정
```
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
**관련 테이블**: `naver_ad_gfa_campaign_daily`, `naver_smartstore_order_daily`
**⚠️ GFA UTM 미연동 → 캠페인별 실매출 기여 추적 불가**
**⚠️ "GFA 장바구니 매출 하락 ≠ 실제 매출 하락" — 반드시 SS 실매출과 대조**

---

### 🏆 대장주(주력 제품) 매출 분석

#### Q: "대장주 분석?" / "K10 매출 비중?" / "주력 제품 매출 추이"
```
분석 방법:
1. 상품별 매출 비중 추이 (naver_smartstore_orders → 상품별 SUM)
2. 비중 변동 판단 기준:
   - ±1%p: 노이즈 무시
   - 1~2%p: 약한 시그널 (2주 관찰)
   - 2~3%p: 중간 시그널 (이상치/연휴 클렌징 필수)
   - 3%p+: 강한 시그널 (그래도 클렌징 필요)
3. 클렌징 절차:
   ① 이상치 제거 (연휴 후 첫 출근, 역대 최고/최저)
   ② 요일 보정 (같은 요일 구성끼리 비교)
   ③ 연휴 기간 제외 (전후 2일 포함)
```
**관련 테이블**: `naver_smartstore_orders`, `naver_smartstore_order_daily`
**⚠️ 연휴 후 첫 출근일 이상치 반드시 제거** (보복소비로 비중 왜곡)
**⚠️ 비교는 반드시 같은 요일 구성끼리** (목 vs 목, 토 vs 토)

---

### ⚠️ 광고 분석 헛다리 방지 프레임워크

> GFA 캠페인 분석에서 반복되는 오류 패턴을 정리한 체크리스트와 기준값.

#### 분석 전 필수 체크리스트

| # | 체크 항목 | 왜 필요한가 |
|---|----------|------------|
| 1 | **캠페인 리네임/구조 변경 여부** | 리네임을 중단→신설로 오인하면 분석 틀어짐 |
| 2 | **캠페인 실제 운영 의도** (운영자 확인) | 숫자만 보면 다른 유형으로 오분류 |
| 3 | **해당 기간 공휴일/연휴** | 연휴 전후 ±50%는 광고와 무관 |
| 4 | **검색 트렌드(시장 수요)** | KDI(MA7)는 단기 스파이크 놓침, raw 확인 필수 |
| 5 | **데이터 일수** | 학습기간 제외 최소 7일 |
| 6 | **비교 구간 요일 구성** | 토요일 끼면 -28% 왜곡 |

#### 상관→인과 오류 방지

| 관찰 | 잘못된 결론 | 올바른 접근 |
|------|-----------|-----------|
| A 시작 후 매출↑ | A가 올렸다 | 동시 변수(계절/연휴/다른 캠페인) 확인 |
| A 감액 후 매출 유지 | A 불필요 | 간접효과(파이프라인) 확인, lag 분석 |
| ROAS 높은 캠페인 | 증액해야 | 소액 크림스키밍 확인, 한계효율 체크 |
| ROAS 낮은 캠페인 | 감액해야 | 파이프라인 기여(하위 퍼널 공급) 확인 |

#### 예산 테스트 설계 규칙

| 분석 유형 | 최소 일수 | 비고 |
|-----------|----------|------|
| ROAS 안정치 판단 | **7일** | 학습기간(3~5일) 제외 |
| 스윗스팟/최적예산 | **14일+** | 2개 예산 수준 각 7일+ |
| A/B 타겟 비교 | **14일** | 주말 2번 포함 |
| 소폭 조정 (±20%) | **7일** | 요일 1사이클 |
| 대폭 변경 (신규/중단) | **21일** | 학습 7일 + 안정 14일 |

#### 판단 기준 (같은 요일 대비 변동)

| 변동폭 | 판단 | 액션 |
|--------|------|------|
| ±10% 이내 | 노이즈 범위 | 관찰 계속 |
| **-15% 이상** | 예산 변경 영향 가능 | 추가 3일 관찰 또는 롤백 |
| **-25% 이상** | 확실한 영향 | **즉시 롤백** |
| +15% 이상 | 긍정적 효과 가능 | 1주 더 유지하여 확인 |

#### 달력/계절 효과 기준값

| 이벤트 | 매출 변동 | 연휴 후 첫 출근 |
|--------|----------|---------------|
| 설날 (5일 연휴) | -53% | +41% |
| 삼일절 (2일 연휴) | -15~20% | +32% |
| 토요일 | -28% (주중 대비) | — |
| 일요일 | -11% (주중 대비) | — |

> **핵심 원칙**: 결론 내기 전에 (1) 달력 확인 (2) 검색 트렌드 raw 확인 (3) 데이터 클렌징 (4) 운영자 의도 확인 (5) 최소 N수 충족 확인

---

## 데이터 접근 경로 요약

### DB별 주요 용도
| DB | 용도 | 핵심 테이블 |
|----|------|------------|
| **TBNWS_ADMIN** | 메인 ERP (상품/재고/발주/매출/B2B/CRM/HR/재무) | erp_*, finance_*, crm_*, member |
| **tbe.kr DB** | 마케팅 분석 (SA/GFA 광고 데이터) | 별도 접근 필요 |
| **gtgear** | GTGear 브랜드 데이터 | gtgear_* |
| **WMS** | 창고 관리 | wms_* |
| **attendance** | 근태 관리 | attendance_* |
| **shopify** | Shopify 연동 | shopify_* |

### 핵심 조인 경로
```
[판매 추적 - 사방넷]
tbnws_sabangnet_order.model_no = erp_product_info.product_code

[판매 추적 - 네이버 커머스] ★ 마진 분석 핵심
naver_commerce.productOrderId = naver_commerce_product_detail.product_order_id
erp_product_info.goods_code = naver_commerce_product_detail.seller_product_code
→ 유효주문: product_order_status IN ('DELIVERING','PURCHASE_DECIDED','DELIVERED','PAYED')

[판매 추적 - 수동입력 (판매분석 대시보드)]
erp_customer_order_list.product_name = erp_customer_order_product_list.product_name
erp_customer_order_list.partner_name = erp_customer_order_partner_list.partner_name
→ 모델 분류: erp_customer_order_product_list.model_name

[재고 추적]
erp_stock.product_code = erp_product_info.product_code
erp_stock.item_seq = erp_order_item.item_seq (로트 추적)

[발주 추적]
erp_order.partner_code = erp_partner.partner_code
erp_order_item.order_seq = erp_order.order_seq
erp_order_item.product_code = erp_product_info.product_code

[원가 추적] ★ 마진 분석 핵심
erp_order_item.product_code = erp_product_info.product_code
→ 환산원가 = buying_price × cur2krw × 1.1(VAT)
→ 최신 원가: MAX(item_seq) 기준

[B2B 추적]
erp_order_recv.biz_code = erp_order_recv_biz.biz_code
erp_order_recv_item.recv_seq = erp_order_recv.recv_seq

[CRM 추적]
crm_as_product.as_seq = crm_as.seq
crm_as_product.product_code = erp_product_info.product_code

[재무 추적]
finance_tax_invoice.biz_code = erp_order_recv_biz.biz_code
finance_tax_invoice_detail.product_code = erp_product_info.product_code
```

### 판매분석 페이지 API 매핑
```
[판매분석 대시보드 (sales-analysis-prototype.html)]
데이터소스: erp_customer_order_list (수동 입력)
기본 CTE: orderBaseDataCTE (erp-sales.xml)

API 엔드포인트:
├─ /api/erp/getMonthlySalesTargetData    → 월별 목표 vs 실적
├─ /api/erp/getSalesSummary              → 월별/주차별 파트너 매출
├─ /api/erp/getSalesSummaryDaily         → 일별 파트너 매출 (count, ea, price)
├─ /api/erp/getAnnualSalesList           → 년별 비교
├─ /api/erp/getBrandSaleData             → 브랜드별 월별
├─ /api/erp/getSalesModelList            → 모델별 월별 (ea + price 반환, 평균단가 계산 가능)
├─ /api/erp/getAccSaleData               → 악세서리별
└─ /api/erp/getAliceSaleData             → 엘리스배열별

⚠️ 모든 API가 erp_customer_order_list(수동) 기반
⚠️ 개별 SKU 실거래 단가 조회에는 naver_commerce_product_detail 필요
```
