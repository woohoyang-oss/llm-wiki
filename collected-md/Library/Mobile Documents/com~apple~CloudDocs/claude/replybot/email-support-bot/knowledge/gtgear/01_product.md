# 상품 도메인 (Product Domain)

> TBNWS ERP 상품 마스터 데이터 — 카테고리/브랜드/상품/옵션 계층 구조 및 가격/채널/외부 연동 테이블
> 상품 코드 체계: `{category_code}-{brand_code}-{goods_code}-{option_code}` (G-code 시스템)

---

## erp_category (ERP 카테고리 목록)

summary: 상품 분류 체계의 최상위 계층으로, 모든 상품은 반드시 하나의 카테고리에 속한다. 'G'(일반상품), 'F'(풀필먼트/CJ) 등으로 구분되며, G와 F는 동일 상품의 창고별 분류이다.
domain: product
related_tables: erp_brand, erp_goods, erp_option, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | PK | 카테고리 코드 | 'G'(일반상품), 'F'(풀필먼트/CJ) |
| category_name | VARCHAR(10) | NOT NULL | 카테고리명 | '일반상품', '풀필먼트' |

### 관계
- erp_brand.category_code → erp_category.category_code (N:1): 브랜드는 하나의 카테고리에 속함
- erp_goods.category_code → erp_category.category_code (N:1): 상품은 하나의 카테고리에 속함
- erp_product_info.category_code → erp_category.category_code (N:1): 제품 마스터 카테고리 참조

### 비즈니스 맥락
- G-code와 F-code는 동일 상품의 다른 창고 구분이다: F-code = CONCAT('F', SUBSTRING(G-code, 2))
- category_code = 'G'는 GT 자체 창고, 'F'는 CJ 풀필먼트 창고를 의미
- 재고 집계 시 반드시 G + F 합산해야 정확한 재고가 산출됨
- product_view는 erp_product_info.product_code에서 SUBSTRING_INDEX로 category_code를 파싱

### 자주 쓰는 쿼리 패턴
- 전체 카테고리 조회: `SELECT category_code, category_name FROM erp_category`
- 판매 상품 필터: `WHERE category_code IN ('G', 'F')` (일반상품 + 풀필먼트)

---

## erp_brand (ERP 브랜드 목록)

summary: 카테고리 하위의 브랜드 정보를 관리하는 테이블. 키크론, GTGear 등 취급 브랜드를 정의하며, 상품 코드 체계의 두 번째 레벨이다.
domain: product
related_tables: erp_category, erp_goods, erp_option, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | PK (복합) | 카테고리 코드 | 'G', 'F' |
| brand_code | CHAR(10) | PK (복합) | 브랜드 코드 | '0001', '0002' |
| brand_name | VARCHAR(100) | NOT NULL | 브랜드명 | 'Keychron', 'Aiper' |

### 관계
- erp_brand.category_code → erp_category.category_code (N:1): 브랜드는 하나의 카테고리에 속함
- erp_goods.brand_code → erp_brand.brand_code (N:1): 상품은 하나의 브랜드에 속함
- erp_product_info.brand_code → erp_brand.brand_code (N:1): 제품 마스터 브랜드 참조

### 비즈니스 맥락
- 동일 브랜드가 G/F 양쪽 카테고리에 모두 등록됨 (복합 PK 이유)
- brand_code는 erp_product_info의 brand_code와 직접 매핑
- 원가 분석 시 brand_code 기준으로 일괄 조회하는 패턴이 자주 사용됨

### 자주 쓰는 쿼리 패턴
- 카테고리별 브랜드 조회: `SELECT category_code, brand_code, brand_name FROM erp_brand WHERE category_code = #{categoryCode}`
- 활성 브랜드 목록: `SELECT brand_code, brand_name FROM erp_brand WHERE category_code IN ('G','F') GROUP BY brand_name ORDER BY brand_name`

---

## erp_goods (ERP 상품 목록)

summary: 브랜드 하위의 상품(모델) 정보를 관리하는 테이블. 상품 코드 체계의 세 번째 레벨이며, 시리얼 번호 관리 여부를 포함한다.
domain: product
related_tables: erp_category, erp_brand, erp_option, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | PK (복합) | 카테고리 코드 | 'G', 'F' |
| brand_code | CHAR(10) | PK (복합) | 브랜드 코드 | '0001' |
| goods_code | CHAR(10) | PK (복합) | 상품 코드 | '0001', '0025' |
| goods_name | VARCHAR(100) | NOT NULL | 상품명 | 'K2 Pro', 'Q1 Max' |
| serial_flag | CHAR | NOT NULL, DEFAULT 'N' | 시리얼 존재 여부 | 'Y', 'N' |

### 관계
- erp_goods.(category_code, brand_code) → erp_brand.(category_code, brand_code) (N:1): 상품은 하나의 브랜드에 속함
- erp_option.(category_code, brand_code, goods_code) → erp_goods.(category_code, brand_code, goods_code) (N:1): 옵션은 하나의 상품에 속함

### 비즈니스 맥락
- goods_code는 4자리 제로패딩 문자열 (LPAD(MAX+1, 4, '0'))
- goods_name 변경 시 erp_goods와 erp_product_info 양쪽 모두 업데이트해야 함
- serial_flag = 'Y'인 상품은 시리얼 번호 개별 추적이 필요한 고가 상품

### 자주 쓰는 쿼리 패턴
- 브랜드별 상품 조회: `SELECT category_code, brand_code, goods_code, goods_name FROM erp_goods WHERE category_code = #{categoryCode} AND brand_code = #{brandCode}`
- 신규 상품코드 생성: `SELECT LPAD(CAST(CAST(COALESCE(MAX(goods_code), '0000') AS UNSIGNED) + 1 AS CHAR), 4, '0') FROM erp_goods WHERE category_code = #{categoryCode} AND brand_code = #{brandCode}`

---

## erp_option (ERP 옵션 목록)

summary: 상품 하위의 옵션(색상, 스위치 타입 등) 정보를 관리하는 테이블. 상품 코드 체계의 최하위 레벨이며, 옵션이 없는 상품은 이 테이블에 레코드가 없다.
domain: product
related_tables: erp_category, erp_brand, erp_goods, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | PK (복합), DEFAULT '' | 카테고리 코드 | 'G', 'F' |
| brand_code | CHAR(10) | PK (복합), DEFAULT '0000' | 브랜드 코드 | '0001' |
| goods_code | CHAR(10) | PK (복합), DEFAULT '0000' | 상품 코드 | '0025' |
| option_code | CHAR(10) | PK (복합), DEFAULT '0000' | 옵션 코드 | '0001', '0002' |
| option_name | VARCHAR(255) | NOT NULL, DEFAULT '' | 옵션 이름 | 'Red', 'Gateron Brown' |

### 관계
- erp_option.(category_code, brand_code, goods_code) → erp_goods.(category_code, brand_code, goods_code) (N:1): 옵션은 하나의 상품에 속함
- erp_product_info.option_code → erp_option.option_code (참조): 제품 마스터에서 옵션 참조

### 비즈니스 맥락
- option_code는 4자리 제로패딩 (COALESCE(LPAD(MAX+1, 4, '0'), '0001'))
- option_name 변경 시 erp_option과 erp_product_info 양쪽 모두 업데이트 + product_name도 재생성
- product_name 형식: `[{brand_name}] {goods_name} - {option_name}`
- 옵션이 없는 상품의 경우 product_code에 option_code 부분이 생략됨

### 자주 쓰는 쿼리 패턴
- 상품별 옵션 조회: `SELECT option_code, option_name FROM erp_option WHERE category_code = #{categoryCode} AND brand_code = #{brandCode} AND goods_code = #{goodsCode}`
- 형제 옵션 재고 비교 (옵션 그룹 분석): brand_code + goods_code로 그룹핑하여 옵션별 재고/판매량 비교

---

## erp_item_code (ERP 품목코드 매핑)

summary: 외부 시스템(eflexs 등)의 품목코드를 내부 G-code 체계(category/brand/goods/option)에 매핑하는 테이블. 외부 ERP 연동 시 상품 식별에 사용된다.
domain: product
related_tables: erp_product_info, eflexs_product

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| code_seq | INT UNSIGNED | PK, AUTO_INCREMENT | 품목코드 번호 | 1, 2, 3 |
| code | VARCHAR(150) | UNIQUE, NOT NULL, DEFAULT '' | 외부 품목코드 | 'ITEM-001' |
| category_code | CHAR | NOT NULL, DEFAULT '' | 카테고리 코드 | 'G' |
| brand_code | CHAR(7) | NOT NULL, DEFAULT '' | 브랜드 코드 | '0001' |
| goods_code | CHAR(6) | NOT NULL, DEFAULT '' | 상품 코드 | '0025' |
| option_code | CHAR(7) | NULL | 옵션 코드 | '0001', NULL |
| regist_date | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | '2024-01-15 10:00:00' |

### 관계
- erp_item_code.(category_code, brand_code, goods_code, option_code) → erp_product_info.(category_code, brand_code, goods_code, option_code) (논리적 1:1): 품목코드는 하나의 제품에 매핑

### 비즈니스 맥락
- 외부 ERP(eflexs) 품목코드와 내부 G-code를 연결하는 브릿지 테이블
- INSERT ON DUPLICATE KEY UPDATE로 upsert 처리
- 조회 시 erp_product_info와 LEFT JOIN하여 product_code, product_name을 함께 반환

### 자주 쓰는 쿼리 패턴
- 품목코드 목록 조회 (제품명 포함): `SELECT a.code, IFNULL(b.product_code, '') AS product_code, IFNULL(b.product_name, '') AS product_name FROM erp_item_code a LEFT JOIN erp_product_info b ON a.category_code = b.category_code AND a.brand_code = b.brand_code AND a.goods_code = b.goods_code AND COALESCE(a.option_code, '') = COALESCE(b.option_code, '')`

---

## erp_product_info (ERP 제품 마스터)

summary: 상품의 모든 마스터 데이터를 비정규화하여 저장하는 핵심 테이블. 카테고리/브랜드/상품/옵션 정보, 물류 규격, 가격, 판매 상태 등을 통합 관리하며, 모든 상품 조회의 시작점이다.
domain: product
related_tables: erp_category, erp_brand, erp_goods, erp_option, erp_stock, erp_channel_price, erp_discounted_products, erp_product_components, erp_product_details, erp_product_image, erp_product_qc, md_keychron_sheet

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_seq | INT UNSIGNED | PK, AUTO_INCREMENT | 상품 Unique Seq | 1, 2, 3 |
| product_code | VARCHAR(30) | NOT NULL, UNIQUE(복합) | 상품 코드 (G-code) | 'G-0001-0025-0001' |
| option_model | VARCHAR(255) | NULL | 옵션 모델명 | 'K2 Pro RGB' |
| series | VARCHAR(100) | NULL | 시리즈 분류 | 'K Pro', 'Q Max' |
| width | DOUBLE | NOT NULL, DEFAULT 0 | 가로 (mm) | 355.0 |
| depth | DOUBLE | NOT NULL, DEFAULT 0 | 세로 (mm) | 130.0 |
| height | DOUBLE | NOT NULL, DEFAULT 0 | 높이 (mm) | 40.0 |
| weight | DOUBLE | NOT NULL, DEFAULT 0 | 무게 (g) | 980.0 |
| units_per_box | INT | NOT NULL, DEFAULT 0 | 입수량 (박스당 수량) | 10 |
| loadage | INT | NOT NULL, DEFAULT 0 | 적재량 | 500 |
| barcode | VARCHAR(30) | NULL | 상품 바코드 | '8809907020014' |
| kc_num | VARCHAR(50) | NULL | KC 인증번호 | 'R-C-TCL-K2Pro' |
| battery_num | VARCHAR(50) | NULL | 배터리 인증번호 | '' |
| receiver_a_num | VARCHAR(50) | NULL | 리시버 TYPE-A 인증번호 | '' |
| receiver_c_num | VARCHAR(50) | NULL | 리시버 TYPE-C 인증번호 | '' |
| currency | CHAR(3) | NOT NULL, DEFAULT 'KRW' | 화폐 단위 | 'KRW', 'USD' |
| msrp | DOUBLE | NOT NULL, DEFAULT 0 | 권장소비자가격 | 139000 |
| duty | FLOAT | NULL, DEFAULT 0 | 관세 | 0.08 |
| fixed_price | BIGINT | NOT NULL, DEFAULT 0 | 정가 | 139000 |
| purchase_price | DOUBLE | NOT NULL, DEFAULT 0 | 사입가 | 85000 |
| actual_price | DOUBLE | NOT NULL, DEFAULT 0 | 실판가 | 129000 |
| safe_ea | INT | NOT NULL, DEFAULT 0 | 안전재고수량 | 50 |
| sell_status | CHAR | NOT NULL, DEFAULT 'Y' | 판매 상태 | 'Y'(판매중), 'D'(단종), 'N'(미취급), 'S'(판매중) |
| detail_type | CHAR | NOT NULL, DEFAULT 'X' | 상세 타입 | 'I'(이미지), 'B'(블로그), 'X'(분류전) |
| product_purpose | CHAR | NOT NULL, DEFAULT 'S' | 용도 | 'S'(판매용), 'D'(납품용), 'B'(판매및납품), 'R'(기타송금), 'E'(기타) |
| actual_price_channel_uptag | INT | NULL, DEFAULT 0 | 채널 업택 가격 | 0 |
| channel_price | INT | NULL, DEFAULT 0 | 채널 상시가 | 0 |
| channel_uptag_price | INT | NULL, DEFAULT 0 | 채널 업택 상시가 | 0 |
| product_name | VARCHAR(255) | NOT NULL, DEFAULT '' | 상품이름 (자동 생성) | '[Keychron] K2 Pro - Red' |
| actual_price_channel | INT | NULL, DEFAULT 0 | TAG가 (채널) | 0 |
| product_num | VARCHAR(255) | NULL | 상품번호 (외부 코드) | '' |
| category_code | VARCHAR(2) | NOT NULL, UNIQUE(복합) | 카테고리 코드 | 'G', 'F' |
| brand_code | VARCHAR(30) | NOT NULL, UNIQUE(복합) | 브랜드 코드 | '0001' |
| goods_code | VARCHAR(30) | NOT NULL, UNIQUE(복합) | 상품 코드 | '0025' |
| option_code | VARCHAR(30) | NOT NULL, DEFAULT '', UNIQUE(복합) | 옵션 코드 | '0001', '' |
| category_name | VARCHAR(255) | NOT NULL | 카테고리 이름 | '일반상품' |
| brand_name | VARCHAR(255) | NOT NULL | 브랜드 이름 | 'Keychron' |
| goods_name | VARCHAR(255) | NOT NULL | 상품 이름 | 'K2 Pro' |
| option_name | VARCHAR(255) | NOT NULL, DEFAULT '' | 옵션 이름 | 'Red' |
| type | CHAR | NULL | 상품 유형 | 'G'(상품), 'P'(부품), 'E'(기타), 'B'(리퍼브) |
| cj_code | VARCHAR(255) | NULL | CJ 코드 | '' |

### 관계
- erp_product_info.(category_code, brand_code, goods_code, option_code) UNIQUE: 복합 유니크 키
- erp_product_info → erp_stock (1:N): product_code로 연결, 재고 데이터 참조
- erp_product_info → erp_channel_price (1:N): product_code로 채널별 가격 참조
- erp_product_info → erp_discounted_products (1:N): product_code로 할인가 이력 참조
- erp_product_info → erp_product_components (1:N): product_code로 구성품 참조
- erp_product_info → erp_product_details (1:N): product_code로 상세 스펙 참조
- erp_product_info → erp_product_image (1:N): product_code로 이미지 참조
- erp_product_info → erp_product_qc (1:1): product_code로 QC 정보 참조

### 비즈니스 맥락
- 모든 상품 조회의 핵심 테이블 — 재고, 가격, 주문, 마진 분석 등 모든 도메인에서 참조
- product_code 형식: `{category_code}-{brand_code}-{goods_code}[-{option_code}]` (옵션 없으면 3단위)
- product_name은 자동 생성: `[{brand_name}] {goods_name} - {option_name}`
- sell_status 필터가 대부분의 조회에 사용됨: `sell_status NOT IN ('N', 'D')` 또는 `sell_status = 'Y'`
- type = 'G'로 필터하면 순수 판매 상품만 조회 (부품/기타/리퍼브 제외)
- G-code와 F-code 매핑: `CONCAT('F', SUBSTRING(G-code, 2))` — 재고 합산 시 필수
- 비정규화 테이블: category_name, brand_name, goods_name, option_name이 직접 저장됨
  - 이름 변경 시 erp_goods/erp_option과 erp_product_info 양쪽 동시 업데이트 필요
- safe_ea(안전재고)는 발주 알림 기준값으로 사용
- currency 컬럼은 수입 원가 환산 시 참조 (USD → KRW)

### 자주 쓰는 쿼리 패턴
- 판매 상품 목록 조회: `SELECT product_code, brand_name, goods_name, option_name FROM erp_product_info WHERE category_code = 'G' AND sell_status = 'Y' AND type = 'G'`
- 상품 코드로 단건 조회: `SELECT * FROM erp_product_info WHERE product_code = #{productCode}`
- 바코드로 조회: `SELECT product_code, brand_name, goods_name, option_name FROM erp_product_info WHERE barcode = #{barcode}`
- 원가 분석 (LATERAL JOIN): `SELECT pi.product_code, pi.msrp, pi.actual_price, lat.buying_krw FROM erp_product_info pi LEFT JOIN LATERAL (SELECT s.buying_price AS buying_krw FROM erp_stock s JOIN erp_order_item oi ON s.item_seq = oi.item_seq WHERE s.product_code = pi.product_code AND s.buying_price > 0 AND oi.cur2krw > 100 ORDER BY s.import_date DESC LIMIT 1) lat ON TRUE WHERE pi.brand_code = #{brandCode} AND pi.sell_status = 'Y'`
- 재고 현황 조회 (G+F 합산): `SELECT g.productCode, g.ea + IFNULL(f.ea, 0) AS total_stock FROM erp_stock_realtime g LEFT JOIN erp_stock_realtime f ON CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode WHERE g.productCode LIKE 'G-%'`

---

## erp_product_details (ERP 제품 상세 스펙)

summary: 상품의 상세 사양을 키-값 쌍으로 관리하는 테이블. 하나의 상품에 여러 개의 상세 항목(제목-내용)을 등록할 수 있다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| product_code | VARCHAR(30) | NOT NULL | 상품 코드 | 'G-0001-0025-0001' |
| title | VARCHAR(100) | NOT NULL | 제목 (스펙 항목명) | '연결방식', '배터리' |
| content | VARCHAR(255) | NOT NULL | 내용 (스펙 값) | 'Bluetooth 5.1', '4000mAh' |

### 관계
- erp_product_details.product_code → erp_product_info.product_code (N:1): 상세 항목은 하나의 상품에 속함

### 비즈니스 맥락
- 상품 상세 페이지에서 스펙 테이블 형태로 표시되는 정보
- 상품별 CRUD: 조회/등록/수정/삭제 모두 product_code 기준

### 자주 쓰는 쿼리 패턴
- 상품 상세 스펙 조회: `SELECT * FROM erp_product_details WHERE product_code = #{product_code}`

---

## erp_product_image (ERP 제품 이미지)

summary: 상품의 이미지 정보(제목, URL)를 관리하는 테이블. 하나의 상품에 여러 이미지를 등록할 수 있다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| product_code | VARCHAR(30) | NOT NULL | 상품 코드 | 'G-0001-0025-0001' |
| title | VARCHAR(100) | NOT NULL | 이미지 제목 | '메인 이미지', '측면' |
| url | VARCHAR(100) | NOT NULL | 이미지 URL | '/images/product/k2pro_main.jpg' |

### 관계
- erp_product_image.product_code → erp_product_info.product_code (N:1): 이미지는 하나의 상품에 속함

### 비즈니스 맥락
- 상품 상세 페이지에서 표시되는 이미지 목록
- CRUD: 등록/수정/삭제 모두 개별 seq 기준, 조회는 product_code 기준

### 자주 쓰는 쿼리 패턴
- 상품 이미지 목록 조회: `SELECT * FROM erp_product_image WHERE product_code = #{product_code}`

---

## erp_product_components (ERP 제품 구성품)

summary: 상품의 구성품 목록을 관리하는 테이블. 패키지 상품 등에서 포함되는 개별 구성품을 기록한다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| product_code | VARCHAR(30) | NOT NULL | 상품 코드 | 'G-0001-0025-0001' |
| components | VARCHAR(255) | NOT NULL | 구성품 | '키보드 본체', 'USB-C 케이블', 'Type-A 어댑터' |

### 관계
- erp_product_components.product_code → erp_product_info.product_code (N:1): 구성품은 하나의 상품에 속함

### 비즈니스 맥락
- 상품 등록 시 구성품을 일괄 삭제 후 재등록하는 패턴 (DELETE → INSERT)
- 키보드 패키지 등에서 포함 구성품을 명시적으로 관리

### 자주 쓰는 쿼리 패턴
- 구성품 목록 조회: `SELECT * FROM erp_product_components WHERE product_code = #{product_code}`
- 구성품 일괄 교체: `DELETE FROM erp_product_components WHERE product_code = #{product_code}` 후 INSERT

---

## erp_product_qc (ERP 제품 QC 정보)

summary: 상품의 품질 검수(QC) 내용을 관리하는 테이블. product_code가 PK이므로 상품당 하나의 QC 기준을 저장하며, UPSERT 패턴으로 관리된다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | VARCHAR(100) | PK | 상품코드 | 'G-0001-0025-0001' |
| contents | MEDIUMTEXT | NULL | QC 내용 | '외관 검사, 키 스위치 테스트...' |
| regist_date | DATETIME | NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | '2024-03-15 14:00:00' |

### 관계
- erp_product_qc.product_code → erp_product_info.product_code (1:1): QC 정보는 상품당 하나

### 비즈니스 맥락
- 입고 검수 시 참조되는 QC 기준 문서
- INSERT ON DUPLICATE KEY UPDATE 패턴으로 등록/수정 통합 처리

### 자주 쓰는 쿼리 패턴
- QC 내용 조회: `SELECT contents FROM erp_product_qc WHERE product_code = #{productCode}`
- QC 등록/수정 (UPSERT): `INSERT INTO erp_product_qc (product_code, contents) VALUES (#{productCode}, #{contents}) ON DUPLICATE KEY UPDATE contents = VALUES(contents)`

---

## erp_channel_list (ERP 판매 채널 목록)

summary: 상품을 판매하는 채널(파트너/마켓플레이스) 목록을 관리하는 테이블. 채널별 가격 관리의 기준이 된다.
domain: product
related_tables: erp_channel_price

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| partner_code | VARCHAR(20) | NOT NULL | 채널 코드 (파트너 코드) | 'ptn0010', 'ptn0091' |
| partner_name | VARCHAR(100) | NOT NULL | 채널 명 | '네이버 스마트스토어', '쿠팡' |

### 관계
- erp_channel_list.partner_code → erp_channel_price.partner_code (1:N): 채널에 여러 상품 가격 등록

### 비즈니스 맥락
- 채널별 가격표(Channel Price List) 관리의 기준 테이블
- 채널 목록을 동적으로 조회하여 피벗 형태의 가격표를 생성

### 자주 쓰는 쿼리 패턴
- 전체 채널 목록: `SELECT * FROM erp_channel_list`

---

## erp_channel_price (ERP 채널별 상품 가격)

summary: 각 판매 채널(마켓플레이스)에서의 상품별 판매 가격을 관리하는 테이블. 동일 상품이 채널마다 다른 가격으로 판매될 수 있다.
domain: product
related_tables: erp_channel_list, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| product_code | VARCHAR(20) | NOT NULL, UNIQUE(복합) | 상품 코드 | 'G-0001-0025-0001' |
| partner_code | VARCHAR(20) | NOT NULL, UNIQUE(복합) | 채널 코드 | 'ptn0010' |
| channel_code | VARCHAR(100) | NOT NULL, UNIQUE(복합) | 채널 상품 코드 | 'NAVER-12345' |
| channel_price | INT | NOT NULL | 채널 가격 | 129000 |

### 관계
- erp_channel_price.(product_code, partner_code, channel_code) UNIQUE: 복합 유니크 키
- erp_channel_price.product_code → erp_product_info.product_code (N:1): 가격은 하나의 상품에 속함
- erp_channel_price.partner_code → erp_channel_list.partner_code (N:1): 가격은 하나의 채널에 속함

### 비즈니스 맥락
- INSERT ON DUPLICATE KEY UPDATE 패턴으로 upsert 처리
- 채널별 가격표는 동적 피벗 쿼리로 생성 (GROUP_CONCAT + CASE WHEN)
- category_code = 'G' 상품만 채널 가격 관리 대상

### 자주 쓰는 쿼리 패턴
- 채널별 가격 피벗: `SELECT p_i.product_code, brand_name, goods_name, IFNULL(GROUP_CONCAT(CASE WHEN c.partner_code = #{partnerCode} THEN c.channel_price END), 0) AS channel_price FROM erp_product_info p_i LEFT JOIN erp_channel_price c ON p_i.product_code = c.product_code WHERE category_code = 'G' GROUP BY p_i.product_code`
- 가격 UPSERT: `INSERT INTO erp_channel_price (product_code, partner_code, channel_code, channel_price) VALUES (...) ON DUPLICATE KEY UPDATE channel_price = VALUES(channel_price)`

---

## erp_discounted_products (ERP 할인 상품 가격)

summary: 상품의 할인가 이력을 관리하는 테이블. 동일 상품에 여러 할인가가 시점별로 등록될 수 있으며, 가장 최신(seq DESC) 레코드가 현재 할인가이다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 번호 | 1, 2, 3 |
| product_code | VARCHAR(30) | NULL | 상품코드 | 'G-0001-0025-0001' |
| price | INT | NULL | 할인가 | 109000 |
| description | VARCHAR(250) | NULL | 비고 | '연말 프로모션' |
| regist_date | DATETIME | NULL | 등록일시 | '2024-12-01 10:00:00' |

### 관계
- erp_discounted_products.product_code → erp_product_info.product_code (N:1): 할인가는 하나의 상품에 속함

### 비즈니스 맥락
- 이력성 테이블 — 최신 레코드가 현재 적용 할인가 (ORDER BY seq DESC LIMIT 1)
- 가격 관리 화면에서 할인가 입력 시 INSERT로 신규 레코드 추가
- 가격 목록 조회 시 LATERAL JOIN으로 최신 할인가를 조회

### 자주 쓰는 쿼리 패턴
- 최신 할인가 조회 (LATERAL): `LEFT JOIN LATERAL (SELECT price AS discountedPrice FROM erp_discounted_products dp WHERE dp.product_code = product.product_code ORDER BY seq DESC LIMIT 1) discount_info ON TRUE`

---

## eflexs_product (eflexs 외부 ERP 상품 매핑)

summary: 외부 ERP 시스템(eflexs)의 상품명(item_desc)과 내부 G-code(product_code)를 매핑하는 테이블. eflexs 연동 시 상품 식별에 사용된다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 시퀀스 | 1, 2, 3 |
| item_desc | VARCHAR(255) | NULL | eflexs 상품명 | 'Keychron K2 Pro Red' |
| product_code | VARCHAR(200) | NOT NULL, UNIQUE | 내부 상품 코드 (G-code) | 'G-0001-0025-0001' |
| created_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | '2024-01-15 10:00:00' |
| modified_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | '2024-06-01 12:00:00' |
| member_id | VARCHAR(30) | NULL | 생성자 | 'admin01' |

### 관계
- eflexs_product.product_code → erp_product_info.product_code (논리적 1:1): eflexs 상품은 내부 상품에 매핑

### 비즈니스 맥락
- 외부 ERP(eflexs) 시스템과의 상품 연동 브릿지 테이블
- 관리자가 수동으로 매핑을 등록/삭제하는 방식
- foreach batch insert로 일괄 등록 지원

### 자주 쓰는 쿼리 패턴
- 전체 매핑 조회: `SELECT seq, product_code, item_desc FROM eflexs_product`
- 매핑 삭제: `DELETE FROM eflexs_product WHERE seq = #{seq}`

---

## eflux_recv_stock (eflexs 입고 재고)

summary: 외부 ERP(eflexs)로부터 수신한 실시간 재고 현황을 저장하는 테이블. 품목코드(item_cd)별 재고 수량, 가용 수량, 보류 수량 등을 관리한다.
domain: product
related_tables: erp_item_code, eflexs_product

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | PK, AUTO_INCREMENT | 시퀀스 | 1, 2, 3 |
| item_cd | VARCHAR(255) | NOT NULL, UNIQUE | 품목코드 (eflexs) | 'ITEM-001' |
| item_qty | INT | NOT NULL, DEFAULT 0 | 총 재고수량 | 100 |
| item_nm | VARCHAR(500) | NOT NULL | 품목명 | 'Keychron K2 Pro Red' |
| prcs_qty | INT | NOT NULL, DEFAULT 0 | 처리 수량 | 5 |
| free_qty | INT | NOT NULL, DEFAULT 0 | 가용 수량 | 90 |
| hld_qty | INT | NOT NULL, DEFAULT 0 | 보류 수량 | 5 |
| bad_qty | INT | NOT NULL, DEFAULT 0 | 불량 수량 | 0 |
| reg_date | DATETIME | NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | '2024-06-01 10:00:00' |
| modified_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | '2024-06-01 12:00:00' |

### 관계
- eflux_recv_stock.item_cd → erp_item_code.code (논리적 1:1): eflexs 품목코드와 내부 코드 매핑

### 비즈니스 맥락
- eflexs 외부 ERP에서 주기적으로 동기화되는 재고 스냅샷
- item_cd가 UNIQUE이므로 품목당 하나의 최신 재고 레코드 유지
- erp_item_code를 통해 내부 G-code로 변환하여 재고 통합 관리

### 자주 쓰는 쿼리 패턴
- 전체 외부 재고 조회: `SELECT * FROM eflux_recv_stock`

---

## TBNWS_ADMIN_eflux_recv_stock (eflexs 입고 재고 백업)

summary: eflux_recv_stock의 백업/스냅샷 테이블로, 동일한 구조이나 PK 제약조건이 없다. 데이터 이력 보관 용도로 사용된다.
domain: product
related_tables: eflux_recv_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | INT UNSIGNED | NULL | 시퀀스 (원본 참조) | 1, 2, 3 |
| item_cd | VARCHAR(255) | NULL | 품목코드 | 'ITEM-001' |
| item_qty | INT | NULL | 총 재고수량 | 100 |
| item_nm | VARCHAR(500) | NULL | 품목명 | 'Keychron K2 Pro Red' |
| prcs_qty | INT | NULL | 처리 수량 | 5 |
| free_qty | INT | NULL | 가용 수량 | 90 |
| hld_qty | INT | NULL | 보류 수량 | 5 |
| bad_qty | INT | NULL | 불량 수량 | 0 |
| reg_date | DATETIME | NULL | 등록일시 | '2024-06-01 10:00:00' |
| modified_at | TIMESTAMP | NULL | 수정일시 | '2024-06-01 12:00:00' |

### 관계
- TBNWS_ADMIN_eflux_recv_stock은 eflux_recv_stock의 스냅샷 복제본

### 비즈니스 맥락
- 모든 컬럼이 NULL 허용이며 PK/UNIQUE 제약조건이 없음 — 이력 보관 목적
- eflux_recv_stock과 동일 구조로, 데이터 동기화 전 백업으로 활용

### 자주 쓰는 쿼리 패턴
- 이력 조회: `SELECT * FROM TBNWS_ADMIN_eflux_recv_stock`

---

## erp_goods_keyword (ERP 상품 키워드 매핑)

summary: 출고 시 사용되는 상품 키워드(약식 명칭)를 내부 G-code에 매핑하는 테이블. 출고 설명(export_description)에서 상품을 자동 식별하는 데 사용된다.
domain: product
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| keyword_seq | INT UNSIGNED | PK, AUTO_INCREMENT | 키워드 시퀀스 | 1, 2, 3 |
| goods_keyword | VARCHAR(300) | NOT NULL, DEFAULT '' | 상품 키워드 | 'K2프로 레드', 'Q1맥스' |
| category_code | CHAR | NOT NULL, DEFAULT '' | 카테고리 코드 | 'G' |
| brand_code | CHAR(7) | NOT NULL, DEFAULT '' | 브랜드 코드 | '0001' |
| goods_code | CHAR(6) | NOT NULL, DEFAULT '' | 상품 코드 | '0025' |
| option_code | CHAR(7) | NULL | 옵션 코드 | '0001', NULL |
| regist_date | DATETIME | NULL, DEFAULT NOW() | 등록일시 | '2024-03-15 10:00:00' |

### 관계
- erp_goods_keyword.(category_code, brand_code, goods_code, option_code) → erp_product_info.(category_code, brand_code, goods_code, option_code) (논리적 N:1): 키워드는 하나의 상품에 매핑

### 비즈니스 맥락
- 출고 설명에 포함된 약식 상품명(키워드)을 정식 G-code로 변환
- 키워드 목록 조회 시 erp_product_info와 JOIN하여 정식 상품명도 함께 표시
- sell_status = 'Y' (판매중) 상품의 키워드만 활성 조회
- 하나의 상품에 여러 키워드를 등록할 수 있음 (약칭, 별명 등)

### 자주 쓰는 쿼리 패턴
- 활성 키워드 목록: `SELECT keyword.*, CONCAT('[', brand_name, ']', goods_name, IF(option_name IS NULL, '', CONCAT('(', option_name, ')'))) AS keyword_product_name FROM erp_goods_keyword keyword LEFT JOIN erp_product_info product_info ON product_info.product_code = CONCAT(keyword.category_code, '-', keyword.brand_code, '-', keyword.goods_code, IF(keyword.option_code IS NULL, '', CONCAT('-', keyword.option_code))) WHERE sell_status = 'Y'`
- CJ 코드 목록: `SELECT CONCAT('F', SUBSTRING(product_code, 2)) AS product_code, cj_code FROM erp_product_info WHERE cj_code != '' AND category_code = 'G'`

---

## currency_exchange_rate (환율 정보)

summary: 일별 환율 정보를 저장하는 테이블. 수입 원가 정산 시 USD → KRW 등 환율 변환에 사용되며, 통화+날짜 기준 유니크 제약이 있다.
domain: product
related_tables: erp_product_info, erp_order_item

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | BIGINT UNSIGNED | PK, AUTO_INCREMENT | 시퀀스 | 1, 2, 3 |
| rate | DECIMAL(10,4) | NULL | 환율 | 1350.5000 |
| currency | VARCHAR(20) | NOT NULL, UNIQUE(복합) | 통화 | 'USD', 'EUR', 'CNY' |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | '2024-06-01 09:00:00' |
| modified_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | '2024-06-01 12:00:00' |
| created_date | DATE | STORED GENERATED, UNIQUE(복합) | 생성일자 (created_at에서 자동 파생) | '2024-06-01' |

### 관계
- currency_exchange_rate.(currency, created_date) UNIQUE: 통화+날짜 유니크
- erp_order_item.cur2krw에 환율 값이 반영됨 (논리적 참조)

### 비즈니스 맥락
- 수입 상품 원가 정산에 사용: buying_price(USD) * 환율 = 원가(KRW)
- created_date는 STORED GENERATED 컬럼으로 created_at에서 자동 생성
- 환율 데이터가 없는 날짜는 가장 최근 환율을 사용하는 비즈니스 규칙
- cur2krw > 100 필터 필수 (1.00 등 비정상 환율 데이터 존재)

### 자주 쓰는 쿼리 패턴
- 최신 환율 조회: `SELECT rate FROM currency_exchange_rate WHERE currency = 'USD' ORDER BY created_at DESC LIMIT 1`
- 특정일 환율: `SELECT rate FROM currency_exchange_rate WHERE currency = #{currency} AND created_date = #{date}`

---

## md_keychron_sheet (키크론 상품 관리 시트)

summary: 네이버 브랜드스토어의 상품 코드(naver_product_code)와 내부 G-code(tobe_product_code)를 매핑하는 테이블. 마진 분석, 가격 비교, 재고 연동 등에 사용되며, 네이버/투비(TOBE) 양쪽 가격을 모두 관리한다.
domain: product
related_tables: erp_product_info, naver_smartstore_orders

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | 시퀀스 | 1, 2, 3 |
| should_register_property | TINYINT(1) | NULL, DEFAULT 0 | 속성 등록할지 여부 | 0, 1 |
| property_registration_status | TINYINT(1) | NULL | 속성 등록 상태 | 0, 1, NULL |
| product_category | VARCHAR(50) | NULL, DEFAULT '' | 상품 구분 | '키보드', '마우스', '액세서리' |
| naver_product_url | VARCHAR(500) | NULL, DEFAULT '' | 네이버 브랜드스토어 상품 링크 | 'https://smartstore.naver.com/...' |
| naver_marketing_url | VARCHAR(500) | NULL, DEFAULT '' | 네이버 마케팅 링크 | '' |
| group_product_code | VARCHAR(100) | NULL, DEFAULT '' | 그룹 상품 코드 | 'GRP-001' |
| naver_product_code | VARCHAR(100) | NULL, DEFAULT '', UNIQUE(복합) | 네이버 상품 코드 | '1234567890' |
| product_name | VARCHAR(500) | NULL, DEFAULT '' | 상품명 | 'Keychron K2 Pro' |
| internal_management_code | VARCHAR(100) | NULL, DEFAULT '', UNIQUE(복합) | 내부 관리 코드 | 'KC-K2PRO' |
| option_managed_code | VARCHAR(100) | NULL, DEFAULT '', UNIQUE(복합) | 옵션 코드 | 'RED-BROWN' |
| option_name | VARCHAR(255) | NULL, DEFAULT '' | 옵션명 (색상, 스위치 등) | 'Red / Gateron Brown' |
| sku_id | VARCHAR(100) | NULL, DEFAULT '' | SKU 고유 ID | 'SKU-12345' |
| sku_name | VARCHAR(500) | NULL, DEFAULT '' | SKU 상세명 | 'K2 Pro Red Gateron Brown' |
| fulfillment_g_code | VARCHAR(100) | NULL, DEFAULT '' | 풀필먼트 G코드 | 'G-0001-0025-0001' |
| fulfillment_item_name | VARCHAR(500) | NULL, DEFAULT '' | 풀필먼트 품목명 | '' |
| fulfillment_stock | INT | NULL, DEFAULT 0 | 풀필먼트 재고 수량 | 50 |
| tobe_product_code | VARCHAR(100) | NULL, DEFAULT '' | 투비 상품 코드 (G-code) | 'G-0001-0025-0001' |
| brand_name | VARCHAR(100) | NULL, DEFAULT '' | 브랜드명 | 'Keychron' |
| product_line | VARCHAR(255) | NULL, DEFAULT '' | 상품 라인 | 'K Pro Series' |
| product_option | VARCHAR(500) | NULL, DEFAULT '' | 상품 옵션 상세 | 'Red / Gateron Brown / RGB' |
| naver_list_price | DECIMAL | NULL, DEFAULT 0 | 네이버 정상가 | 139000 |
| naver_sale_price | DECIMAL | NULL, DEFAULT 0 | 네이버 판매가(상시가) | 129000 |
| discount_amount | DECIMAL | NULL, DEFAULT 0 | 할인 금액 | 10000 |
| tobe_list_price | DECIMAL | NULL, DEFAULT 0 | 투비 정상가 | 139000 |
| tobe_sale_price | DECIMAL | NULL, DEFAULT 0 | 투비 판매가(상시가) | 129000 |
| created_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | '2024-01-15 10:00:00' |
| updated_at | TIMESTAMP | NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | '2024-06-01 12:00:00' |

### 관계
- md_keychron_sheet.(internal_management_code, option_managed_code, naver_product_code) UNIQUE: 복합 유니크 키
- md_keychron_sheet.tobe_product_code → erp_product_info.product_code (논리적 N:1): G-code 매핑
- md_keychron_sheet.naver_product_code → naver_smartstore_orders.product_no (논리적 1:N): 네이버 주문 연동

### 비즈니스 맥락
- 네이버 브랜드스토어 상품 코드와 내부 G-code를 연결하는 핵심 매핑 테이블
- 마진 분석 시 naver_product_code로 매출 데이터를 조회하고, tobe_product_code로 원가를 조회
- naver_product_code → tobe_product_code 매핑이 1:N 관계일 수 있으므로 MIN(tobe_product_code) 등으로 대표 G-code 결정 필요
- COLLATE utf8mb4_unicode_ci 지정 필수 (naver_smartstore_orders와 collation mismatch 방지)
- 네이버/투비 양쪽 가격을 관리하여 가격 정합성 검증에 활용

### 자주 쓰는 쿼리 패턴
- G-code 매핑 (마진 분석): `WITH gcode_map AS (SELECT naver_product_code, MIN(tobe_product_code) AS gcode FROM md_keychron_sheet WHERE naver_product_code IS NOT NULL AND naver_product_code != '' AND tobe_product_code IS NOT NULL AND tobe_product_code != '' GROUP BY naver_product_code) SELECT nso.product_name, gm.gcode, SUM(nso.quantity) AS total_qty FROM naver_smartstore_orders nso LEFT JOIN gcode_map gm ON nso.product_no COLLATE utf8mb4_unicode_ci = gm.naver_product_code COLLATE utf8mb4_unicode_ci WHERE nso.account = #{account} GROUP BY nso.product_name, gm.gcode`

---

## brand_view (브랜드 뷰)

summary: erp_brand와 erp_category를 조인하여 카테고리명을 포함한 브랜드 정보를 제공하는 뷰. 카테고리 코드만으로는 알 수 없는 카테고리명을 함께 조회할 때 사용한다.
domain: product
related_tables: erp_brand, erp_category

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | - | 카테고리 코드 | 'G', 'F' |
| category_name | VARCHAR(10) | - | 카테고리명 | '일반상품' |
| brand_code | CHAR(10) | - | 브랜드 코드 | '0001' |
| brand_name | VARCHAR(100) | - | 브랜드명 | 'Keychron' |

### 비즈니스 맥락
- 뷰 정의: `erp_brand LEFT JOIN erp_category ON brand.category_code = category.category_code`
- erp_brand 단독으로는 category_name이 없으므로, 카테고리명이 필요한 조회에서 사용

### 자주 쓰는 쿼리 패턴
- 카테고리명 포함 브랜드 조회: `SELECT * FROM brand_view WHERE category_code = 'G'`

---

## goods_view (상품 뷰)

summary: erp_goods, erp_category, erp_brand를 조인하여 카테고리명/브랜드명을 포함한 상품 정보를 제공하는 뷰. 상품 레벨까지의 계층 정보를 비정규화하여 제공한다.
domain: product
related_tables: erp_goods, erp_category, erp_brand

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | - | 카테고리 코드 | 'G' |
| category_name | VARCHAR(10) | - | 카테고리명 | '일반상품' |
| brand_code | CHAR(10) | - | 브랜드 코드 | '0001' |
| brand_name | VARCHAR(100) | - | 브랜드명 | 'Keychron' |
| goods_code | CHAR(10) | - | 상품 코드 | '0025' |
| goods_name | VARCHAR(100) | - | 상품명 | 'K2 Pro' |

### 비즈니스 맥락
- 뷰 정의: `erp_goods LEFT JOIN erp_category ON category_code LEFT JOIN erp_brand ON (category_code, brand_code)`
- 카테고리/브랜드/상품 3단계 계층을 플랫하게 제공

### 자주 쓰는 쿼리 패턴
- 브랜드명 포함 상품 조회: `SELECT * FROM goods_view WHERE brand_code = '0001'`

---

## option_view (옵션 뷰)

summary: erp_option, erp_category, erp_brand, erp_goods를 조인하여 전체 계층 정보(카테고리~옵션)를 제공하는 뷰. 옵션 레벨까지의 모든 상위 계층 이름을 포함한다.
domain: product
related_tables: erp_option, erp_category, erp_brand, erp_goods

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| category_code | CHAR | - | 카테고리 코드 | 'G' |
| category_name | VARCHAR(10) | - | 카테고리명 | '일반상품' |
| brand_code | CHAR(10) | - | 브랜드 코드 | '0001' |
| brand_name | VARCHAR(100) | - | 브랜드명 | 'Keychron' |
| goods_code | CHAR(10) | - | 상품 코드 | '0025' |
| goods_name | VARCHAR(100) | - | 상품명 | 'K2 Pro' |
| option_code | CHAR(10) | - | 옵션 코드 | '0001' |
| option_name | VARCHAR(255) | - | 옵션 이름 | 'Red' |

### 비즈니스 맥락
- 뷰 정의: `erp_option LEFT JOIN erp_category LEFT JOIN erp_brand LEFT JOIN erp_goods` (4테이블 조인)
- 카테고리/브랜드/상품/옵션 4단계 전체 계층을 플랫하게 제공
- erp_option에 데이터가 있는 상품만 포함 (옵션이 없는 상품은 제외)

### 자주 쓰는 쿼리 패턴
- 전체 계층 포함 옵션 조회: `SELECT * FROM option_view WHERE brand_code = '0001' AND goods_code = '0025'`

---

## product_view (제품 통합 뷰)

summary: erp_product_info의 product_code를 파싱하여 erp_category/erp_brand/erp_goods/erp_option과 조인하는 통합 뷰. 상품 코드만으로 전체 계층 정보를 역참조할 수 있다.
domain: product
related_tables: erp_product_info, erp_category, erp_brand, erp_goods, erp_option

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | VARCHAR(30) | - | 상품 코드 (G-code) | 'G-0001-0025-0001' |
| category_code | CHAR | - | 카테고리 코드 (product_code 파싱) | 'G' |
| category_name | VARCHAR(10) | - | 카테고리명 | '일반상품' |
| brand_code | CHAR(10) | - | 브랜드 코드 (product_code 파싱) | '0001' |
| brand_name | VARCHAR(100) | - | 브랜드명 | 'Keychron' |
| goods_code | CHAR(10) | - | 상품 코드 (product_code 파싱) | '0025' |
| goods_name | VARCHAR(100) | - | 상품명 | 'K2 Pro' |
| option_code | CHAR(10) | - | 옵션 코드 (product_code 파싱) | '0001' |
| option_name | VARCHAR(255) | - | 옵션 이름 | 'Red' |

### 비즈니스 맥락
- 뷰 정의: erp_product_info.product_code를 SUBSTRING_INDEX로 파싱하여 category_code/brand_code/goods_code/option_code를 추출한 후, 각각의 마스터 테이블과 LEFT JOIN
- product_code 파싱 방식:
  - category_code = SUBSTRING_INDEX(product_code, '-', 1)
  - brand_code = SUBSTRING_INDEX(SUBSTRING_INDEX(product_code, '-', 2), '-', -1)
  - goods_code = SUBSTRING_INDEX(SUBSTRING_INDEX(product_code, '-', 3), '-', -1)
  - option_code = SUBSTRING_INDEX(SUBSTRING_INDEX(product_code, '-', 4), '-', -1)
- erp_product_info에 이미 비정규화된 이름 컬럼이 있으므로, 이 뷰는 마스터 테이블의 원본 이름을 확인할 때 주로 사용
- 모든 erp_product_info 레코드에 대해 뷰가 생성됨 (옵션 유무 무관)

### 자주 쓰는 쿼리 패턴
- 상품코드로 전체 계층 조회: `SELECT * FROM product_view WHERE product_code = 'G-0001-0025-0001'`
- 판매 상품 선택기: `SELECT product_name, product_code FROM erp_product_info WHERE category_code != 'F' AND sell_status NOT IN ('N', 'D') AND type = 'G'`
