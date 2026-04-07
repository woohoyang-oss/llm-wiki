# WMS / 부품 / 시리얼 / 풀필먼트 / 메시지 / 기타 테이블 스키마

---

## wms_warehouse (창고 마스터)

summary: WMS 창고 정보를 관리하는 마스터 테이블. GT(지티기어), GJ(광주), AS(AS센터) 등 창고 태그로 구분하며, 재고 이동(relocation) 및 적재 현황의 기준이 된다.
domain: wms
related_tables: wms_relocation, wms_relocation_item, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| warehouse_tag | varchar(10) | PK | 창고 단축 태그 | 'GT', 'GJ', 'AS' |
| warehouse_name | varchar(50) | NULL | 창고명 | '지티기어 본사 창고' |
| address | varchar(255) | NULL | 주소 | |
| description | varchar(255) | NULL | 비고 | |
| manager | varchar(100) | NULL | 관리자 | |
| manager_contact | varchar(50) | NULL | 관리자 연락처 | |

### 관계
- wms_relocation.from_warehouse_tag, wms_relocation.to_warehouse_tag가 이 테이블의 warehouse_tag를 참조
- erp_stock.warehouse_tag가 이 테이블을 참조하여 재고의 창고 위치를 나타냄

### 비즈니스 맥락
- 물류 창고 단위를 정의하는 마스터 테이블
- warehouse_tag는 짧은 코드로, 재고 이동(relocation) 시 출발/도착 창고를 지정하는 데 사용
- WMS 시스템에서 location(위치)과 결합하여 정밀한 적재 위치 관리

### 자주 쓰는 쿼리 패턴
```sql
-- 전체 창고 목록 조회
SELECT warehouse_tag, warehouse_name, address, description, manager, manager_contact
FROM wms_warehouse;

-- 특정 창고 정보 조회
SELECT warehouse_tag, warehouse_name, address, description, manager, manager_contact
FROM wms_warehouse
WHERE warehouse_tag = #{warehouseTag};
```

---

## wms_relocation (재고 이동 작업)

summary: 창고 간 재고 이동(로케이션) 작업의 헤더 정보를 관리하는 테이블. 출발 창고와 도착 창고, 작업 제목, 완료 일시를 기록한다.
domain: wms
related_tables: wms_warehouse, wms_relocation_item, wms_relocation_history, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| relocation_seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| title | varchar(255) | NULL | 이동 제목 | '2024년 3월 본사→AS센터 이동' |
| from_warehouse_tag | varchar(10) | NULL, INDEX | 출발 창고 | 'GT' |
| to_warehouse_tag | varchar(10) | NULL, INDEX | 도착 창고 | 'AS' |
| description | varchar(255) | NULL | 비고 | |
| complete_date | datetime | NULL | 완료일시 | NULL이면 미완료 |

### 관계
- from_warehouse_tag, to_warehouse_tag → wms_warehouse.warehouse_tag
- wms_relocation_item.relocation_seq → wms_relocation.relocation_seq (FK, CASCADE DELETE)
- wms_relocation_history.relocation_item_seq → wms_relocation_item을 통해 간접 연결

### 비즈니스 맥락
- 창고 간 재고 이동 작업 단위를 정의 (헤더 역할)
- complete_date가 NULL이면 아직 이동이 완료되지 않은 상태
- 완료 시 erp_stock의 product_code 접두사(G↔F)가 전환됨 (일반 창고 ↔ 풀필먼트 창고)

### 자주 쓰는 쿼리 패턴
```sql
-- 이동 작업 목록 조회 (창고명 포함)
SELECT r.relocation_seq, r.regist_date, r.title,
       r.from_warehouse_tag, fw.warehouse_name AS from_warehouse_name,
       r.to_warehouse_tag, tw.warehouse_name AS to_warehouse_name,
       r.description, r.complete_date
FROM wms_relocation r
INNER JOIN wms_warehouse fw ON r.from_warehouse_tag = fw.warehouse_tag
INNER JOIN wms_warehouse tw ON r.to_warehouse_tag = tw.warehouse_tag;

-- 이동 작업 완료 처리
UPDATE wms_relocation SET complete_date = CURRENT_TIMESTAMP
WHERE relocation_seq = #{relocationSeq};
```

---

## wms_relocation_item (재고 이동 품목)

summary: 재고 이동 작업의 개별 품목(제품 코드, 수량)을 관리하는 테이블. wms_relocation에 종속되며, 이동 대상 제품과 수량을 정의한다.
domain: wms
related_tables: wms_relocation, erp_product_info, erp_stock, wms_relocation_history

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| relocation_item_seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| relocation_seq | int unsigned | NOT NULL, FK | 로케이션 이동 번호 | |
| product_code | varchar(30) | NOT NULL, INDEX | 제품 코드 | 'G-0029-0001-0001' |
| ea | int unsigned | NOT NULL | 수량 | |
| import_date | datetime | NULL | 입고일(완료 시 설정) | |

### 관계
- relocation_seq → wms_relocation.relocation_seq (FK, ON UPDATE CASCADE, ON DELETE CASCADE)
- product_code → erp_product_info.product_code (조인용)
- erp_stock.relocation_item_seq가 이 테이블의 relocation_item_seq를 참조

### 비즈니스 맥락
- 이동 작업 내에서 어떤 제품을 몇 개 이동할지 지정
- erp_stock 테이블의 relocation_item_seq가 설정되면 해당 재고가 이동 대상으로 잠김
- import_date가 설정되면 해당 품목의 이동이 완료된 상태
- 완료 시 product_code 접두사가 G↔F로 전환 (일반↔풀필먼트)

### 자주 쓰는 쿼리 패턴
```sql
-- 이동 작업의 품목 목록 조회 (제품 정보 포함)
SELECT item.relocation_item_seq, item.product_code, item.ea, item.import_date,
       product.category_name, product.brand_name, product.goods_name, product.option_name
FROM wms_relocation_item AS item
LEFT JOIN erp_product_info AS product ON item.product_code = product.product_code
WHERE item.relocation_seq = #{relocationSeq};

-- 미완료 품목 수 확인
SELECT COUNT(*) FROM wms_relocation_item
WHERE relocation_seq = #{relocationSeq} AND import_date IS NULL;
```

---

## wms_relocation_history (재고 이동 히스토리)

summary: 재고 이동 시 개별 재고(stock)의 이전 상태를 기록하는 히스토리 테이블. 이동 전 product_code를 보존하여 추적 가능하게 한다.
domain: wms
related_tables: wms_relocation_item, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint unsigned | PK, AUTO_INCREMENT | 번호 | |
| stock_seq | bigint | NULL | 재고번호 | |
| relocation_item_seq | bigint | NULL | 로케이션 번호 | |
| before_product_code | varchar(255) | NULL | 기존 상품 코드 | 'G-0029-0001-0001' |
| created_at | timestamp | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 발생시기 | |
| update_date | datetime | NULL, ON UPDATE CURRENT_TIMESTAMP | 수정시기 | |

### 관계
- stock_seq → erp_stock.stock_seq (논리적 참조)
- relocation_item_seq → wms_relocation_item.relocation_item_seq (논리적 참조)

### 비즈니스 맥락
- 재고 이동 완료 시 각 stock의 이전 product_code를 기록하여 이력 추적 가능
- G→F 또는 F→G 코드 전환 전의 원래 코드가 before_product_code에 저장

### 자주 쓰는 쿼리 패턴
```sql
-- 이동 히스토리 벌크 삽입
INSERT INTO wms_relocation_history(stock_seq, relocation_item_seq, before_product_code)
VALUES (#{stock_seq}, #{relocation_item_seq}, #{before_product_code});
```

---

## erp_parts (AS 부품)

summary: AS 수리에 사용되는 부품의 개별 재고를 관리하는 테이블. 부품 코드, 시리얼 번호, 보관 위치, 상태, 사용 여부 등을 기록하며, 대여(loaner) 기능도 지원한다.
domain: parts
related_tables: erp_parts_info, erp_parts_location, erp_parts_status, erp_parts_loaner, erp_product_info, erp_stock, as_material_parts_rel, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| product_code | varchar(30) | NOT NULL | 상품 코드 | 'G-0029-0001-0001' |
| parts_code | varchar(10) | NULL | 부품 코드 | |
| stock_seq | int unsigned | NULL, INDEX | 원 상품 재고번호 | |
| serial_num | varchar(50) | NULL | 시리얼 번호 | |
| component | varchar(255) | NULL | 구성 | |
| status | varchar(255) | NULL | 상태 (erp_parts_status.seq 참조) | |
| location | varchar(255) | NULL | 보관 장소 (erp_parts_location.seq 참조) | |
| box_num | varchar(50) | NULL | 박스 번호 | |
| description | text | NULL | 비고 | |
| import_date | date | NULL | 입고일자 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록 일시 | |
| regist_member_id | int unsigned | NOT NULL, INDEX | 등록자 member id | |
| use_flag | char | NOT NULL, DEFAULT 'N' | 사용 여부 | 'Y', 'N' |
| as_seq_list | varchar(100) | NULL, DEFAULT ',' | AS 사용 리스트 | ',123,456,' |
| loanable_flag | char | NOT NULL, DEFAULT 'N' | 임대품 사용 가능 여부 | 'Y', 'N' |

### 관계
- product_code → erp_product_info.product_code (조인용, 제품 정보)
- parts_code + product_code → erp_parts_info (부품명 조회)
- status → erp_parts_status.seq (상태명 조회)
- location → erp_parts_location.seq (위치명 조회)
- stock_seq → erp_stock.stock_seq (원 재고)
- regist_member_id → member.id
- erp_parts_loaner.parts_seq → erp_parts.seq (대여 기록)
- as_material_parts_rel.parts_seq → erp_parts.seq (AS에서 사용된 부품)

### 비즈니스 맥락
- AS 수리 시 사용되는 개별 부품 단위 관리
- use_flag로 사용/미사용 구분, as_seq_list에 사용된 AS 번호 이력 보관
- loanable_flag='Y'인 부품은 고객에게 임대 가능 (erp_parts_loaner와 연동)
- status, location은 각각 erp_parts_status, erp_parts_location의 seq를 참조하는 코드형 필드

### 자주 쓰는 쿼리 패턴
```sql
-- 부품 목록 조회 (상태/위치명 포함)
SELECT p.seq, p.product_code, i.product_name, p.serial_num, p.parts_code,
       p_i.parts_name, p_s.status, p.status AS status_code,
       p_l.location, p.location AS location_code,
       p.box_num, p.description, p.import_date, p.regist_date, p.use_flag, p.as_seq_list
FROM erp_parts AS p
LEFT JOIN erp_product_info AS i ON p.product_code = i.product_code
LEFT JOIN erp_parts_info AS p_i ON p.product_code = p_i.product_code AND p.parts_code = p_i.parts_code
LEFT JOIN erp_parts_location AS p_l ON p.location = p_l.seq
LEFT JOIN erp_parts_status AS p_s ON p.status = p_s.seq;

-- 대여 가능 부품 목록 조회
SELECT parts.seq AS parts_seq, parts.product_code, parts.serial_num, ...
FROM erp_parts AS parts
LEFT JOIN erp_parts_loaner AS loaner ON parts.seq = loaner.parts_seq
WHERE parts.loanable_flag = 'Y';
```

---

## erp_parts_info (부품 정보 마스터)

summary: 제품별 부품 코드와 부품명을 정의하는 마스터 테이블. product_code + parts_code 조합으로 어떤 제품의 어떤 부품인지 식별한다.
domain: parts
related_tables: erp_parts, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | varchar(30) | NOT NULL | 상품 코드 | 'G-0029-0001-0001' |
| parts_code | varchar(10) | NOT NULL | 부품 코드 | |
| parts_name | varbinary(100) | NOT NULL | 부품명 | '키캡 세트', '스위치' |

### 관계
- product_code → erp_product_info.product_code (조인용)
- erp_parts 테이블에서 product_code + parts_code 조합으로 조인하여 부품명 조회

### 비즈니스 맥락
- 제품별로 어떤 부품이 있는지 정의하는 마스터 데이터
- product_code의 카테고리/브랜드/상품/옵션 구조를 파싱하여 계층적 부품 조회 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 제품의 부품 목록
SELECT product_code, parts_code, parts_name
FROM erp_parts_info
WHERE product_code = #{productCode};
```

---

## erp_parts_loaner (부품 대여 관리)

summary: AS 부품의 대여(임대) 기록을 관리하는 테이블. 대여자 정보, 대여 기간, 반납 상태를 추적한다.
domain: parts
related_tables: erp_parts, member, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| parts_seq | int unsigned | NULL, INDEX | 파츠 번호 | |
| borrower_name | varchar(30) | NULL | 대여자 | |
| borrower_phone | varchar(30) | NULL | 대여자 연락처 | |
| borrower_address | text | NULL | 대여자 주소 | |
| description | text | NULL | 비고 | |
| loan_date | datetime | NULL | 임대시작일시 | |
| expected_return_date | date | NULL | 임대종료예정일 | |
| return_date | datetime | NULL | 실제 임대종료일 | NULL이면 대여 중 |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| member_id | int unsigned | NOT NULL | 등록자 멤버 ID | |

### 관계
- parts_seq → erp_parts.seq (대여 대상 부품)
- member_id → member.id (등록자)

### 비즈니스 맥락
- loanable_flag='Y'인 부품에 대해 대여 기록 관리
- return_date가 NULL이면 현재 대여 중
- expected_return_date보다 현재 날짜가 크고 return_date가 NULL이면 연체 상태
- 한 부품에 여러 대여 기록 가능 (가장 최근 loan_date 기준으로 현재 대여 여부 판단)

### 자주 쓰는 쿼리 패턴
```sql
-- 대여 목록 조회 (부품/제품 정보 포함)
SELECT loaner.seq AS loaner_seq, loaner.parts_seq, loaner.borrower_name,
       loaner.borrower_phone, loaner.loan_date, loaner.expected_return_date,
       loaner.return_date, parts.product_code,
       product.brand_name, product.goods_name, product.option_name
FROM erp_parts_loaner AS loaner
LEFT JOIN erp_parts AS parts ON loaner.parts_seq = parts.seq
LEFT JOIN erp_product_info AS product ON parts.product_code = product.product_code
ORDER BY loaner.seq DESC;

-- 연체 대여 확인
SELECT * FROM erp_parts_loaner
WHERE return_date IS NULL AND expected_return_date <= CURRENT_TIMESTAMP;
```

---

## erp_parts_location (부품 보관 위치 마스터)

summary: AS 부품의 보관 위치를 정의하는 코드 마스터 테이블. erp_parts.location 필드에서 seq로 참조한다.
domain: parts
related_tables: erp_parts

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| location | varchar(100) | NOT NULL | 보관 위치 | 'AS센터 선반 A-1' |

### 관계
- erp_parts.location → erp_parts_location.seq

### 비즈니스 맥락
- 부품 보관 위치의 코드 테이블. 위치 이름의 일관성을 보장

### 자주 쓰는 쿼리 패턴
```sql
SELECT seq, location FROM erp_parts_location;
```

---

## erp_parts_status (부품 상태 마스터)

summary: AS 부품의 상태를 정의하는 코드 마스터 테이블. erp_parts.status 필드에서 seq로 참조한다.
domain: parts
related_tables: erp_parts

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| status | varchar(100) | NOT NULL | 상태 | '양품', '불량', '수리중' |

### 관계
- erp_parts.status → erp_parts_status.seq

### 비즈니스 맥락
- 부품의 현재 상태를 코드로 관리. 양품/불량/수리중 등

### 자주 쓰는 쿼리 패턴
```sql
SELECT seq, status FROM erp_parts_status;
```

---

## erp_serial (시리얼 번호 목록)

summary: 제품별 시리얼 번호를 관리하는 테이블. 정품등록 가능 여부를 제어하며, erp_serial_registration과 연동하여 고객 정품등록 정보를 추적한다.
domain: serial
related_tables: erp_product_info, erp_serial_registration, erp_serial_registration_manual

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| product_seq | int unsigned | NOT NULL, FK, UNIQUE(product_seq, serial_no) | 제품 정보 번호 | |
| serial_no | varchar(30) | NOT NULL | 시리얼 번호 | 'K2P-ABC-12345' |
| is_registrable | char | NOT NULL, DEFAULT 'Y' | 등록 가능 여부 | 'Y', 'N' |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| updated_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 마지막 수정일시 | |

### 관계
- product_seq → erp_product_info.product_seq (FK, ON UPDATE CASCADE)
- erp_serial_registration.serial_seq → erp_serial.seq (FK)
- UNIQUE(product_seq, serial_no): 동일 제품에 같은 시리얼 중복 방지

### 비즈니스 맥락
- 각 개별 제품에 고유 시리얼 번호를 부여하여 정품 인증 및 AS 이력 추적에 활용
- is_registrable='Y'면 고객이 정품등록 가능, 'N'이면 등록 불가(이미 등록됨 등)
- erp_serial_registration과 LEFT JOIN하여 등록 여부 확인

### 자주 쓰는 쿼리 패턴
```sql
-- 시리얼 정보 + 정품등록 정보 통합 조회
SELECT s.seq AS serial_seq, s.product_seq, s.serial_no, s.is_registrable,
       p.product_code, p.brand_code, p.brand_name, p.goods_name, p.option_name,
       r.seq AS registration_seq, r.customer_name, r.customer_phone, r.purchased_at
FROM erp_serial AS s
LEFT JOIN erp_product_info AS p ON s.product_seq = p.product_seq
LEFT JOIN erp_serial_registration AS r ON s.seq = r.serial_seq
WHERE p.brand_code = #{brandCode} AND s.serial_no = #{serial};

-- 시리얼 검색 (페이지네이션)
SELECT s.seq AS serial_seq, s.serial_no, ...
FROM erp_serial AS s
LEFT JOIN erp_product_info AS p ON s.product_seq = p.product_seq
LEFT JOIN erp_serial_registration AS r ON s.seq = r.serial_seq
ORDER BY s.seq DESC LIMIT #{limit} OFFSET #{offset};
```

---

## erp_serial_registration (정품등록 정보)

summary: 고객의 제품 정품등록 정보를 관리하는 테이블. 시리얼 번호 기반으로 고객 정보, 구매 정보, 쿠폰 발급 내역을 기록한다.
domain: serial
related_tables: erp_serial, coupon_list, erp_serial_registration_manual

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| serial_seq | int unsigned | NULL, FK | 시리얼 번호 | |
| customer_name | varchar(30) | NULL | 고객명 | |
| customer_phone | varchar(30) | NULL | 전화번호 | |
| customer_email | varchar(255) | NULL | 이메일 주소 | |
| purchased_at | date | NULL | 구매일 | |
| purchase_channel | varchar(255) | NULL | 구매처 | '네이버', '쿠팡', 'GT기어' |
| is_marketing_agreed | char | NOT NULL, DEFAULT 'N' | 마케팅 동의 여부 | 'Y', 'N' |
| coupon_seq | bigint | NULL, FK | 정품등록 쿠폰 1 | |
| second_coupon_seq | bigint | NULL, FK | 정품등록 쿠폰 2 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| updated_at | datetime | NOT NULL, ON UPDATE CURRENT_TIMESTAMP | 마지막 수정일시 | |

### 관계
- serial_seq → erp_serial.seq (FK, ON UPDATE CASCADE, ON DELETE SET NULL)
- coupon_seq → coupon_list.seq (FK, 정품등록 쿠폰 1)
- second_coupon_seq → coupon_list.seq (FK, 정품등록 쿠폰 2)
- erp_serial_registration_manual.registration_seq → erp_serial_registration.seq (수동등록 승인 시 연결)

### 비즈니스 맥락
- 고객이 정품등록을 하면 시리얼에 고객 정보가 연결됨
- 정품등록 시 쿠폰이 발급될 수 있음 (coupon_seq, second_coupon_seq)
- 수동 정품등록(erp_serial_registration_manual) 승인 시에도 이 테이블에 레코드 생성

### 자주 쓰는 쿼리 패턴
```sql
-- 정품등록 고객 정보 조회
SELECT s.serial_no, r.customer_phone, r.customer_email, r.purchased_at,
       r.purchase_channel, p.brand_code, p.goods_name, p.option_name,
       c1.coupon AS coupon_1, c2.coupon AS coupon_2
FROM erp_serial_registration AS r
LEFT JOIN erp_serial AS s ON r.serial_seq = s.seq
LEFT JOIN erp_product_info AS p ON s.product_seq = p.product_seq
LEFT JOIN coupon_list AS c1 ON r.coupon_seq = c1.seq
LEFT JOIN coupon_list AS c2 ON r.second_coupon_seq = c2.seq
WHERE p.brand_code = #{brandCode};
```

---

## erp_serial_registration_manual (수동 정품등록 신청)

summary: 자동 시리얼 매칭이 불가능한 경우 고객이 수동으로 정품등록을 신청하는 테이블. 관리자의 승인/반려 워크플로를 지원한다.
domain: serial
related_tables: erp_serial_registration, erp_brand, erp_goods, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| brand_code | varchar(10) | NULL | 신청 브랜드 코드 | '0029' |
| goods_code | varchar(10) | NULL | 신청 상품 코드 | |
| option_model | varchar(50) | NULL | 고객이 입력한 옵션모델 값 | |
| serial_no | varchar(255) | NULL | 고객 입력 시리얼(검증 전) | |
| customer_name | varchar(30) | NULL | 신청 고객명 | |
| customer_phone | varchar(30) | NULL | 신청 고객 전화번호 | |
| customer_email | varchar(255) | NULL | 신청 고객 이메일 | |
| purchased_at | date | NULL | 구매일 | |
| purchased_channel | varchar(255) | NULL | 구매처 | |
| is_marketing_agreed | char | NOT NULL, DEFAULT 'N' | 마케팅 동의 여부 | 'Y', 'N' |
| serial_image_url | text | NULL | 시리얼 이미지 URL | |
| purchase_image_url | text | NULL | 구매 증빙 이미지 URL | |
| is_approved | char | NOT NULL, DEFAULT 'N' | 승인 상태 | 'N'(대기/반려), 'Y'(승인) |
| reviewed_by | int unsigned | NULL, FK | 검수자 member id | |
| reviewed_at | datetime | NULL | 검수일시 | |
| registration_seq | int unsigned | NULL, FK | 승인된 등록정보의 FK | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| updated_at | datetime | NOT NULL, ON UPDATE CURRENT_TIMESTAMP | 마지막 수정일시 | |

### 관계
- registration_seq → erp_serial_registration.seq (FK, 승인 시 연결)
- reviewed_by → member.id (FK, 검수자)

### 비즈니스 맥락
- 시리얼 자동 매칭 불가 시 고객이 이미지 증빙과 함께 수동 신청
- 상태 구분: is_approved='N' + reviewed_at IS NULL → 대기, is_approved='Y' → 승인, is_approved='N' + reviewed_at IS NOT NULL → 반려
- 승인 시 erp_serial_registration에 레코드 생성 후 registration_seq 연결

### 자주 쓰는 쿼리 패턴
```sql
-- 수동정품등록 목록 조회 (대기 건 우선 정렬)
SELECT m.seq, m.brand_code, m.serial_no, m.customer_name, m.is_approved,
       b.brand_name, g.goods_name
FROM erp_serial_registration_manual AS m
LEFT JOIN erp_brand AS b ON b.category_code = 'G' AND m.brand_code = b.brand_code
LEFT JOIN erp_goods AS g ON g.category_code = 'G' AND m.brand_code = g.brand_code AND m.goods_code = g.goods_code
ORDER BY CASE WHEN m.is_approved = 'N' AND m.reviewed_at IS NULL THEN 0 ELSE 1 END, m.seq DESC;

-- 대기 건수 조회
SELECT COUNT(*) FROM erp_serial_registration_manual
WHERE is_approved = 'N' AND reviewed_at IS NULL;
```

---

## erp_partner (거래처/채널 파트너)

summary: 판매 채널 및 거래처(파트너)를 관리하는 마스터 테이블. B2C/B2B 구분, 사방넷 몰 매핑, 각 브랜드별 URL 등을 관리한다.
domain: fulfillment
related_tables: erp_stock, message_adcall_item, tbnws_sabangnet_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| partner_code | char(7) | PK, DEFAULT 'ptn0000' | 파트너 코드 | 'ptn0015', 'ptn0091' |
| partner_name | varchar(150) | NOT NULL, DEFAULT '' | 파트너명 | '네이버', '쿠팡', 'GT기어' |
| partner_type | varchar(6) | NOT NULL, DEFAULT '' | 파트너 유형 | |
| partner_target | varchar(3) | NOT NULL, DEFAULT 'B2C' | B2B or B2C | 'B2C', 'B2B' |
| gtgear_url | varchar(250) | NULL | GTGear URL | |
| playpod_url | varchar(250) | NULL | PlayPod URL | |
| kapalapi_url | varchar(250) | NULL | Kapalapi URL | |
| purchase_history_url | varchar(250) | NULL | 주문내역 바로가기 링크 | |
| purchase_history_url_M | varchar(250) | NULL | 주문내역 바로가기 링크(모바일) | |
| sabangnet_mall_name | varchar(30) | NULL | 사방넷 몰 이름 | |
| is_use_yn | varchar(1) | NULL, DEFAULT 'N' | 채널 등록 여부 | 'Y', 'N' |

### 관계
- erp_stock.export_partner_code → erp_partner.partner_code (출고처)
- message_adcall_item.partner_code → erp_partner.partner_code (애드콜 대상 파트너)
- sabangnet_mall_name으로 사방넷 주문의 mall_id와 매핑

### 비즈니스 맥락
- 모든 판매 채널의 마스터 데이터 (네이버, 쿠팡, 자사몰 등)
- partner_target으로 B2C(일반 소비자 채널)와 B2B(기업 거래) 구분
- sabangnet_mall_name 필드를 통해 사방넷 주문의 mall_id와 파트너 코드를 매핑
- 출고 시 export_partner_code로 어느 채널에서 판매되었는지 추적

### 자주 쓰는 쿼리 패턴
```sql
-- 사방넷 mall_id로 파트너 코드 조회
SELECT partner_code FROM erp_partner WHERE sabangnet_mall_name = #{mallId};

-- 파트너 정보 조회
SELECT partner_code, partner_name, partner_target FROM erp_partner;
```

---

## task_list (업무 목록)

summary: 직원에게 할당되는 업무(태스크)의 목록을 관리하는 테이블. 제목, 내용, 우선순위, 완료 여부, 반복 여부 등을 기록한다.
domain: misc
related_tables: task_assignment, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| member_id | int | NULL | 직원 아이디 | |
| title | varchar(255) | NOT NULL | 제목 | |
| type | char | NOT NULL, DEFAULT 'L' | 타입(우선순위) | 'L'(Low), 'M'(Medium), 'H'(High) |
| content | text | NULL | 내용 | |
| url | text | NULL | 참고 URL | |
| url_title | text | NULL | URL 속성 | |
| is_completed | tinyint(1) | NULL, DEFAULT 0 | 완료 여부 | 0, 1 |
| start_datetime | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 시작일시 | |
| end_datetime | datetime | NULL | 종료일시 | |
| created_by | int | NULL | 등록 직원 아이디 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| is_deleted | tinyint(1) | NULL, DEFAULT 0 | 삭제 여부 | 0, 1 |
| is_completed_date | date | NULL | 완료 날짜 | |
| is_recurring | tinyint(1) | NULL, DEFAULT 0 | 반복 여부 | 0, 1 |

### 관계
- task_assignment.task_seq → task_list.seq (FK, 업무 할당)
- member_id → member.id (업무 담당자)
- created_by → member.id (업무 등록자)

### 비즈니스 맥락
- 내부 업무 관리 시스템의 기본 단위
- type 필드로 우선순위 구분 (L/M/H)
- is_deleted=1로 소프트 삭제 구현
- is_recurring으로 반복 업무 표시

### 자주 쓰는 쿼리 패턴
```sql
-- 활성 업무 목록 조회
SELECT seq, title, type, content, is_completed, start_datetime, end_datetime
FROM task_list
WHERE is_deleted = 0;
```

---

## task_assignment (업무 할당)

summary: 업무(task_list)를 개별 직원에게 할당하고 진행 상황을 추적하는 테이블. 한 업무에 여러 직원이 할당될 수 있다.
domain: misc
related_tables: task_list, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 할당 ID | |
| task_seq | int unsigned | NOT NULL, FK, INDEX | 업무 ID | |
| member_id | int unsigned | NOT NULL, FK, INDEX | 담당 직원 ID | |
| is_completed | tinyint(1) | NULL, DEFAULT 0 | 완료 여부 | 0, 1 |
| completed_at | datetime | NULL | 완료일시 | |
| task_note | text | NULL | 개인 할당 업무 노트 | |
| note | text | NULL | 개인별 진행상황 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 할당일시 | |
| is_deleted | tinyint(1) | NULL, DEFAULT 0 | 삭제 여부 | 0, 1 |
| is_completed_date | date | NULL | 완료날짜 | |

### 관계
- task_seq → task_list.seq (FK)
- member_id → member.id (FK)

### 비즈니스 맥락
- 하나의 업무에 여러 직원이 할당될 수 있으며, 각자의 완료 상태/노트를 개별 관리
- is_completed로 개인별 완료 상태, completed_at으로 완료 시점 기록

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 직원의 할당된 업무 목록
SELECT ta.seq, ta.task_seq, tl.title, ta.is_completed, ta.note
FROM task_assignment ta
JOIN task_list tl ON ta.task_seq = tl.seq
WHERE ta.member_id = #{memberId} AND ta.is_deleted = 0;
```

---

## tbnws_sabangnet_order (사방넷 주문 정보)

summary: 사방넷을 통해 수집된 멀티채널 주문 정보를 관리하는 테이블. 네이버, 쿠팡, 자사몰 등 다양한 채널의 주문이 통합 저장된다.
domain: fulfillment
related_tables: tbnws_sabangnet_preOrder, tbnws_sabangnet_fault_order, erp_partner, erp_stock, tbnws_combined_packaging

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | varchar(50) | PK | 사방넷 주문번호 | |
| mall_order_id | varchar(50) | NULL | 부주문번호 | |
| order_date | date | NULL | 주문일자 | |
| ord_confirm_date | date | NULL | 주문 확인일자 | |
| reg_date | datetime | NULL, INDEX | 수집일자 | |
| order_status | varchar(20) | NULL | 주문상태 | |
| order_gubun | char(10) | NULL | 주문구분 | |
| mall_product_id | varchar(50) | NULL | 상품코드 (쇼핑몰) | |
| model_no | varchar(150) | NULL | G코드 | |
| model_name | varchar(255) | NULL | 모델명 | |
| compayny_goods_cd | text | NULL | 자체상품코드 | |
| p_product_name | varchar(200) | NULL | 확정된 상품명 | |
| box_ea | int | NULL | 묶음 개수 | |
| sale_cnt | int | NULL | 수량 | |
| p_ea | int | NULL | EA(확정) | |
| user_id | varchar(50) | NULL | 주문자 아이디 | |
| user_name | varchar(50) | NULL | 주문자명 | |
| user_tel | varchar(50) | NULL | 주문자전화번호 | |
| user_cel | varchar(50) | NULL | 주문자핸드폰번호 | |
| user_email | varchar(100) | NULL | 주문자 이메일주소 | |
| receive_name | varchar(50) | NULL | 수취인명 | |
| receive_tel | varchar(50) | NULL | 수취인전화번호 | |
| receive_cel | varchar(50) | NULL | 수취인핸드폰번호 | |
| receive_email | varchar(100) | NULL | 수취인이메일주소 | |
| receive_zipcode | varchar(50) | NULL | 수취인우편번호 | |
| receive_addr | varchar(200) | NULL | 수취인주소 | |
| delv_msg | text | NULL | 배송 메세지1 | |
| delv_msg1 | varchar(200) | NULL | 배송 메세지2 | |
| delivery_id | varchar(20) | NULL | 택배사코드 | |
| hope_delv_date | date | NULL | 배송희망일자 | |
| pay_cost | int | NULL | 결제금액 | |
| sale_cost | int | NULL | 판매가 | |
| total_cost | int | NULL | 주문금액 | |
| delv_cost | int | NULL | 배송비(수집) | |
| mall_won_cost | int | NULL | 공급단가 | |
| mall_id | varchar(50) | NULL | 쇼핑몰 코드 | |
| invoice | varchar(255) | NULL | 송장번호 | |
| inv_send_msg | varchar(255) | NULL | 송장 발송 메시지 | |
| isEfluxRequest | char | NOT NULL, DEFAULT 'N' | 이플럭스 요청 여부 | 'Y', 'N' |
| efluxRequestDate | datetime | NULL | 이플럭스 요청일시 | |
| combined_packing_seq | varchar(255) | NULL | 합포 번호 | |
| isFreeGift | varchar(3) | NOT NULL, DEFAULT 'N' | 사은품 여부 | 'Y', 'N' |
| isStockDecreased | varchar(1) | NULL, DEFAULT 'N' | 재고 차감 여부 | 'Y', 'N' |

### 관계
- mall_id → erp_partner.sabangnet_mall_name (파트너 매핑)
- compayny_goods_cd → erp_product_info.product_code (제품 매핑)

### 비즈니스 맥락
- 사방넷을 통해 수집된 확정 주문 (출고 완료 또는 처리 완료 상태)
- tbnws_sabangnet_preOrder와 UNION하여 전체 주문 흐름 조회
- isFreeGift로 사은품 주문 구분
- isStockDecreased로 재고 차감 처리 여부 추적

### 자주 쓰는 쿼리 패턴
```sql
-- 주문 목록 조회 (완료 + 수집 통합)
SELECT idx, mall_order_id, p_product_name, mall_id, p_ea, order_date, reg_date,
       receive_name, invoice, '완료' AS status
FROM tbnws_sabangnet_order
UNION ALL
SELECT idx, mall_order_id, p_product_name, mall_id, p_ea, order_date, reg_date,
       receive_name, invoice, '수집' AS status
FROM tbnws_sabangnet_preOrder
WHERE reg_date > (SELECT MAX(reg_date) FROM tbnws_sabangnet_order)
ORDER BY reg_date DESC;
```

---

## tbnws_sabangnet_preOrder (사방넷 사전 주문)

summary: 사방넷에서 수집되었지만 아직 확정 처리되지 않은 사전 주문(수집 단계) 정보를 저장하는 테이블. 구조는 tbnws_sabangnet_order와 동일하다.
domain: fulfillment
related_tables: tbnws_sabangnet_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | varchar(50) | PK | 사방넷 주문번호 | |
| mall_order_id | varchar(50) | NULL | 부주문번호 | |
| order_date | date | NULL | 주문일자 | |
| ord_confirm_date | date | NULL | 주문 확인일자 | |
| reg_date | datetime | NULL | 수집일자 | |
| order_status | varchar(20) | NULL | 주문상태 | |
| order_gubun | char(10) | NULL | 주문구분 | |
| mall_product_id | varchar(50) | NULL | 상품코드 (쇼핑몰) | |
| model_no | varchar(150) | NULL | G코드 | |
| model_name | varchar(255) | NULL | 모델명 | |
| compayny_goods_cd | text | NULL | 자체상품코드 | |
| p_product_name | varchar(200) | NULL | 확정된 상품명 | |
| box_ea | int | NULL | 묶음 개수 | |
| sale_cnt | int | NULL | 수량 | |
| p_ea | int | NULL | EA(확정) | |
| user_id | varchar(50) | NULL | 주문자 아이디 | |
| user_name | varchar(50) | NULL | 주문자명 | |
| user_tel | varchar(50) | NULL | 주문자전화번호 | |
| user_cel | varchar(50) | NULL | 주문자핸드폰번호 | |
| user_email | varchar(100) | NULL | 주문자 이메일주소 | |
| receive_name | varchar(50) | NULL | 수취인명 | |
| receive_tel | varchar(50) | NULL | 수취인전화번호 | |
| receive_cel | varchar(50) | NULL | 수취인핸드폰번호 | |
| receive_email | varchar(100) | NULL | 수취인이메일주소 | |
| receive_zipcode | varchar(50) | NULL | 수취인우편번호 | |
| receive_addr | varchar(200) | NULL | 수취인주소 | |
| delv_msg | text | NULL | 배송 메세지1 | |
| delv_msg1 | varchar(200) | NULL | 배송 메세지2 | |
| delivery_id | varchar(20) | NULL | 택배사코드 | |
| hope_delv_date | date | NULL | 배송희망일자 | |
| pay_cost | int | NULL | 결제금액 | |
| sale_cost | int | NULL | 판매가 | |
| total_cost | int | NULL | 주문금액 | |
| delv_cost | int | NULL | 배송비(수집) | |
| mall_won_cost | int | NULL | 공급단가 | |
| mall_id | varchar(255) | NULL | 쇼핑몰 코드 | |
| invoice | varchar(255) | NULL | 송장번호 | |

### 관계
- tbnws_sabangnet_order와 동일 구조이며, 확정 후 tbnws_sabangnet_order로 이동

### 비즈니스 맥락
- 사방넷에서 수집된 미확정 주문이 저장되는 임시 테이블
- tbnws_sabangnet_order에 아직 없는 최신 수집 건을 UNION하여 전체 주문 흐름 조회

### 자주 쓰는 쿼리 패턴
```sql
-- 최신 수집 주문 조회 (아직 확정되지 않은 건)
SELECT * FROM tbnws_sabangnet_preOrder
WHERE reg_date > (SELECT MAX(reg_date) FROM tbnws_sabangnet_order)
ORDER BY reg_date DESC;
```

---

## tbnws_sabangnet_fault_order (사방넷 오류 주문)

summary: 사방넷 주문 처리 중 오류가 발생한 주문을 별도로 관리하는 테이블. 사방넷 미매핑, 이플럭스 미매핑, 재고 부족 등의 오류 유형을 기록한다.
domain: fulfillment
related_tables: tbnws_sabangnet_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | varchar(50) | PK | 사방넷 주문번호 | |
| mall_order_id | varchar(50) | NULL | 부주문번호 | |
| order_date | date | NULL | 주문일자 | |
| ord_confirm_date | date | NULL | 주문 확인일자 | |
| reg_date | datetime | NULL | 수집일자 | |
| order_status | varchar(20) | NULL | 주문상태 | |
| order_gubun | char(10) | NULL | 주문구분 | |
| mall_product_id | varchar(50) | NULL | 상품코드 (쇼핑몰) | |
| model_no | varchar(150) | NULL | G코드 | |
| model_name | varchar(255) | NULL | 모델명 | |
| compayny_goods_cd | text | NULL | 자체상품코드 | |
| p_product_name | varchar(200) | NULL | 확정된 상품명 | |
| box_ea | int | NULL | 묶음 개수 | |
| sale_cnt | int | NULL | 수량 | |
| p_ea | int | NULL | EA(확정) | |
| user_id | varchar(50) | NULL | 주문자 아이디 | |
| user_name | varchar(50) | NULL | 주문자명 | |
| user_tel | varchar(50) | NULL | 주문자전화번호 | |
| user_cel | varchar(50) | NULL | 주문자핸드폰번호 | |
| user_email | varchar(100) | NULL | 주문자 이메일주소 | |
| receive_name | varchar(50) | NULL | 수취인명 | |
| receive_tel | varchar(50) | NULL | 수취인전화번호 | |
| receive_cel | varchar(50) | NULL | 수취인핸드폰번호 | |
| receive_email | varchar(100) | NULL | 수취인이메일주소 | |
| receive_zipcode | varchar(50) | NULL | 수취인우편번호 | |
| receive_addr | varchar(200) | NULL | 수취인주소 | |
| delv_msg | text | NULL | 배송 메세지1 | |
| delv_msg1 | varchar(200) | NULL | 배송 메세지2 | |
| delivery_id | varchar(20) | NULL | 택배사코드 | |
| hope_delv_date | date | NULL | 배송희망일자 | |
| pay_cost | int | NULL | 결제금액 | |
| sale_cost | int | NULL | 판매가 | |
| total_cost | int | NULL | 주문금액 | |
| delv_cost | int | NULL | 배송비(수집) | |
| mall_won_cost | int | NULL | 공급단가 | |
| mall_id | varchar(255) | NULL | 쇼핑몰 코드 | |
| invoice | varchar(255) | NULL | 송장번호 | |
| error_code | varchar(10) | NULL, DEFAULT '' | 오류 코드 | 'S'(사방넷 미매핑), 'E'(이플럭스 미매핑), 'F'(재고부족), 'H'(수기등록) |
| error_reason | varchar(200) | NULL, DEFAULT '' | 오류 사유 | |

### 관계
- tbnws_sabangnet_order와 동일 구조 + error_code, error_reason 추가

### 비즈니스 맥락
- 정상 처리되지 못한 주문을 별도 관리하여 수동 처리 지원
- error_code: S=사방넷 상품 미매핑, E=이플럭스 미매핑, F=재고 부족, H=수기등록 필요

### 자주 쓰는 쿼리 패턴
```sql
-- 오류 주문 목록 조회
SELECT * FROM tbnws_sabangnet_fault_order ORDER BY reg_date DESC;
```

---

## tbnws_sabangnet_gtgear_order (사방넷 GT기어 주문)

summary: GT기어 브랜드에 특화된 사방넷 주문 정보 테이블. 합포 처리와 엑셀 다운로드 여부를 추가로 관리한다.
domain: fulfillment
related_tables: tbnws_sabangnet_order, tbnws_combined_packaging

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | varchar(50) | PK | 사방넷 주문번호 | |
| mall_order_id | varchar(50) | NULL | 부주문번호 | |
| order_date | date | NULL | 주문일자 | |
| ord_confirm_date | date | NULL | 주문 확인일자 | |
| reg_date | datetime | NULL, INDEX | 수집일자 | |
| order_status | varchar(20) | NULL | 주문상태 | |
| order_gubun | char(10) | NULL | 주문구분 | |
| mall_product_id | varchar(50) | NULL | 상품코드 (쇼핑몰) | |
| model_no | varchar(150) | NULL | G코드 | |
| model_name | varchar(255) | NULL | 모델명 | |
| compayny_goods_cd | text | NULL | 자체상품코드 | |
| p_product_name | varchar(200) | NULL | 확정된 상품명 | |
| box_ea | int | NULL | 묶음 개수 | |
| sale_cnt | int | NULL | 수량 | |
| p_ea | int | NULL | EA(확정) | |
| user_id | varchar(50) | NULL | 주문자 아이디 | |
| user_name | varchar(50) | NULL | 주문자명 | |
| user_tel | varchar(50) | NULL | 주문자전화번호 | |
| user_cel | varchar(50) | NULL | 주문자핸드폰번호 | |
| user_email | varchar(100) | NULL | 주문자 이메일주소 | |
| receive_name | varchar(50) | NULL | 수취인명 | |
| receive_tel | varchar(50) | NULL | 수취인전화번호 | |
| receive_cel | varchar(50) | NULL | 수취인핸드폰번호 | |
| receive_email | varchar(100) | NULL | 수취인이메일주소 | |
| receive_zipcode | varchar(50) | NULL | 수취인우편번호 | |
| receive_addr | varchar(200) | NULL | 수취인주소 | |
| delv_msg | text | NULL | 배송 메세지1 | |
| delv_msg1 | varchar(200) | NULL | 배송 메세지2 | |
| delivery_id | varchar(20) | NULL | 택배사코드 | |
| hope_delv_date | date | NULL | 배송희망일자 | |
| pay_cost | int | NULL | 결제금액 | |
| sale_cost | int | NULL | 판매가 | |
| total_cost | int | NULL | 주문금액 | |
| delv_cost | int | NULL | 배송비(수집) | |
| mall_won_cost | int | NULL | 공급단가 | |
| mall_id | varchar(255) | NULL | 쇼핑몰 코드 | |
| invoice | varchar(255) | NULL | 송장번호 | |
| combined_packing_seq | varchar(255) | NOT NULL, DEFAULT '' | 합포 번호 | |
| isDownExcel | char | NOT NULL, DEFAULT 'N' | 엑셀 다운로드 여부 | 'Y', 'N' |

### 관계
- combined_packing_seq → tbnws_combined_packaging (합포 주문 참조)

### 비즈니스 맥락
- GT기어 브랜드 전용 주문 테이블로, 합포 처리와 엑셀 다운로드 관리 기능 추가
- combined_packing_seq로 여러 주문을 하나의 배송으로 합포

### 자주 쓰는 쿼리 패턴
```sql
SELECT * FROM tbnws_sabangnet_gtgear_order ORDER BY reg_date DESC;
```

---

## tbnws_prelaunching_list (사전 출시 목록)

summary: 제품 사전 출시(프리런칭) 이벤트를 관리하는 테이블. 프리런칭 기간, 예약구매 기간, URL 등을 관리한다.
domain: misc
related_tables: tbnws_prelaunching_list_item, tbnws_prelaunching_object_item, tbnws_prelaunching_user

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_title | varchar(255) | NOT NULL | 프리런칭 제목 | |
| object | varchar(255) | NOT NULL, UNIQUE | 오브젝트(고유 식별자) | |
| start_date | datetime | NOT NULL | 시작 일자 | |
| end_date | datetime | NULL | 종료 일자 | |
| buy_start_date | datetime | NULL | 예약구매 시작일자 | |
| buy_end_date | datetime | NULL | 예약구매 종료일자 | |
| url | varchar(50) | NULL, DEFAULT '' | 프리런칭 단축 URL | |
| brand_code | char(7) | NOT NULL, DEFAULT '0029' | 프리런칭 메이커 브랜드코드 | |
| prelaunching_url | varchar(100) | NULL | 프리런칭 상품 URL | |
| prelaunching_memo | text | NULL | 메모 | |
| createdBy | varchar(100) | NOT NULL | 등록자 | |
| createdDate | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록 일자 | |
| lastModifiedBy | varchar(100) | NULL | 수정자 | |
| lastModifyDate | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수정 일자 | |

### 관계
- object → tbnws_prelaunching_object_item.object (프리런칭 옵션 항목)
- object → tbnws_prelaunching_user.object (프리런칭 참여 사용자)
- tbnws_prelaunching_list_item.prelaunching_seq → tbnws_prelaunching_list.seq

### 비즈니스 맥락
- 신제품 출시 전 사전 예약 이벤트를 관리
- object 필드가 고유 식별자로 하위 테이블과 연결됨
- start_date~end_date: 프리런칭 기간, buy_start_date~buy_end_date: 예약구매 기간

### 자주 쓰는 쿼리 패턴
```sql
SELECT seq, prelaunching_title, object, start_date, end_date, buy_start_date, buy_end_date
FROM tbnws_prelaunching_list
ORDER BY createdDate DESC;
```

---

## tbnws_prelaunching_list_item (사전 출시 아이템)

summary: 프리런칭 이벤트에 포함된 개별 제품 항목을 관리하는 테이블. 카테고리, 브랜드, 상품, 옵션 코드로 대상 제품을 지정한다.
domain: misc
related_tables: tbnws_prelaunching_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | AUTO_INCREMENT, INDEX | 프리런칭 아이템 번호 | |
| prelaunching_seq | int | NOT NULL | 프리런칭 번호 | |
| prelaunching_type | char(7) | NOT NULL, DEFAULT '' | 아이템 타입 | |
| category_code | char(7) | NOT NULL, DEFAULT '' | 카테고리 코드 | |
| brand_code | char(7) | NOT NULL, DEFAULT 'brd0000' | 브랜드 코드 | |
| goods_code | char(6) | NOT NULL, DEFAULT 'gs0000' | 상품 코드 | |
| option_code | char(7) | NULL | 옵션 코드 | |
| import_date | date | NULL | 입고 날짜 | |

### 관계
- prelaunching_seq → tbnws_prelaunching_list.seq

### 비즈니스 맥락
- 프리런칭에 포함되는 개별 제품(SKU) 정보

### 자주 쓰는 쿼리 패턴
```sql
SELECT * FROM tbnws_prelaunching_list_item WHERE prelaunching_seq = #{seq};
```

---

## tbnws_prelaunching_object_item (사전 출시 옵션 집계)

summary: 프리런칭 이벤트의 옵션별 신청 카운트를 관리하는 테이블. 키보드의 프레임/스위치 옵션 조합별 수량을 집계한다.
domain: misc
related_tables: tbnws_prelaunching_list, tbnws_prelaunching_user

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| object | varchar(50) | NOT NULL, INDEX | 프리런칭 오브젝트 | |
| option_1 | varchar(100) | NOT NULL | 프레임 | |
| option_2 | varchar(100) | NOT NULL | 스위치 | |
| count | int | NULL, DEFAULT 0 | 카운트 | |

### 관계
- object → tbnws_prelaunching_list.object

### 비즈니스 맥락
- 프리런칭에서 옵션 조합별(프레임+스위치) 신청 인원 집계
- 키보드 프리런칭에 특화된 구조 (option_1=프레임, option_2=스위치)

### 자주 쓰는 쿼리 패턴
```sql
SELECT object, option_1, option_2, count
FROM tbnws_prelaunching_object_item
WHERE object = #{object};
```

---

## tbnws_prelaunching_user (사전 출시 참여자)

summary: 프리런칭 이벤트에 참여한 고객 정보를 관리하는 테이블. 고객의 선택 옵션, 참여/취소 상태를 기록한다.
domain: misc
related_tables: tbnws_prelaunching_list, tbnws_prelaunching_object_item

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 번호 | |
| object | varchar(50) | NOT NULL | 프리런칭 오브젝트 | |
| name | varchar(50) | NULL, DEFAULT '' | 이름 | |
| email | varchar(100) | NULL | 이메일 | |
| phone | varchar(50) | NULL | 전화번호 | |
| keyboard_frame | varchar(50) | NULL, DEFAULT '' | 키보드 프레임 선택 | |
| keyboard_switch | varchar(50) | NULL, DEFAULT '' | 키보드 스위치 선택 | |
| prelaunching_date | datetime | NULL, DEFAULT CURRENT_TIMESTAMP | 참여일시 | |
| unprelaunching_flag | char | NULL, DEFAULT 'N' | 취소 여부 | 'Y', 'N' |
| unprelaunching_date | datetime | NULL | 취소일시 | |
| export_seq | int | NULL | 프리런칭 출고번호 | |
| prelaunching_flag | varchar(1) | NULL, DEFAULT 'N' | 프리런칭 확정 여부 | 'Y', 'N' |
| option_1 | varchar(50) | NULL, DEFAULT '' | 옵션1 | |
| option_2 | varchar(50) | NULL, DEFAULT '' | 옵션2 | |
| brand_code | char(7) | NOT NULL, DEFAULT 'brd0029' | 프리런칭 브랜드 코드 | |

### 관계
- object → tbnws_prelaunching_list.object
- UNIQUE(object, email, phone, keyboard_frame, keyboard_switch): 동일 조건 중복 참여 방지

### 비즈니스 맥락
- 프리런칭에 참여한 고객과 선택 옵션 기록
- unprelaunching_flag='Y'이면 참여 취소
- prelaunching_flag='Y'이면 프리런칭 확정 (구매 확정)

### 자주 쓰는 쿼리 패턴
```sql
-- 프리런칭 참여자 목록 조회
SELECT * FROM tbnws_prelaunching_user
WHERE object = #{object} AND unprelaunching_flag = 'N'
ORDER BY prelaunching_date;
```

---

## tbnws_certificate (인증서 관리)

summary: 직원의 인증서(재직증명서 등) 발급 신청과 승인을 관리하는 테이블.
domain: misc
related_tables: member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 신청번호 | |
| muid | varchar(100) | NULL | 신청자 UID | |
| apply_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 신청 일자 | |
| select_use | char | NOT NULL, DEFAULT '' | 용도 | |
| branch | char | NOT NULL, DEFAULT '' | 직장 | |
| approval_m_uid | varchar(30) | NULL | 승인 처리자 | |
| approval_date | datetime | NULL | 승인 일자 | |
| flag | char | NOT NULL, DEFAULT 'N' | 상태 플래그 | 'N'(미승인), 'Y'(승인) |

### 관계
- muid → member.uid (신청자)
- approval_m_uid → member.uid (승인자)

### 비즈니스 맥락
- 직원이 재직증명서 등 인증서 발급을 신청하고 관리자가 승인하는 워크플로
- flag로 승인/미승인 상태 구분

### 자주 쓰는 쿼리 패턴
```sql
SELECT seq, muid, apply_date, select_use, branch, approval_m_uid, approval_date, flag
FROM tbnws_certificate
ORDER BY apply_date DESC;
```

---

## alimtalk_template (카카오 알림톡 템플릿)

summary: 카카오 알림톡(알리고) 템플릿을 관리하는 테이블. 템플릿 코드, 메시지 유형, 강조 유형, 대체 문자 등의 정보를 저장한다.
domain: messaging
related_tables: alimtalk_template_button, adcall_alimtalk_rel, crm_as_alimtalk_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| template_code | varchar(10) | PK | 템플릿코드 | |
| channel_code | char | NOT NULL | 채널 코드 | |
| tag | varchar(50) | NOT NULL | 메세지 구분 태그 | |
| template_type | char(2) | NOT NULL, DEFAULT 'BA' | 템플릿 메세지 유형 | 'BA'(기본), 'EX'(부가정보형), 'AD'(광고추가형), 'MI'(복합형) |
| title | varchar(255) | NOT NULL | 템플릿명(제목) | |
| content | text | NOT NULL | 내용 | |
| additional_info | text | NULL | 부가정보 | |
| em_type | varchar(5) | NOT NULL, DEFAULT 'NONE' | 강조유형 | 'NONE'(없음), 'TEXT'(강조표기형), 'IMAGE'(이미지형) |
| emtitle | varchar(255) | NULL | 강조표기형 제목 | |
| em_title | varchar(255) | NULL | 강조표기형 제목 | |
| em_subtitle | varchar(255) | NULL | 강조표기형(기본형) 부제목 | |
| em_image | varchar(255) | NULL | 강조표기형 이미지 링크 | |
| fail_title | varchar(255) | NULL | 대체 문자 제목 | |
| fail_content | text | NULL | 대체 문자 내용 | |
| description | varchar(255) | NULL | 비고 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| inspection_status | char(3) | NOT NULL, DEFAULT 'REG' | 검수 상태 | 'REG'(등록), 'REQ'(심사요청), 'APR'(승인), 'REJ'(반려) |

### 관계
- alimtalk_template_button.template_code → alimtalk_template.template_code (FK, CASCADE)
- adcall_alimtalk_rel.template_code → alimtalk_template.template_code (FK, CASCADE)
- crm_as_alimtalk_template.template_code → alimtalk_template.template_code (FK)

### 비즈니스 맥락
- 카카오 알림톡 발송에 사용되는 템플릿 관리 (알리고 서비스 연동)
- 카카오 검수 프로세스: REG→REQ→APR/REJ
- fail_title, fail_content는 알림톡 발송 실패 시 대체 SMS 내용

### 자주 쓰는 쿼리 패턴
```sql
-- 템플릿 목록 + 버튼 조회
SELECT template.template_code, template.channel_code, template.tag, template.title,
       template.content, button.seq AS button_seq, button.button_name, button.link_type
FROM alimtalk_template AS template
LEFT JOIN alimtalk_template_button AS button ON template.template_code = button.template_code;
```

---

## alimtalk_template_button (알림톡 템플릿 버튼)

summary: 카카오 알림톡 템플릿에 포함되는 버튼 정보를 관리하는 테이블. 웹링크, 앱링크, 채널추가 등 다양한 버튼 유형을 지원한다.
domain: messaging
related_tables: alimtalk_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| template_code | varchar(10) | NOT NULL, FK | 템플릿코드 | |
| button_name | varchar(255) | NOT NULL | 버튼명 | |
| link_type | char(2) | NOT NULL | 링크 타입 코드 | 'AC'(채널 추가), 'DS'(배송조회), 'WL'(웹링크), 'AL'(앱링크), 'BK'(봇키워드), 'MD'(메시지전달) |
| link_type_name | varchar(50) | NOT NULL | 링크 타입명 | '채널 추가', '배송조회', '웹링크', '앱링크' |
| link_pc | varchar(255) | NULL | 웹링크 | |
| link_mob | varchar(255) | NULL | 모바일 링크 | |
| link_ios | varchar(255) | NULL | IOS Scheme | |
| link_and | varchar(255) | NULL | Android Scheme | |

### 관계
- template_code → alimtalk_template.template_code (FK, ON UPDATE CASCADE, ON DELETE CASCADE)

### 비즈니스 맥락
- 하나의 알림톡 템플릿에 여러 버튼을 추가할 수 있음
- link_type에 따라 필요한 링크 필드가 다름 (WL: link_pc/link_mob, AL: link_ios/link_and 등)

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 템플릿의 버튼 조회
SELECT seq AS button_seq, button_name, link_type, link_type_name,
       link_pc, link_mob, link_ios, link_and
FROM alimtalk_template_button
WHERE template_code = #{templateCode};
```

---

## message_adcall (애드콜 목록)

summary: 출고 후 일정 기간 경과한 고객에게 자동 발송하는 광고성 메시지(애드콜) 캠페인을 관리하는 테이블.
domain: messaging
related_tables: message_adcall_item, adcall_alimtalk_rel, alimtalk_template, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| title | varchar(255) | NULL | 제목 | |
| content | text | NULL | 내용 | |
| start_date | date | NULL | 시작일(구매 기준) | |
| end_date | date | NULL | 종료일(구매 기준) | |
| min_age_days | tinyint unsigned | NOT NULL | 출고 후 대기할 일자 | 7, 14, 30 |
| send_clock | tinyint unsigned | NOT NULL | 전송 시각(0~23) | 10, 14, 18 |
| active_flag | char | NOT NULL, DEFAULT 'N' | 활성화 여부 | 'Y', 'N' |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- adcall_alimtalk_rel.adcall_seq → message_adcall.seq (FK, CASCADE)
- message_adcall_item.adcall_seq → message_adcall.seq (FK, CASCADE)

### 비즈니스 맥락
- 출고 후 min_age_days일 경과 시점에 send_clock 시각에 자동 발송
- active_flag='Y'인 캠페인만 스케줄러에서 처리
- start_date~end_date는 대상 구매 기간 필터

### 자주 쓰는 쿼리 패턴
```sql
-- 활성 애드콜 + 알림톡 템플릿 조회
SELECT a.seq AS adcall_seq, a.title, a.content, a.start_date, a.end_date,
       a.min_age_days, a.send_clock, a.active_flag,
       r.template_code, r.template_content
FROM message_adcall AS a
LEFT JOIN adcall_alimtalk_rel AS r ON r.adcall_seq = a.seq
WHERE a.active_flag = 'Y' AND a.send_clock = #{hour};
```

---

## adcall_alimtalk_rel (애드콜-알림톡 릴레이션)

summary: 애드콜 캠페인과 알림톡 템플릿 간의 매핑 테이블. 애드콜 발송 시 사용할 알림톡 템플릿과 커스터마이즈된 내용을 연결한다.
domain: messaging
related_tables: message_adcall, alimtalk_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| adcall_seq | int unsigned | PK, FK | 애드콜 번호 | |
| template_code | varchar(10) | NOT NULL, FK, UNIQUE(adcall_seq, template_code) | 알림톡 템플릿 코드 | |
| template_content | text | NULL | 애드콜용 템플릿 내용 | |

### 관계
- adcall_seq → message_adcall.seq (FK, ON UPDATE CASCADE, ON DELETE CASCADE)
- template_code → alimtalk_template.template_code (FK, ON UPDATE CASCADE, ON DELETE CASCADE)

### 비즈니스 맥락
- 1:1 관계 (PK가 adcall_seq이므로 하나의 애드콜에 하나의 알림톡 템플릿)
- template_content에 템플릿의 원본 내용을 애드콜용으로 커스터마이즈하여 저장

### 자주 쓰는 쿼리 패턴
```sql
-- UPSERT 패턴으로 애드콜 알림톡 연결
INSERT INTO adcall_alimtalk_rel (adcall_seq, template_code, template_content)
VALUES (#{adCallSeq}, #{templateCode}, #{templateContent})
ON DUPLICATE KEY UPDATE
    template_code = VALUES(template_code),
    template_content = VALUES(template_content);
```

---

## message_adcall_item (애드콜 대상 항목)

summary: 애드콜 캠페인의 대상 제품과 파트너를 지정하는 테이블. 어떤 제품을 어떤 채널에서 구매한 고객에게 발송할지 정의한다.
domain: messaging
related_tables: message_adcall, erp_partner, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| adcall_seq | int unsigned | NOT NULL, FK, INDEX | 애드콜 번호 | |
| product_code | varchar(20) | NULL | 제품 코드 (prefix 매칭 가능) | 'G-0029' |
| partner_code | char(7) | NULL | 파트너 코드 | 'ptn0015' |

### 관계
- adcall_seq → message_adcall.seq (FK, ON UPDATE CASCADE, ON DELETE CASCADE)
- product_code → erp_product_info.product_code (LIKE 매칭)
- partner_code → erp_partner.partner_code

### 비즈니스 맥락
- product_code는 정확 매칭이 아닌 LIKE prefix 매칭으로 사용 (예: 'G-0029'로 해당 브랜드 전체 대상)
- partner_code가 NULL이면 전체 파트너 대상
- 하나의 애드콜에 여러 대상 항목 지정 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 애드콜의 대상 항목 + 파트너/제품 정보
SELECT i.seq AS adcall_item_seq, i.product_code, i.partner_code,
       t.partner_name, p.brand_name, p.goods_name, p.option_name
FROM message_adcall_item AS i
LEFT JOIN erp_partner AS t ON i.partner_code = t.partner_code
LEFT JOIN erp_product_info AS p ON i.product_code = p.product_code
WHERE i.adcall_seq = #{adCallSeq};
```

---

## as_board_rel (AS-게시판 릴레이션)

summary: AS 건과 GT기어 포럼 게시판(젠데스크) 글을 연결하는 릴레이션 테이블.
domain: misc
related_tables: crm_as, gtgear.gtgear_forum_board

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| as_seq | int unsigned | PK(복합), FK | AS seq | |
| board_id | int | PK(복합), FK | 게시판 board id | |

### 관계
- as_seq → crm_as.seq (FK, ON UPDATE CASCADE, ON DELETE CASCADE)
- board_id → gtgear.gtgear_forum_board.id (FK, ON UPDATE CASCADE, ON DELETE CASCADE)

### 비즈니스 맥락
- AS 접수 건과 고객이 올린 게시판 글(젠데스크)을 연결하여 참조 가능하게 함
- 복합 PK로 다대다 관계 구현

### 자주 쓰는 쿼리 패턴
```sql
SELECT as_seq, board_id FROM as_board_rel WHERE as_seq = #{asSeq};
```

---

## as_material_parts_rel (AS 부품 사용 릴레이션)

summary: AS 수리에 사용된 부품을 기록하는 릴레이션 테이블. crm_as_product와 erp_parts를 연결한다.
domain: misc
related_tables: crm_as_product, erp_parts

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| as_product_seq | int unsigned | PK(복합), FK | AS 제품 seq | |
| parts_seq | int unsigned | PK(복합), INDEX | 부품 seq | |

### 관계
- as_product_seq → crm_as_product.seq (FK)
- parts_seq → erp_parts.seq (논리적 참조)

### 비즈니스 맥락
- AS 수리 시 어떤 부품이 사용되었는지 기록
- 복합 PK로 하나의 AS 제품에 여러 부품 사용 가능

### 자주 쓰는 쿼리 패턴
```sql
SELECT as_product_seq, parts_seq
FROM as_material_parts_rel
WHERE as_product_seq = #{asProductSeq};
```

---

## as_material_stock_rel (AS 제품 사용 릴레이션)

summary: AS 수리 시 교체 투입된 재고(stock)를 기록하는 릴레이션 테이블. crm_as_product와 erp_stock을 1:1로 연결한다.
domain: misc
related_tables: crm_as_product, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| as_product_seq | int unsigned | PK, FK | AS 제품 seq | |
| stock_seq | int unsigned | NOT NULL, INDEX | 재고 stock seq | |

### 관계
- as_product_seq → crm_as_product.seq (FK)
- stock_seq → erp_stock.stock_seq (논리적 참조)

### 비즈니스 맥락
- AS 수리 시 교체 투입된 재고 단위를 기록 (1:1 관계)
- as_product_seq가 PK이므로 하나의 AS 제품에 하나의 stock만 연결

### 자주 쓰는 쿼리 패턴
```sql
SELECT as_product_seq, stock_seq
FROM as_material_stock_rel
WHERE as_product_seq = #{asProductSeq};
```

---

## crawler_coupang_price (쿠팡 가격 크롤링 대상)

summary: 쿠팡 가격 모니터링 대상 SKU 목록을 관리하는 테이블. 마지막으로 수집된 가격 정보와 활성화/알림 설정을 관리한다.
domain: misc
related_tables: crawler_coupang_price_history

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| sku_id | varchar(30) | PK | 크롤링 대상 SKU ID 고유값 | |
| product_id | varchar(30) | NULL, UNIQUE(product_id, vendor_item_id) | 상품 고유 일련번호(URL path에 사용) | |
| vendor_item_id | varchar(30) | NULL | 상점 내 상품 일련번호(URL query에 사용) | |
| page_title | varchar(255) | NULL | 쿠팡 페이지 타이틀(쿠팡 상품명) | |
| product_name | varchar(255) | NULL | 사용자 정의 상품명 | |
| original_price | int | NOT NULL, DEFAULT 0 | 정가(마지막으로 수집된) | |
| sales_price | int | NOT NULL, DEFAULT 0 | 쿠팡판매가(마지막으로 수집된) | |
| final_price | int | NOT NULL, DEFAULT 0 | 쿠폰할인(마지막으로 수집된) | |
| is_soldout | tinyint(1) | NOT NULL, DEFAULT 0 | 품절 여부 | 0, 1 |
| soldout_type | char | NULL | 품절 구분 | 'T'(임시품절), 'P'(영구품절) |
| is_active | char | NOT NULL, DEFAULT 'N' | 활성화(수집) 여부 | 'Y', 'N' |
| alert_flag | char | NOT NULL, DEFAULT 'Y' | 경고 알림 여부 | 'Y', 'N' |
| history_flag | char | NOT NULL, DEFAULT 'N' | 히스토리 저장 여부 | 'Y', 'N' |
| updated_at | datetime | NULL | 마지막 업데이트 일시 | |

### 관계
- crawler_coupang_price_history.sku_id → crawler_coupang_price.sku_id

### 비즈니스 맥락
- 쿠팡에서 자사 제품의 가격 변동을 모니터링하기 위한 크롤링 대상 목록
- is_active='Y'인 SKU만 크롤링 수행
- alert_flag='Y'면 가격 이상 시 알림 발송
- 가격이 변경되면 최신 가격이 이 테이블에 업데이트되고, history_flag='Y'면 히스토리에도 저장

### 자주 쓰는 쿼리 패턴
```sql
-- 활성화된 크롤링 대상 조회
SELECT sku_id, product_name, original_price, sales_price, final_price, is_soldout
FROM crawler_coupang_price
WHERE is_active = 'Y';
```

---

## crawler_coupang_price_history (쿠팡 가격 변동 히스토리)

summary: 쿠팡 가격 크롤링 수집 히스토리를 저장하는 테이블. 수집 시점의 가격, 알럿 발생 여부 등을 시계열로 기록한다.
domain: misc
related_tables: crawler_coupang_price

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK(복합), AUTO_INCREMENT | 번호 | |
| sku_id | varchar(30) | PK(복합), NOT NULL | 쿠팡 상품 리스트의 PK | |
| product_id | varchar(30) | PK(복합), NOT NULL | Product ID | |
| vendor_item_id | varchar(30) | NOT NULL | Vendor item ID | |
| discounted_price | int | NOT NULL, DEFAULT 0 | 수집 당시 상품 상시가 | |
| alert_price | int | NOT NULL, DEFAULT 0 | 상시가에서 계산된 경고 가격 | |
| original_price | int | NOT NULL, DEFAULT 0 | 쿠팡 정가 | |
| sales_price | int | NOT NULL, DEFAULT 0 | 쿠팡판매가 | |
| final_price | int | NOT NULL, DEFAULT 0 | 할인가(실판매가격) | |
| alert_flag | char | NOT NULL, DEFAULT 'N' | 알럿 설정 여부 | 'Y', 'N' |
| is_success | char | NOT NULL, DEFAULT 'N' | 성공 여부 | 'Y', 'N' |
| is_alert | char | NOT NULL, DEFAULT 'N' | 알럿 발생 여부 | 'Y', 'N' |
| collected_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수집일시 | |

### 관계
- sku_id → crawler_coupang_price.sku_id
- 복합 PK: (seq, sku_id, product_id)

### 비즈니스 맥락
- 가격 크롤링 결과를 시계열로 보존하여 가격 변동 추이 분석 가능
- is_alert='Y'이면 해당 수집 시점에 가격 이상이 감지됨
- is_success='N'이면 크롤링 실패 (페이지 접근 불가 등)

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 SKU의 가격 변동 히스토리 조회
SELECT seq, sku_id, original_price, sales_price, final_price,
       is_alert, collected_at
FROM crawler_coupang_price_history
WHERE sku_id = #{skuId}
ORDER BY collected_at DESC;
```

---

## gtgear_partner_list (GT기어 거래처 목록)

summary: GT기어 브랜드의 오프라인 거래처(체험 매장 등) 정보를 관리하는 테이블. 위치(위경도), 영업시간, 쿠폰 계약 여부 등을 포함한다.
domain: misc
related_tables: (독립 테이블)

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| partner_code | varchar(20) | PK, DEFAULT '' | 파트너코드 | |
| partner_name | varchar(100) | NULL, DEFAULT '' | 파트너명 | |
| address | varchar(255) | NULL, DEFAULT '' | 주소 | |
| phone | varchar(20) | NULL, DEFAULT '' | 전화번호 | |
| opening | varchar(50) | NULL, DEFAULT '' | 영업시간 | '10:00~20:00' |
| url | varchar(100) | NULL, DEFAULT '' | 상세페이지URL | |
| latitude | varchar(20) | NULL, DEFAULT '' | 위도 | |
| longitude | varchar(20) | NULL, DEFAULT '' | 경도 | |
| collabo_flag | char | NULL, DEFAULT 'N' | 쿠폰 계약 여부 | 'Y', 'N' |

### 관계
- 독립 테이블 (erp_partner와는 별도)

### 비즈니스 맥락
- GT기어 브랜드 제품을 체험/판매하는 오프라인 매장 정보
- 위경도 정보로 지도 표시 가능
- collabo_flag='Y'인 매장은 쿠폰 제휴 진행 중
- erp_partner와는 별도로 GT기어 전용 오프라인 파트너를 관리

### 자주 쓰는 쿼리 패턴
```sql
SELECT partner_code, partner_name, address, phone, opening, url,
       latitude, longitude, collabo_flag
FROM gtgear_partner_list;
```
