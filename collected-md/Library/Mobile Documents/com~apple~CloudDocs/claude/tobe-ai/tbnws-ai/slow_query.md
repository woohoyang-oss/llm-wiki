# DB 성능 개선 사항 (Slow Query & Index 권고)

> 대상 DB: TBNWS_ADMIN (MySQL, 222.122.42.221:3306)
> 작성일: 2026-03-01
> 근거: tbnws_data_api 전 엔드포인트 SQL 분석 기반

---

## 1. 우선순위 HIGH — 즉시 적용 권고

### 1.1 `erp_stock_realtime` — F-코드 LEFT JOIN 성능

**문제**: 재고 관련 쿼리 (CURRENT_STOCK, REORDER_ALERTS, SMART_REORDER_ALERTS) 모두
`CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode` 패턴으로 JOIN.
이 함수 기반 JOIN은 인덱스를 타지 못해 Full Table Scan 발생 가능.

**영향 쿼리**: 6개 (inventory 도메인 전체)
```sql
LEFT JOIN erp_stock_realtime f
  ON CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode
```

**권고**:
```sql
-- 방법 1: productCode에 인덱스 확인 (이미 PK/UNIQUE면 OK)
-- f.productCode 검색이므로 f 테이블의 productCode 인덱스 필수
SHOW INDEX FROM erp_stock_realtime WHERE Column_name = 'productCode';

-- 방법 2: 가상 컬럼 추가 (MySQL 5.7+)
ALTER TABLE erp_stock_realtime
  ADD COLUMN g_code_ref VARCHAR(50) AS (
    CASE WHEN productCode LIKE 'G-%' THEN productCode
         WHEN productCode LIKE 'F-%' THEN CONCAT('G', SUBSTRING(productCode, 2))
    END
  ) STORED,
  ADD INDEX idx_gcode_ref (g_code_ref);

-- 방법 3: G/F 쌍 매핑 테이블 (가장 안전)
CREATE TABLE IF NOT EXISTS stock_code_mapping (
  g_code VARCHAR(50) NOT NULL,
  f_code VARCHAR(50) NOT NULL,
  PRIMARY KEY (g_code),
  INDEX idx_f_code (f_code)
);
INSERT INTO stock_code_mapping (g_code, f_code)
SELECT productCode, CONCAT('F', SUBSTRING(productCode, 2))
FROM erp_stock_realtime
WHERE productCode LIKE 'G-%';
```

**예상 효과**: JOIN 성능 10~100x 개선 (데이터 규모에 따라)

---

### 1.2 `naver_smartstore_orders` — 복합 인덱스 부재

**문제**: 매출 쿼리 전체가 `account + order_date + product_order_status` 3컬럼 조합 사용.
개별 인덱스만 있으면 WHERE 절에서 index merge 또는 full scan 발생.

**영향 쿼리**: 8개 (sales, cost, cs 도메인)
```sql
WHERE account = %s
  AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
```

**권고**:
```sql
-- 확인
SHOW INDEX FROM naver_smartstore_orders;

-- 복합 인덱스 추가 (order_date 범위 스캔 최적화)
ALTER TABLE naver_smartstore_orders
  ADD INDEX idx_account_orderdate_status (account, order_date, product_order_status);

-- product_name LIKE 검색용 (PRODUCT_REAL_PRICE 쿼리)
ALTER TABLE naver_smartstore_orders
  ADD INDEX idx_account_prodname (account, product_name(100));
```

**예상 효과**: 매출 관련 쿼리 전체 3~10x 개선

---

### 1.3 `naver_ad_keyword_daily` — GROUP BY keyword 풀스캔

**문제**: WASTE_KEYWORDS, TOP_KEYWORDS 쿼리에서
`account + stat_date` 범위 필터 후 `keyword` GROUP BY.
키워드 수가 수만~수십만일 경우 임시 테이블 + 정렬 부하.

**권고**:
```sql
ALTER TABLE naver_ad_keyword_daily
  ADD INDEX idx_account_statdate_keyword (account, stat_date, keyword);
```

---

## 2. 우선순위 MEDIUM — 데이터 증가 시 필요

### 2.1 `erp_stock` + `erp_order_item` LATERAL JOIN

**문제**: PRODUCT_COST, PRODUCTS_COST_BATCH 쿼리의 LATERAL JOIN은
각 product_code마다 서브쿼리를 실행. 상품 수 × erp_stock 크기만큼 반복.

```sql
LEFT JOIN LATERAL (
    SELECT ... FROM erp_stock s
    JOIN erp_order_item oi ON s.item_seq = oi.item_seq
    WHERE s.product_code = pi.product_code
      AND s.buying_price > 0
      AND oi.cur2krw > 100
    ORDER BY s.import_date DESC LIMIT 1
) lat ON TRUE
```

**권고**:
```sql
-- erp_stock 복합 인덱스
ALTER TABLE erp_stock
  ADD INDEX idx_prodcode_importdate (product_code, import_date DESC);

-- erp_order_item
SHOW INDEX FROM erp_order_item WHERE Column_name = 'item_seq';
-- item_seq가 PK가 아니면:
ALTER TABLE erp_order_item
  ADD INDEX idx_item_seq (item_seq);
```

### 2.2 `naver_store_customer_inquiries` — 미답변 필터

**문제**: `WHERE answered = 0` 풀스캔 가능 (answered 인덱스 없을 시)

**권고**:
```sql
ALTER TABLE naver_store_customer_inquiries
  ADD INDEX idx_answered_date (answered, inquiry_registration_date_time);
```

### 2.3 `crm_as` + `crm_as_product` JOIN

**문제**: AS 관련 쿼리에서 `start_date` 범위 + `complete_date IS NULL` 조합 사용.

**권고**:
```sql
ALTER TABLE crm_as
  ADD INDEX idx_startdate_completedate (start_date, complete_date);

ALTER TABLE crm_as_product
  ADD INDEX idx_as_seq (as_seq);
```

### 2.4 `naver_ad_campaign_daily` / `naver_ad_gfa_campaign_daily`

**문제**: 모든 광고 쿼리가 `account + stat_date` 조합 사용.

**권고**:
```sql
ALTER TABLE naver_ad_campaign_daily
  ADD INDEX idx_account_statdate (account, stat_date);

ALTER TABLE naver_ad_gfa_campaign_daily
  ADD INDEX idx_account_statdate (account, stat_date);
```

---

## 3. 우선순위 LOW — 모니터링

### 3.1 `tbnws_sabangnet_order` — 채널별 매출

```sql
-- order_date 인덱스 확인
ALTER TABLE tbnws_sabangnet_order
  ADD INDEX idx_orderdate (order_date);
```

### 3.2 `erp_purchase_order` — 발주 현황

```sql
-- register_date 인덱스 + order_seq NULL 체크용
ALTER TABLE erp_purchase_order
  ADD INDEX idx_registerdate (register_date);
```

### 3.3 `naver_smartstore_qna` — QnA 미답변

```sql
ALTER TABLE naver_smartstore_qna
  ADD INDEX idx_answered_questiondate (answered, question_date);
```

### 3.4 `md_keychron_sheet` — 마진 분석 매핑

```sql
-- naver_product_code로 JOIN하므로 인덱스 필수
ALTER TABLE md_keychron_sheet
  ADD INDEX idx_naver_prodcode (naver_product_code);
```

---

## 4. COLLATION 문제

### 4.1 `md_keychron_sheet` × `naver_smartstore_orders` JOIN

**문제**: MARGIN_BY_PRODUCT 쿼리에서 `COLLATE utf8mb4_unicode_ci` 명시 필요.
두 테이블의 문자열 컬럼 collation이 다르면 인덱스를 타지 못함.

```sql
-- 현재 collation 확인
SELECT TABLE_NAME, COLUMN_NAME, COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME IN ('md_keychron_sheet', 'naver_smartstore_orders')
  AND COLUMN_NAME IN ('naver_product_code', 'product_no');
```

**권고**: 가능하면 두 테이블의 collation을 통일.
```sql
ALTER TABLE md_keychron_sheet
  MODIFY naver_product_code VARCHAR(255)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## 5. 적용 전 확인 사항

```sql
-- 1. 기존 인덱스 확인 (중복 방지)
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'TBNWS_ADMIN'
  AND TABLE_NAME IN (
    'erp_stock_realtime', 'naver_smartstore_orders',
    'naver_ad_keyword_daily', 'erp_stock', 'erp_order_item',
    'naver_store_customer_inquiries', 'crm_as', 'crm_as_product',
    'naver_ad_campaign_daily', 'naver_ad_gfa_campaign_daily'
  )
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- 2. 테이블 크기 확인
SELECT TABLE_NAME, TABLE_ROWS,
       ROUND(DATA_LENGTH/1024/1024, 1) AS data_mb,
       ROUND(INDEX_LENGTH/1024/1024, 1) AS index_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'TBNWS_ADMIN'
  AND TABLE_NAME IN (
    'erp_stock_realtime', 'naver_smartstore_orders',
    'naver_ad_keyword_daily', 'erp_stock'
  );

-- 3. 슬로우 쿼리 로그 확인
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
-- 권장: long_query_time = 1 (1초 이상 쿼리 기록)
```

---

## 6. 요약 — 인덱스 추가 스크립트 (한번에 실행)

```sql
-- ⚠️ 실행 전 반드시 기존 인덱스 확인 (섹션 5 참조)
-- ⚠️ 대용량 테이블은 업무 시간 외 실행 권장

-- HIGH: 매출 복합 인덱스
ALTER TABLE naver_smartstore_orders
  ADD INDEX IF NOT EXISTS idx_account_orderdate_status (account, order_date, product_order_status);

-- HIGH: 광고 키워드
ALTER TABLE naver_ad_keyword_daily
  ADD INDEX IF NOT EXISTS idx_account_statdate_keyword (account, stat_date, keyword);

-- MEDIUM: 원가 조회
ALTER TABLE erp_stock
  ADD INDEX IF NOT EXISTS idx_prodcode_importdate (product_code, import_date DESC);

-- MEDIUM: CS 미답변
ALTER TABLE naver_store_customer_inquiries
  ADD INDEX IF NOT EXISTS idx_answered_date (answered, inquiry_registration_date_time);

-- MEDIUM: AS 처리
ALTER TABLE crm_as
  ADD INDEX IF NOT EXISTS idx_startdate_completedate (start_date, complete_date);

-- MEDIUM: 광고 성과
ALTER TABLE naver_ad_campaign_daily
  ADD INDEX IF NOT EXISTS idx_account_statdate (account, stat_date);

ALTER TABLE naver_ad_gfa_campaign_daily
  ADD INDEX IF NOT EXISTS idx_account_statdate (account, stat_date);

-- LOW: 기타
ALTER TABLE tbnws_sabangnet_order
  ADD INDEX IF NOT EXISTS idx_orderdate (order_date);

ALTER TABLE erp_purchase_order
  ADD INDEX IF NOT EXISTS idx_registerdate (register_date);

ALTER TABLE naver_smartstore_qna
  ADD INDEX IF NOT EXISTS idx_answered_questiondate (answered, question_date);

ALTER TABLE md_keychron_sheet
  ADD INDEX IF NOT EXISTS idx_naver_prodcode (naver_product_code);
```
