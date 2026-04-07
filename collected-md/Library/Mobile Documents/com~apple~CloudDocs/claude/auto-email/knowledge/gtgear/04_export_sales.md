# 출고/매출 도메인 테이블 (Export & Sales)

---

## erp_export_log (출고 이력 로그)

summary: 재고(erp_stock)의 출고, 출고 수정, 재고 조정, 출고 취소 등 변경 이력을 기록하는 감사(audit) 테이블이다. 출고 관련 모든 변경사항을 추적할 수 있다.
domain: export
related_tables: erp_stock, erp_partner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| log_seq | bigint unsigned | PK, AUTO_INCREMENT | 로그 일련번호 | |
| stock_seq | int unsigned | NOT NULL | 재고 번호 (erp_stock FK) | |
| product_seq | int unsigned | NULL | 상품 번호 | |
| log_type | enum | NOT NULL | 로그 유형 | 'EXPORT', 'EXPORT_UPDATE', 'STOCK_ADJUST', 'EXPORT_CANCEL' |
| export_seq | int | NULL | 출고 번호 | |
| export_num | int | NULL | 출고 수량 | |
| export_date | date | NULL | 출고일 | |
| export_partner_code | char(7) | NULL | 출고 파트너 코드 | ptn0000(직접출고), ptn0001(CJ대한통운) |
| selling_price | decimal(10,2) | NULL | 판매 가격 | |
| user_name | varchar(50) | NULL | 수취인명 | |
| user_address | varchar(250) | NULL | 수취인 주소 | |
| courier | varchar(10) | NULL | 택배사 코드 | |
| invoice | varchar(255) | NULL | 송장번호 | |
| action_at | datetime | DEFAULT CURRENT_TIMESTAMP | 작업 일시 | |
| action_by | varchar(255) | NULL | 작업자 | |
| action_reason | varchar(255) | NULL | 작업 사유 | |

### 관계
- `stock_seq` -> `erp_stock.stock_seq`: 해당 출고 이력이 연결된 재고 레코드
- `export_partner_code` -> `erp_partner.partner_code`: 출고 파트너(택배사/거래처)

### 비즈니스 맥락
- 출고 처리 시 자동으로 로그가 생성되어 이력을 추적한다.
- log_type으로 출고 생성(EXPORT), 수정(EXPORT_UPDATE), 재고 조정(STOCK_ADJUST), 출고 취소(EXPORT_CANCEL)를 구분한다.
- 인덱스: `idx_stock(stock_seq)`, `idx_type_date(log_type, action_at)` — 재고별/유형+일자별 조회에 최적화되어 있다.

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 재고의 출고 이력 조회
SELECT * FROM erp_export_log WHERE stock_seq = ? ORDER BY action_at DESC;

-- 특정 기간 출고 취소 건수 조회
SELECT COUNT(*) FROM erp_export_log WHERE log_type = 'EXPORT_CANCEL' AND action_at BETWEEN ? AND ?;
```

---

## erp_export_request (출고 요청)

summary: 출고 요청 정보를 관리하는 테이블이다. 카테고리/브랜드/상품 정보와 수취인 정보를 포함하며, 요청 등록 후 승인(request_flag)을 통해 출고 프로세스가 진행된다.
domain: export
related_tables: erp_product_info, erp_category, member, crm_as_export_product

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| request_id | int | PK, AUTO_INCREMENT | 출고 요청 ID | |
| request_title | varchar(100) | NULL | 요청 제목 | |
| request_type | varchar(20) | NULL | 요청 유형 | |
| export_schedule_date | date | DEFAULT curdate() | 출고 예정일 | |
| export_method | varchar(10) | NULL | 출고 방법 | |
| stock_flag | char | DEFAULT 'O' | 재고 플래그 | 'O' |
| category_code | char | NULL | 카테고리 코드 | 'G'(GTGear), 'F'(리퍼브) |
| brand_code | char(7) | DEFAULT 'brd0000' | 브랜드 코드 | |
| goods_code | char(6) | DEFAULT 'gs0000' | 상품 코드 | |
| option_code | char(7) | NULL | 옵션 코드 | |
| ea | int | DEFAULT 0 | 수량 | |
| user_name | varchar(30) | DEFAULT '' | 수취인명 | |
| user_contact | varchar(15) | DEFAULT '' | 수취인 연락처 | |
| user_address | varchar(250) | DEFAULT '' | 수취인 주소 | |
| user_postal_code | varchar(10) | DEFAULT '' | 우편번호 | |
| user_memo | varchar(250) | DEFAULT '' | 배송 메모 | |
| description | varchar(250) | DEFAULT '' | 비고 | |
| request_flag | char | NULL | 처리 상태 | NULL(미처리), 'Y'(완료) |
| regist_muid | varchar(30) | NULL | 등록자 UID (구버전) | |
| regist_member_id | int unsigned | FK -> member.id | 등록자 회원 ID (신버전) | |
| regist_date | datetime | DEFAULT now() | 등록일시 | |
| product_code | varchar(100) | NULL | 상품 코드 (조합형) | 'G-0029-0001-0001001' |
| complete_date | datetime | NULL | 완료일시 | |

### 관계
- `regist_member_id` -> `member.id`: 출고 요청 등록자
- `request_id` <- `crm_as_export_product.export_request_seq`: A/S 출고 제품에서 참조
- `category_code` + `brand_code` + `goods_code` + `option_code` -> `erp_product_info`: 상품 정보 조인

### 비즈니스 맥락
- 출고 요청은 관리자가 등록하며, request_flag가 NULL이면 미처리, 'Y'이면 완료 상태이다.
- 주문번호(order_number)는 등록일 + request_id로 동적 생성된다: `CONCAT(DATE_FORMAT(regist_date, '%y%m%d'), LPAD(request_id, 9, '0'))`.
- A/S 출고 프로세스에서도 이 테이블을 사용하여 출고 요청을 생성한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 출고 요청 목록 조회 (상태별, 기간별 필터)
SELECT request_id,
       CONCAT(DATE_FORMAT(regist_date, '%y%m%d'), LPAD(request_id, 9, '0')) AS order_number,
       request_title, user_name, request_type, request_flag, regist_date
FROM erp_export_request
WHERE request_flag IS NULL
  AND DATE_FORMAT(regist_date, '%Y-%m-%d') >= ? AND DATE_FORMAT(regist_date, '%Y-%m-%d') <= ?
ORDER BY request_flag, regist_date DESC;

-- 엑셀 데이터 추출 (상품 정보 포함)
SELECT e_r.*, p_i.brand_name, p_i.goods_name, p_i.option_name
FROM erp_export_request e_r
LEFT JOIN erp_product_info p_i ON e_r.product_code = p_i.product_code
WHERE request_flag = 'Y';
```

---

## erp_export_schedule (출고 스케줄)

summary: 출고 일정(배치) 단위를 관리하는 테이블이다. 하나의 스케줄에 여러 출고 스케줄 아이템(erp_export_schedule_item)이 포함된다.
domain: export
related_tables: erp_export_schedule_item

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| schedule_seq | int | PK, AUTO_INCREMENT | 스케줄 일련번호 | |
| schedule_title | varchar(100) | NULL | 스케줄 제목 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- `schedule_seq` <- `erp_export_schedule_item.schedule_seq`: 스케줄에 포함된 출고 아이템 목록

### 비즈니스 맥락
- 출고 스케줄은 출고 작업을 배치 단위로 묶어 관리한다.
- 스케줄 내 모든 아이템의 출고가 완료되면 전체 완료 처리한다.
- 아이템의 수량 합계(date_count)와 출고 완료 수량(export_date_count)을 비교하여 진행률을 파악한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 출고 스케줄 목록 (진행상황 포함)
SELECT a.*,
       IFNULL(SUM(b.ea), 0) AS date_count,
       SUM(CASE WHEN b.export_flag = 'Y' THEN b.ea ELSE 0 END) AS export_date_count,
       IFNULL(GROUP_CONCAT(DISTINCT export_app_date SEPARATOR '/'), '') AS export_app_date_group
FROM erp_export_schedule a
LEFT JOIN erp_export_schedule_item b ON a.schedule_seq = b.schedule_seq
GROUP BY a.schedule_seq
ORDER BY a.schedule_seq DESC;
```

---

## erp_export_schedule_item (출고 스케줄 아이템)

summary: 출고 스케줄의 개별 출고 아이템 정보를 관리한다. 수취인, 상품, 수량, 출고 파트너, 송장 등 실제 출고에 필요한 모든 상세 정보를 담고 있다.
domain: export
related_tables: erp_export_schedule, erp_product_info, erp_partner, export_scheduled_ea_view

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| item_seq | int | PK, AUTO_INCREMENT | 아이템 일련번호 | |
| schedule_seq | int | FK, INDEX | 스케줄 번호 | |
| user_name | varchar(50) | NULL | 수취인명 | |
| user_contact | varchar(50) | NULL | 수취인 연락처 | |
| user_phone | varchar(50) | NULL | 수취인 전화번호 | |
| user_address | varchar(250) | NULL | 수취인 주소 | |
| user_memo | varchar(250) | NULL | 배송 메모 | |
| goods_name | varchar(250) | NULL | 상품명 | |
| selected_import_seq | int | DEFAULT 0 | 선택된 입고 번호 | |
| product_code | varchar(50) | NULL, INDEX | 상품 코드 | 'G-0029-0001-0001001' |
| category_code | char | NULL | 카테고리 코드 | |
| brand_code | char(7) | NULL | 브랜드 코드 | |
| goods_code | char(6) | NULL | 상품 코드 | |
| option_code | char(7) | NULL | 옵션 코드 | |
| ea | int | NULL | 수량 | |
| partner_code | char(7) | NULL | 출고 파트너 코드 | ptn0000, ptn0001, ptn0209 |
| description | varchar(250) | NULL | 비고 | |
| export_app_date | date | NULL | 출고 예정일 | |
| export_num | int | NULL | 출고 번호 | |
| courier | varchar(10) | DEFAULT '' | 택배사 | |
| invoice | varchar(250) | DEFAULT '' | 송장번호 | |
| delivery_type | varchar(50) | DEFAULT '' | 배송 유형 | |
| export_flag | char(2) | NULL | 출고 완료 여부 | 'Y'(완료), 'N'(미완료) |
| sms_flag | char(2) | NULL | SMS 발송 여부 | 'Y', 'N' |

### 관계
- `schedule_seq` -> `erp_export_schedule.schedule_seq`: 소속 스케줄
- `product_code` -> `erp_product_info.product_code`: 상품 정보
- `partner_code` -> `erp_partner.partner_code`: 출고 파트너(택배사/거래처)
- 이 테이블의 집계 -> `export_scheduled_ea_view`: 미출고 수량 뷰

### 비즈니스 맥락
- export_flag가 'N'이면 미출고, 'Y'이면 출고 완료이다.
- partner_code 'ptn0209'는 안전재고 관련 파트너로, 안전재고 체크(safety_stock_check)에 사용된다.
- 인덱스: `idx_export_schedule_flag_product(export_flag, product_code)` — 출고 상태별 상품 조회에 최적화.

### 자주 쓰는 쿼리 패턴
```sql
-- 스케줄별 아이템 상세 조회 (상품/파트너 정보 포함)
SELECT item.*, info.product_code, info.brand_name, info.goods_name, info.option_name,
       partner.partner_name
FROM erp_export_schedule_item item
LEFT JOIN erp_product_info info
  ON item.category_code = info.category_code AND item.brand_code = info.brand_code
     AND item.goods_code = info.goods_code AND item.option_code = info.option_code
LEFT JOIN erp_partner partner ON item.partner_code = partner.partner_code
WHERE schedule_seq = ?;

-- 미출고 아이템의 상품코드별 출고 예정 수량
SELECT product_code, ea, export_app_date
FROM erp_export_schedule_item
WHERE schedule_seq = ? AND export_flag = 'N';
```

---

## erp_item_shipment (입고/선적 관리)

summary: 해외 공급업체로부터의 입고(선적) 프로세스를 추적하는 테이블이다. 출발, 항구 도착, 통관, 입고 확인, 판매 확인 등 물류 전체 단계를 관리한다.
domain: export
related_tables: erp_order_item, erp_shipment_group, tbnws_packing_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| shipment_seq | int unsigned | PK, AUTO_INCREMENT | 선적 일련번호 | |
| item_seq | int unsigned | NULL, INDEX | 발주 상품 번호 (erp_order_item FK) | |
| ea | int | NULL | 수량 | |
| import_confirm_date | datetime | NULL | 입고예정 등록일시 | |
| import_confirm_member_id | varchar(255) | NULL | 입고 확인자 ID | |
| clearance_confirm_member_id | varchar(255) | NULL | 통관 확인자 ID | |
| sale_confirm_date | datetime | NULL | 판매 확인일시 | |
| sale_confirm_member_id | varchar(255) | NULL | 판매 확인자 ID | |
| description | varchar(255) | NULL | 비고 | |
| pl_number | varchar(255) | NOT NULL, DEFAULT '' | PL(Packing List) 번호 | |
| departure_date | datetime | NULL | 출발일자 | |
| departure_confirm_member_id | varchar(255) | NULL | 출발 확인자 ID | |
| arrive_date | datetime | NULL | 항구 도착일자 | |
| arrive_confirm_member_id | varchar(255) | NULL | 도착 확인자 ID | |
| clearance_date | datetime | NULL | 통관확인일자 | |

### 관계
- `item_seq` -> `erp_order_item.item_seq`: 발주 상품 항목
- `group_seq` -> `erp_shipment_group.group_seq`: 선적 그룹 (매퍼에서 JOIN)
- `pl_number` -> `tbnws_packing_list.pl_number`: 패킹리스트

### 비즈니스 맥락
- 선적 프로세스 단계: 입고예정등록(import_confirm) -> 출발(departure) -> 항구도착(arrive) -> 통관(clearance) -> 판매확인(sale_confirm)
- 각 단계마다 확인 일시와 확인자 ID가 기록된다.
- 발주(erp_order) -> 발주아이템(erp_order_item) -> 선적(erp_item_shipment) 순서로 연결된다.

### 자주 쓰는 쿼리 패턴
```sql
-- 선적 상세 조회 (발주 아이템, 그룹 정보 포함)
SELECT shipment.*, item.product_code, item.ea, shipment_group.group_seq
FROM erp_item_shipment AS shipment
LEFT JOIN erp_order_item AS item ON shipment.item_seq = item.item_seq
LEFT JOIN erp_shipment_group AS shipment_group ON shipment.group_seq = shipment_group.group_seq
WHERE shipment_seq = ?;

-- 발주 번호로 선적 내역 조회
SELECT * FROM erp_item_shipment
WHERE item_seq IN (SELECT item_seq FROM erp_order_item WHERE order_seq = ?)
ORDER BY shipment_seq DESC;

-- 판매 확인 처리 (일괄)
UPDATE erp_item_shipment SET sale_confirm_date = NOW(), sale_confirm_member_id = ?
WHERE shipment_seq IN (?);
```

---

## erp_advance_shipment (사전 선적 알림)

summary: 입고 예정(사전 선적) 정보를 기록하는 테이블이다. 아직 정식 선적 등록 전에 미리 입고 예정 수량과 출고일을 알려주는 용도로 사용된다.
domain: export
related_tables: erp_order_item, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| advance_shipment_seq | int unsigned | PK, AUTO_INCREMENT | 사전 선적 일련번호 | |
| item_seq | int unsigned | NOT NULL, INDEX | 발주 상품 번호 | |
| product_code | char(16) | NOT NULL, INDEX | 상품 코드 | |
| export_date | datetime | NULL | 출고(선적)일 | |
| order_seq | int | NULL | 발주 번호 | |
| ea | int | NOT NULL | 수량 | |

### 관계
- `item_seq` -> `erp_order_item.item_seq`: 발주 상품 항목
- `product_code` -> `erp_product_info.product_code`: 상품 정보
- `order_seq` -> `erp_order.order_seq`: 발주 정보

### 비즈니스 맥락
- 해외 공급업체로부터 사전 입고 통보를 받았을 때 미리 수량과 예상 입고일을 등록한다.
- 재고 계획 수립 시 입고 예정 수량으로 활용된다.

### 자주 쓰는 쿼리 패턴
```sql
-- 상품별 사전 선적 수량 조회
SELECT product_code, SUM(ea) AS total_advance_ea
FROM erp_advance_shipment
GROUP BY product_code;
```

---

## tbnws_combined_packaging (합포 주문 정보)

summary: 이플럭스 합포 주문과 사방넷 주문을 연결하는 테이블이다. 여러 주문을 하나의 배송으로 합치는 합포 처리 정보를 관리한다.
domain: export
related_tables: tbnws_sabangnet_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| eflux_combined_packaging_seq | varchar(255) | NOT NULL, INDEX | 이플럭스 합포 번호 | |
| sabangnet_idx | varchar(255) | NOT NULL, INDEX | 사방넷 주문번호 | |
| mall_id | varchar(50) | NOT NULL | 몰 ID | |
| receive_name | varchar(50) | NOT NULL | 수취인명 | |
| receive_addr | varchar(200) | NOT NULL | 수취인 주소 | |
| receive_zipcode | varchar(10) | NOT NULL | 우편번호 | |
| receive_cel | varchar(20) | NOT NULL | 연락처 | |
| order_date | date | NOT NULL | 주문일자 | |
| product_code | varchar(1000) | NOT NULL | 상품코드 (다건 포함 가능) | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |

### 관계
- `sabangnet_idx` -> `tbnws_sabangnet_order.idx`: 사방넷 주문
- `eflux_combined_packaging_seq`: 이플럭스 WMS 합포 번호와 연결

### 비즈니스 맥락
- 동일 수취인에게 보내는 여러 주문을 하나의 박스로 합포 배송할 때 사용한다.
- 이플럭스(물류 대행사)의 합포 번호와 사방넷(주문 통합 시스템)의 주문번호를 매핑한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 합포 번호로 포함된 주문 조회
SELECT * FROM tbnws_combined_packaging
WHERE eflux_combined_packaging_seq = ?;

-- 사방넷 주문번호로 합포 여부 확인
SELECT * FROM tbnws_combined_packaging
WHERE sabangnet_idx = ?;
```

---

## tbnws_packing_list (패킹리스트)

summary: 해외 수입 화물의 패킹리스트(PL)와 선하증권(BL) 정보를 관리하는 테이블이다. 통관 프로세스(수입신고, 반입신고, 반출)의 진행 상태를 추적한다.
domain: export
related_tables: erp_item_shipment

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 일련번호 | |
| pl_number | varchar(255) | NOT NULL, UNIQUE | PL 번호 | |
| bl_number | varchar(255) | DEFAULT '', NULL | BL(선하증권) 번호 | |
| arr_date | datetime | NULL | 도착일자 | |
| pl_files | text | NOT NULL | PL PDF 파일 경로 | |
| createdDate | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일자 | |
| lastModifyDate | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일자 | |
| inboundCompletedBy | varchar(100) | NULL | 입고완료자 | |
| inboundCompleteDate | datetime | NULL | 입고완료일시 | |
| import_clear_yn | varchar(2) | NOT NULL, DEFAULT 'N' | 통관 수입신고 수리 여부 | 'Y', 'N' |
| carry_in_yn | varchar(2) | NOT NULL, DEFAULT 'N' | 통관 반입신고 여부 | 'Y', 'N' |
| receipt_order_yn | varchar(2) | NOT NULL, DEFAULT 'N' | 통관 반출 여부 | 'Y', 'N' |

### 관계
- `pl_number` <- `erp_item_shipment.pl_number`: 선적 건에서 패킹리스트 참조

### 비즈니스 맥락
- 수입 통관 프로세스: 수입신고 수리(import_clear_yn) -> 반입신고(carry_in_yn) -> 반출(receipt_order_yn)
- 각 단계가 'Y'로 변경되면 다음 단계로 진행 가능하다.
- PL 번호는 고유하며(UNIQUE), 중복 등록을 방지한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 패킹리스트 전체 목록 조회 (최신순)
SELECT ROW_NUMBER() OVER (ORDER BY createdDate DESC) AS seq,
       pl_number, bl_number, arr_date, pl_files, createdDate,
       import_clear_yn, carry_in_yn, receipt_order_yn
FROM tbnws_packing_list;

-- BL 번호 목록 조회
SELECT bl_number FROM tbnws_packing_list GROUP BY bl_number;

-- 통관 수입신고 수리 처리
UPDATE tbnws_packing_list SET import_clear_yn = 'Y'
WHERE pl_number = ? AND bl_number = ?;
```

---

## erp_settlement (정산)

summary: 판매 채널(마켓플레이스)별 정산 데이터를 저장하는 테이블이다. 상품별 판매가, 몰 원가, 결제 금액 등 정산에 필요한 금액 정보와 SKU 정보를 포함한다.
domain: sales
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| settle_idx | varchar(40) | PK | 정산 인덱스 (고유키) | |
| mail_id | varchar(40) | NULL, INDEX | 메일 ID | |
| PRODUCT_NAME | varchar(255) | NULL | 상품명 | |
| P_PRODUCT_NAME | varchar(255) | NULL | 부모 상품명 | |
| SALE_COST | varchar(50) | NULL | 판매가 | |
| MALL_WON_COST | varchar(50) | NULL | 몰 원가 | |
| MODEL_NO | varchar(255) | NULL | 모델 번호 | |
| SET_GUBUN | varchar(5) | NULL | 세트 구분 | |
| TOTAL_COST | varchar(50) | NULL | 총 금액 | |
| PAY_COST | varchar(50) | NULL | 결제 금액 | |
| SKU_VALUE | blob | NULL | SKU 정보 (바이너리) | |
| createdBy | date | NULL, INDEX | 생성일자 | |
| order_status | varchar(50) | NOT NULL | 주문 상태 | |
| pEa | int | NOT NULL | 수량 | |
| category_code | varchar(10) | NULL | 카테고리 코드 | |
| brand_code | varchar(10) | NULL | 브랜드 코드 | |
| good_code | varchar(10) | NULL | 상품 코드 | |
| option_code | varchar(10) | NULL | 옵션 코드 | |

### 관계
- `category_code` + `brand_code` + `good_code` + `option_code` -> `erp_product_info`: 상품 정보

### 비즈니스 맥락
- 판매 채널(네이버, 사방넷 등)로부터 수집된 정산 데이터를 저장한다.
- 금액 컬럼들이 varchar 타입이므로 집계 시 CAST가 필요하다.
- mail_id와 createdBy 기준 복합 인덱스가 존재한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 기간별 정산 데이터 조회
SELECT * FROM erp_settlement
WHERE createdBy BETWEEN ? AND ?;

-- 상품별 정산 합계
SELECT PRODUCT_NAME, SUM(CAST(PAY_COST AS DECIMAL(15,2))) AS total_pay
FROM erp_settlement
WHERE createdBy BETWEEN ? AND ?
GROUP BY PRODUCT_NAME;
```

---

## erp_annual_sales (연간 브랜드별 매출)

summary: 연간 브랜드별, 유형별 매출 금액을 저장하는 집계 테이블이다. 2025년 이전 과거 데이터를 보관하며, 2025년 이후 데이터는 erp_customer_order_list에서 실시간 집계한다.
domain: sales
related_tables: erp_annual_model_sales, erp_annual_acc_sales, erp_annual_alice_layout_sales

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| year | varchar(10) | NULL | 년도 | '2024' |
| month | varchar(10) | NULL | 월 | '1', '2', ... '12' |
| brand | varchar(100) | NULL | 브랜드명 | '키크론', 'AIPER' |
| type | varchar(10) | NULL | 매출 유형 | |
| price | int | NULL | 매출 금액 | |

### 관계
- 같은 도메인의 집계 테이블: erp_annual_model_sales, erp_annual_acc_sales, erp_annual_alice_layout_sales

### 비즈니스 맥락
- 연간 판매 분석 대시보드에서 브랜드별 월간 매출 추이를 보여주기 위해 사용한다.
- 2025년 이후 데이터는 erp_customer_order_list + erp_customer_order_partner_list에서 UNION ALL로 결합하여 조회한다.
- PK가 없는 집계 테이블이다.

### 자주 쓰는 쿼리 패턴
```sql
-- 과거 연도 브랜드별 매출 조회 (2025년 이후는 order_base_data CTE 사용)
SELECT year, month, brand AS brand_name, type, price
FROM erp_annual_sales
WHERE year = ?;

-- 2025년 이후: CTE로 실시간 집계
-- WITH order_base_data AS (...) SELECT ... FROM order_base_data WHERE YEAR(order_date) = ?
```

---

## erp_annual_acc_sales (연간 악세서리별 판매량)

summary: 연간 악세서리 유형별 판매 수량과 금액을 저장하는 집계 테이블이다. 키캡, 스위치, 손목받침대 등 액세서리 카테고리별 판매 현황을 파악하는 데 사용된다.
domain: sales
related_tables: erp_annual_sales, erp_annual_model_sales

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| year | int | NULL | 년도 | 2024 |
| month | varchar(10) | NULL | 월 | '1', '2', ... '12' |
| acc_type | varchar(100) | NULL | 악세서리 유형 | |
| price | int | NULL | 매출 금액 | |
| ea | int | NULL | 판매 수량 | |

### 관계
- 같은 도메인의 집계 테이블 그룹에 속한다

### 비즈니스 맥락
- 2025년 이전 과거 악세서리 판매 데이터를 보관한다.
- 2025년 이후에는 order_base_data CTE의 acc_type 기준으로 실시간 집계한다.
- PK가 없는 집계 테이블이다.

### 자주 쓰는 쿼리 패턴
```sql
-- 연도별 악세서리 유형 판매량 조회
SELECT month, acc_type, ea, price
FROM erp_annual_acc_sales
WHERE year = ?;
```

---

## erp_annual_model_sales (연간 모델별 판매량)

summary: 연간 키보드 모델별 판매 수량과 금액을 저장하는 집계 테이블이다. Q1, K2, V6 등 모델 단위로 판매 추이를 분석하는 데 사용된다.
domain: sales
related_tables: erp_annual_sales, erp_annual_acc_sales

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| year | int | NULL | 년도 | 2024 |
| month | varchar(10) | NULL | 월 | '1', '2', ... '12' |
| model_name | varchar(100) | NULL | 모델명 | |
| price | int | NULL | 매출 금액 | |
| ea | int | NULL | 판매 수량 | |

### 관계
- 같은 도메인의 집계 테이블 그룹에 속한다

### 비즈니스 맥락
- 2025년 이전 과거 모델별 판매 데이터를 보관한다.
- 2025년 이후에는 order_base_data CTE의 model_name 기준으로 실시간 집계한다.
- PK가 없는 집계 테이블이다.

### 자주 쓰는 쿼리 패턴
```sql
-- 연도별 모델 판매량 조회
SELECT month, model_name, ea, price
FROM erp_annual_model_sales
WHERE year = ?;
```

---

## erp_annual_alice_layout_sales (연간 레이아웃별 판매량)

summary: 연간 키보드 레이아웃(배열)별 판매 수량과 금액을 저장하는 집계 테이블이다. 풀배열, 텐키리스, 65% 등 레이아웃 기준으로 판매 분석에 사용된다.
domain: sales
related_tables: erp_annual_sales, erp_annual_model_sales

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| year | int | NULL | 년도 | 2024 |
| month | varchar(10) | NULL | 월 | '1', '2', ... '12' |
| layout_name | varchar(100) | NULL | 레이아웃명 | |
| price | int | NULL | 매출 금액 | |
| ea | int | NULL | 판매 수량 | |

### 관계
- 같은 도메인의 집계 테이블 그룹에 속한다

### 비즈니스 맥락
- 2025년 이전 과거 레이아웃별 판매 데이터를 보관한다.
- 2025년 이후에는 order_base_data CTE의 layout_name 기준으로 실시간 집계한다.
- PK가 없는 집계 테이블이다.

### 자주 쓰는 쿼리 패턴
```sql
-- 연도별 레이아웃 판매량 조회
SELECT month, layout_name, ea, price
FROM erp_annual_alice_layout_sales
WHERE year = ?;
```

---

## erp_annual_sales_memos (연간 매출 메모)

summary: 연간 매출 분석 대시보드에서 관리자가 작성하는 메모를 저장하는 테이블이다. 브랜드별, 기간별 특이사항이나 분석 코멘트를 기록한다. JSON metadata로 필터 조건을 저장한다.
domain: sales
related_tables: erp_annual_sales, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | 메모 ID | |
| memo_key | varchar(500) | NOT NULL, UNIQUE(memo_key, memo_type) | 메모 키 | |
| memo_type | varchar(50) | NOT NULL, INDEX | 메모 유형 | |
| memo_text | text | NULL | 메모 내용 | |
| metadata | json | NULL | 메타데이터 (date, brand, year 등) | {"date":"2024-03","brand":"키크론","year":"2024"} |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| member_id | int unsigned | NULL | 작성자 회원 ID | |

### 관계
- `member_id` -> `member.id`: 메모 작성자

### 비즈니스 맥락
- memo_key와 memo_type의 복합 유니크 제약으로 중복 메모를 방지한다.
- metadata JSON 필드를 통해 브랜드, 연도, 날짜 등의 필터 조건을 유연하게 저장한다.
- JSON_EXTRACT 함수로 metadata 내부 값을 기준으로 조회한다.

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 브랜드/기간의 메모 조회
SELECT memo_key, memo_text, memo_type, metadata
FROM erp_annual_sales_memos
WHERE memo_type = ?
  AND JSON_UNQUOTE(JSON_EXTRACT(metadata, '$.date')) BETWEEN ? AND ?
  AND JSON_EXTRACT(metadata, '$.brand') = ?;

-- 연도별 메모 조회
SELECT memo_key, memo_text, memo_type, metadata
FROM erp_annual_sales_memos
WHERE memo_type = ? AND JSON_EXTRACT(metadata, '$.year') = ?;

-- 메모 존재 여부 확인
SELECT EXISTS(SELECT * FROM erp_annual_sales_memos WHERE memo_key = ? AND memo_type = ?);
```

---

## erp_import_log (입고 이력 로그)

summary: 재고(erp_stock)의 입고, 입고 수정, 입고 취소, 외부 입고, 반품, 재배치, 재고 조정 등 입고 관련 모든 변경 이력을 기록하는 감사(audit) 테이블이다.
domain: export
related_tables: erp_stock, erp_partner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| log_seq | bigint unsigned | PK, AUTO_INCREMENT | 로그 일련번호 | |
| stock_seq | int unsigned | NOT NULL, INDEX | 재고 번호 (erp_stock FK) | |
| product_seq | int unsigned | NULL | 상품 번호 | |
| log_type | enum | NOT NULL | 로그 유형 | 'IMPORT', 'IMPORT_UPDATE', 'IMPORT_CANCEL', 'IMPORT_EXTERNAL', 'RETURN', 'RELOCATION', 'STOCK_ADJUST' |
| item_seq | int unsigned | NULL | 발주 아이템 번호 | |
| import_date | date | NULL | 입고일 | |
| import_partner_code | char(7) | NULL | 입고 파트너 코드 | |
| buying_price | decimal(15,3) | NULL | 매입 가격 | |
| warehouse_tag | varchar(10) | NULL | 창고 태그 | |
| action_at | datetime | DEFAULT CURRENT_TIMESTAMP | 작업 일시 | |
| action_by | int unsigned | NULL | 작업자 ID | |
| action_reason | varchar(255) | NULL | 작업 사유 | |

### 관계
- `stock_seq` -> `erp_stock.stock_seq`: 해당 입고 이력이 연결된 재고 레코드
- `import_partner_code` -> `erp_partner.partner_code`: 입고 파트너

### 비즈니스 맥락
- log_type으로 다양한 입고 관련 작업을 구분한다:
  - IMPORT: 정상 입고
  - IMPORT_UPDATE: 입고 정보 수정
  - IMPORT_CANCEL: 입고 취소
  - IMPORT_EXTERNAL: 외부 입고
  - RETURN: 반품
  - RELOCATION: 재배치
  - STOCK_ADJUST: 재고 조정
- 인덱스: `idx_stock(stock_seq)`, `idx_type_date(log_type, action_at)` — 재고별/유형+일자별 조회에 최적화되어 있다.

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 재고의 입고 이력 조회
SELECT * FROM erp_import_log WHERE stock_seq = ? ORDER BY action_at DESC;

-- 특정 기간 반품 건수 조회
SELECT COUNT(*) FROM erp_import_log WHERE log_type = 'RETURN' AND action_at BETWEEN ? AND ?;

-- 입고 유형별 건수 통계
SELECT log_type, COUNT(*) AS cnt FROM erp_import_log
WHERE action_at BETWEEN ? AND ? GROUP BY log_type;
```

---

## export_scheduled_ea_view (출고 예정 수량 뷰)

summary: 상품별 미출고(출고 예정) 수량을 집계하는 뷰이다. erp_export_schedule_item에서 export_flag가 'Y'가 아닌(미출고) 아이템의 수량을 상품 코드별로 합산한다.
domain: export
related_tables: erp_export_schedule_item, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | varchar(50) | - | 상품 코드 | 'G-0029-0001-0001001' |
| export_scheduled_ea | decimal | - | 출고 예정 수량 (SUM) | |

### 관계
- 원본 테이블: `erp_export_schedule_item` (export_flag <> 'Y' 조건으로 필터)
- `product_code` -> `erp_product_info.product_code`: 상품 정보와 조인하여 상품명 등을 확인

### 비즈니스 맥락
- 재고 현황 대시보드에서 "출고 예정 수량"을 표시할 때 사용한다.
- goods_view 등 다른 뷰에서 LEFT JOIN으로 참조되어 실질적 가용 재고를 계산하는 데 활용된다.
- 뷰 정의: `SELECT product_code, SUM(ea) AS export_scheduled_ea FROM erp_export_schedule_item WHERE export_flag <> 'Y' GROUP BY product_code`

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 상품의 출고 예정 수량 조회
SELECT export_scheduled_ea FROM export_scheduled_ea_view WHERE product_code = ?;

-- 재고 현황에서 가용 재고 계산 시 활용
SELECT p.product_code, p.goods_name,
       s.ea AS current_stock,
       IFNULL(es.export_scheduled_ea, 0) AS scheduled_export,
       s.ea - IFNULL(es.export_scheduled_ea, 0) AS available_stock
FROM erp_product_info p
LEFT JOIN erp_stock_realtime s ON p.product_code = s.productCode
LEFT JOIN export_scheduled_ea_view es ON p.product_code = es.product_code;
```
