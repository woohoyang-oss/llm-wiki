# 주문/발주 도메인 (Order Domain)

---

## erp_order (발주 테이블)

summary: 공급업체에 대한 상품 발주(구매 주문)를 관리하는 핵심 테이블. 발주 제목, 발주처, 운송 방법, 결제 정보(예약금/잔금), PI 번호 등을 저장하며, 해외/국내 발주와 발주외입고를 구분한다.
domain: order
related_tables: erp_order_item, erp_partner, erp_purchase_order, erp_item_shipment, erp_order_payment_history, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| order_seq | int unsigned | PK, AUTO_INCREMENT | 발주 번호 | 1, 2, 3... |
| order_title | varchar(255) | NOT NULL, DEFAULT 'ORDER_TITLE' | 발주 제목 | '키크론 Q1 발주', '[과입고] - ...' |
| description | text | NULL | 비고 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록 일시 | |
| order_member_id | int unsigned | NOT NULL | 발주자 member id | member.id 참조 |
| order_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 발주 일시 | |
| partner_code | varchar(7) | NOT NULL | 발주처 코드 | erp_partner.partner_code 참조 |
| partner_country | varchar(2) | NULL | 파트너 국가 코드 | 'KR', 'CN', 'US' |
| category_code | char | NOT NULL | 카테고리 코드 | 'G' (GTGear), 'K' (Keychron) 등 |
| pay_method | varchar(10) | NULL | 결제 방법 | 'bank' (계좌이체), 'card' (카드), 'cash' (현금) |
| transport | varchar(6) | NOT NULL | 운송 방법 | 'LAND' (육로), 'FLIGHT' (항공), 'MARINE' (해운) |
| upload_file | text | NULL | 업로드 파일 경로 | |
| deposit_price | double | NOT NULL, DEFAULT 0 | 예약금 | |
| deposit_date | datetime | NULL | 예약금 결제일시 | |
| balance_price | double | NOT NULL, DEFAULT 0 | 잔금 | |
| balance_date | datetime | NULL | 잔금 결제일자 | |
| vat_flag | char | NOT NULL, DEFAULT 'N' | 부가세 포함 여부 | 'Y', 'N' |
| delivery_price | double | NOT NULL, DEFAULT 0 | 배송비 | |
| etc_price | double | NOT NULL, DEFAULT 0 | 기타 금액 | |
| parent_order_id | int | NOT NULL, DEFAULT 0 | 과입고 원본 발주 번호 | 0이면 원본, 0이 아니면 과입고 분리 발주 |
| pi_number | varchar(255) | NOT NULL, DEFAULT '' | PI(Proforma Invoice) 번호 | |
| is_external_order | tinyint(1) | DEFAULT 0 | 발주외입고 여부 | 0: 발주, 1: 발주외입고 |
| md_discount | decimal(10,2) | DEFAULT 0.00 | MD 할인가 (음수 허용) | |
| credit_point | decimal(10,2) | DEFAULT 0.00 | 크레딧 포인트 (음수 허용) | |
| shipping_date | date | NULL | 예상 상품 발송 일자 | |

### 인덱스
- `idx_erp_order_date` — (order_date)
- `idx_erp_order_partner_code` — (partner_code)
- `idx_erp_order_optimized` — (category_code, partner_code, is_external_order)
- `idx_erp_order_category_external` — (category_code, is_external_order)
- `idx_order_date_seq` — (order_date, order_seq)
- `erp_order_order_seq_is_external_order_index` — (order_seq, is_external_order)

### 관계
- erp_order.order_seq → erp_order_item.order_seq (1:N): 발주에 포함된 품목
- erp_order.partner_code → erp_partner.partner_code (N:1): 발주처 정보
- erp_order.order_member_id → member.id (N:1): 발주 담당자
- erp_order.order_seq → erp_purchase_order.order_seq (1:N): PO 연결
- erp_order.parent_order_id → erp_order.order_seq (Self-Join): 과입고 분리 발주

### 비즈니스 맥락
- 공급업체에 상품을 구매하기 위한 발주 정보를 관리하는 테이블
- `is_external_order = 0`이 일반 발주, `1`이 발주외입고 (예: 무상 제공, 샘플 등)
- `parent_order_id`가 0이 아닌 경우 과입고로 인해 원본 발주에서 분리된 추가 발주
- 결제는 예약금(deposit) → 잔금(balance) 2단계로 진행되며, erp_order_payment_history에 상세 기록
- 해외 발주 시 transport(FLIGHT/MARINE) + partner_country로 운송 경로 추적
- erp_order → erp_order_item → erp_item_shipment으로 이어지는 입고 프로세스 추적

### 자주 쓰는 쿼리 패턴
- 발주 목록 조회 (금액 합산): `SELECT a.*, SUM(b.buying_price * b.ea) AS total_price FROM erp_order a LEFT JOIN erp_order_item b ON a.order_seq = b.order_seq WHERE a.is_external_order = 0 GROUP BY a.order_seq ORDER BY a.order_seq DESC`
- 카테고리별 발주 목록: `SELECT * FROM erp_order WHERE category_code = 'G' AND is_external_order = 0`
- 발주외입고 조회: `SELECT * FROM erp_order WHERE is_external_order = 1 AND category_code = #{categoryCode}`
- 리드타임 계산 (발주일 → 입고일): `SELECT ROUND(AVG(DATEDIFF(s.import_date, o.order_date)), 0) AS lead_time_days FROM erp_order o INNER JOIN erp_order_item oi ON o.order_seq = oi.order_seq INNER JOIN erp_stock s ON oi.item_seq = s.item_seq WHERE s.import_date IS NOT NULL`

---

## erp_order_item (발주 품목)

summary: 발주(erp_order)에 포함된 개별 상품 품목 정보를 저장한다. 상품코드, 수량, 매입단가, 통화, 환율 정보를 관리하며, 입고 상태(일반/과입고/미입고)를 추적한다.
domain: order
related_tables: erp_order, erp_stock, erp_item_shipment, erp_product_info, erp_purchase_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| item_seq | int unsigned | PK, AUTO_INCREMENT | 품목 시퀀스 | |
| order_seq | int unsigned | NOT NULL | 발주 번호 | erp_order.order_seq 참조 |
| product_code | varchar(30) | NOT NULL | 상품 코드 | 'G-brd0001-gs0001-opt001' |
| ea | int | NOT NULL | 수량 | |
| buying_price | decimal(10,3) | NOT NULL, DEFAULT 0.000 | 매입 단가 | |
| currency | char(3) | NULL | 통화 | 'USD', 'KRW', 'CNY' |
| cur2krw | decimal(10,2) | NOT NULL, DEFAULT 0.00 | 원화 환율 | 1300.00 |
| import_status | char | NOT NULL, DEFAULT 'R' | 입고 상태 | 'R' (일반), 'O' (과입고), 'L' (미입고) |
| po_seq | bigint | NULL | PO 번호 | erp_purchase_order.seq 참조 |

### 인덱스
- `idx_erp_order_item_order_seq` — (order_seq)
- `idx_erp_order_item_product_code` — (product_code)
- `erp_order_item_product_code_item_seq_index` — (product_code, item_seq)
- `idx_orderitem_orderseq_product_item` — (order_seq, product_code, item_seq, ea, buying_price, currency, cur2krw) 커버링 인덱스

### 관계
- erp_order_item.order_seq → erp_order.order_seq (N:1): 소속 발주
- erp_order_item.product_code → erp_product_info.product_code (N:1): 상품 마스터 정보
- erp_order_item.item_seq → erp_item_shipment.item_seq (1:N): 운송/입고 추적
- erp_order_item.item_seq → erp_stock.item_seq (1:N): 재고 연결
- erp_order_item.po_seq → erp_purchase_order.seq (N:1): PO 연결

### 비즈니스 맥락
- 하나의 발주(erp_order)에 여러 상품 품목이 포함될 수 있음
- `buying_price`는 통화 기준 단가이며, `cur2krw`를 곱하면 원화 금액이 됨
- `import_status`로 입고 진행 상태 추적: R(정상), O(과입고 - 주문보다 많이 입고), L(미입고 - 일부 미도착)
- erp_stock과의 JOIN으로 원가 추적 가능 (s.buying_price = oi.buying_price * oi.cur2krw)
- erp_item_shipment과의 JOIN으로 운송 단계별 진행 상황 추적

### 자주 쓰는 쿼리 패턴
- 발주별 품목 조회: `SELECT * FROM erp_order_item WHERE order_seq = #{orderSeq}`
- 원가 조회 (재고 기반): `SELECT s.product_code, s.buying_price, oi.cur2krw FROM erp_stock s JOIN erp_order_item oi ON s.item_seq = oi.item_seq WHERE s.product_code = #{code} AND s.buying_price > 0 AND oi.cur2krw > 100 ORDER BY s.import_date DESC LIMIT 1`
- 미입고 품목 확인: `SELECT * FROM erp_order_item WHERE import_status = 'L'`

---

## erp_order_payment_history (발주 결제 이력)

summary: 발주(erp_order)에 대한 결제 이력을 단계별로 기록한다. 예약금(DEPOSIT), 잔금(BALANCE), 기타(ETC) 세 가지 결제 유형을 관리하며, 각 단계별 환율과 금액을 추적한다.
domain: order
related_tables: erp_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| payment_id | bigint | PK, AUTO_INCREMENT | 결제 이력 ID | |
| order_seq | bigint | NOT NULL | 발주 번호 | erp_order.order_seq 참조 |
| pay_type | enum('DEPOSIT','BALANCE','ETC') | NOT NULL | 결제 유형 | 'DEPOSIT' (예약금), 'BALANCE' (잔금), 'ETC' (기타) |
| currency | varchar(10) | NULL | 통화 | 'USD', 'KRW' |
| cur2krw | decimal(10,2) | NULL | 환율 | |
| pay_method | varchar(100) | NULL | 결제 방법 | |
| deposit_price | decimal(15,2) | NULL | 예약금 금액 | |
| deposit_date | datetime | NULL | 예약금 결제일시 | |
| balance_price | decimal(15,2) | NULL | 잔금 금액 | |
| balance_date | datetime | NULL | 잔금 결제일자 | |
| etc_price | decimal(15,2) | NULL | 기타 금액 | |
| delivery_price | decimal(15,2) | NULL | 배송비 | |
| md_discount | decimal(15,2) | NULL | MD 할인가 | |
| credit_point | decimal(15,2) | NULL | 크레딧 포인트 | |
| created_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 | |
| updated_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일 | |

### 관계
- erp_order_payment_history.order_seq → erp_order.order_seq (N:1): 소속 발주

### 비즈니스 맥락
- 발주 결제를 예약금/잔금/기타 3단계로 분리 관리
- 각 단계별 환율을 별도 기록하여 가중평균 환율 계산에 활용
- pay_type = 'ETC'는 배송비, MD 할인, 크레딧 포인트 등 부수 비용을 기록
- 가중평균 환율: (DEPOSIT 환율 x 예약금 + BALANCE 환율 x 잔금) / (예약금 + 잔금)

### 자주 쓰는 쿼리 패턴
- 발주별 결제 이력 조회: `SELECT * FROM erp_order_payment_history WHERE order_seq = #{orderSeq} ORDER BY payment_id DESC`
- 가중평균 환율 계산: `SELECT ROUND((SUM(CASE WHEN pay_type = 'DEPOSIT' THEN cur2krw * deposit_price ELSE 0 END) + SUM(CASE WHEN pay_type = 'BALANCE' THEN cur2krw * balance_price ELSE 0 END)) / (SUM(CASE WHEN pay_type = 'DEPOSIT' THEN deposit_price ELSE 0 END) + SUM(CASE WHEN pay_type = 'BALANCE' THEN balance_price ELSE 0 END)), 0) FROM erp_order_payment_history WHERE order_seq = #{orderSeq} AND pay_type IN ('DEPOSIT', 'BALANCE')`

---

## erp_order_recv (B2C 수주)

summary: B2C 채널을 통해 접수된 고객 수주를 관리하는 테이블. 수주처(biz_code), 담당자 정보, 입금/출고 상태, 세금계산서 연결 등을 포함하며, erp_order_recv_item과 함께 수주 상세 품목을 관리한다.
domain: order
related_tables: erp_order_recv_item, erp_order_recv_biz, erp_order_recv_memo, finance_tax_invoice, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| recv_seq | int unsigned | PK, AUTO_INCREMENT | 수주 번호 | |
| recv_muid | varchar(30) | NOT NULL, DEFAULT '' | 등록 회원 UID | member.uid 참조 |
| tax_invoice_seq | int | NULL | 세금계산서 번호 | finance_tax_invoice.taxInvoiceSeq 참조 |
| tax_issue_type | varchar(7) | DEFAULT 'fixed' | 세금계산서 발행 타입 | 'fixed' |
| category_code | char | NOT NULL, DEFAULT '' | 카테고리 코드 | 'G', 'K' 등 |
| order_title | varchar(100) | NOT NULL, DEFAULT '' | 수주 제목 | |
| biz_code | char(7) | NOT NULL, DEFAULT '' | 수주처 코드 | erp_order_recv_biz.biz_code 참조 |
| chg_name | varchar(50) | NOT NULL, DEFAULT '' | 담당자 이름 | |
| chg_contact | varchar(50) | NOT NULL, DEFAULT '' | 담당자 연락처 | |
| chg_email | varchar(100) | NOT NULL, DEFAULT '' | 담당자 이메일 | |
| order_date | datetime | NULL | 발주일자 | |
| due_date | datetime | NULL | 입금예정일자 | |
| pay_date | datetime | NULL | 입금일자 | |
| pay_muid | varchar(30) | NULL | 입금확인자 UID | |
| upload_file | text | NULL | 업로드 파일 | |
| description | text | NOT NULL | 비고 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일자 | |
| export_method | varchar(10) | NULL | 출고 요청 방법 | |
| export_schedule_date | datetime | NULL | 출고 예정 일자 | |

### 인덱스
- `idx_recv_category_registDate` — (recv_seq DESC, category_code ASC, regist_date ASC)
- `idx_recv_regist_date_recv_seq` — (regist_date, recv_seq)

### 관계
- erp_order_recv.recv_seq → erp_order_recv_item.recv_seq (1:N): 수주 품목 목록
- erp_order_recv.biz_code → erp_order_recv_biz.biz_code (N:1): 수주처(거래처) 정보
- erp_order_recv.tax_invoice_seq → finance_tax_invoice.taxInvoiceSeq (N:1): 세금계산서 연결
- erp_order_recv.recv_muid → member.uid (N:1): 등록자

### 비즈니스 맥락
- 고객(또는 판매처)으로부터 받은 주문(수주)을 관리
- 수주 → 출고 → 입금 확인 프로세스를 추적
- 세금계산서와 연동하여 재무 처리 가능
- category_code로 브랜드별(GTGear, Keychron 등) 수주 분리 관리

### 자주 쓰는 쿼리 패턴
- 수주 목록 조회 (금액/출고 상태 포함): `SELECT recv.*, SUM(item.export_price * item.ea) AS total_price, SUM(CASE WHEN item.export_date IS NOT NULL THEN 1 ELSE 0 END) AS export_count FROM erp_order_recv recv LEFT JOIN erp_order_recv_item item ON recv.recv_seq = item.recv_seq INNER JOIN erp_order_recv_biz biz ON recv.biz_code = biz.biz_code WHERE recv.category_code = #{categoryCode} GROUP BY recv.recv_seq ORDER BY recv.recv_seq DESC`
- 수주 상세 품목 조회: `SELECT * FROM erp_order_recv a LEFT JOIN erp_order_recv_item b ON a.recv_seq = b.recv_seq WHERE a.recv_seq = #{recvSeq}`

---

## erp_order_recv_item (B2C 수주 품목)

summary: 수주(erp_order_recv)에 포함된 개별 상품 품목 정보를 관리한다. 출고지 정보, 택배사/송장, 입금 확인, 영업주문번호 등을 상품 단위로 추적한다.
domain: order
related_tables: erp_order_recv, erp_category, erp_brand, erp_goods, erp_option

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| recv_item_seq | int unsigned | PK, AUTO_INCREMENT | 수주 상품 번호 | |
| recv_seq | int | NOT NULL | 수주 번호 | erp_order_recv.recv_seq 참조 |
| category_code | char | NOT NULL, DEFAULT '' | 카테고리 코드 | |
| brand_code | char(7) | DEFAULT '' | 브랜드 코드 | |
| goods_code | char(6) | DEFAULT '' | 상품 코드 | |
| option_code | char(7) | NULL | 옵션 코드 | |
| ea | int | NOT NULL, DEFAULT 0 | 수량 | |
| export_price | int | NOT NULL, DEFAULT 0 | 출고가 | |
| description | text | NULL | 비고 | |
| export_name | varchar(50) | NOT NULL, DEFAULT '' | 출고지 이름 | |
| export_contact | varchar(50) | NOT NULL, DEFAULT '' | 출고지 연락처 | |
| export_address | varchar(250) | NOT NULL, DEFAULT '' | 출고지 주소 | |
| export_date | datetime | NULL | 출고 일자 | |
| export_confirm_muid | varchar(30) | NULL | 출고 확인자 UID | |
| courier | varchar(10) | NOT NULL, DEFAULT '' | 택배사 | 'cj', 'lg', 'lt', 'kd' |
| invoice | varchar(255) | NOT NULL, DEFAULT '' | 송장 번호 | |
| pay_date | datetime | NULL | 입금 일자 | |
| pay_confirm_muid | varchar(30) | NULL | 입금 확인자 UID | |
| order_num | varchar(25) | NULL | 영업주문번호 | |
| transport_num | varchar(15) | NULL | 운송지시번호 | |

### 인덱스
- `idx_erp_order_recv_item_recvSeq` — (recv_seq)
- `idx_date` — (export_date, category_code, brand_code)
- `tbnws_erp_order_recv_item_idx01` — (category_code, brand_code, goods_code, option_code)

### 관계
- erp_order_recv_item.recv_seq → erp_order_recv.recv_seq (N:1): 소속 수주
- erp_order_recv_item.(category_code, brand_code, goods_code, option_code) → erp_product_info: 상품 마스터 참조 (조합키)

### 비즈니스 맥락
- 수주 품목별로 출고지 정보(이름/연락처/주소)를 개별 관리
- 출고 일자와 택배사/송장으로 배송 추적
- pay_date로 개별 품목의 입금 확인 상태 추적
- order_num(영업주문번호), transport_num(운송지시번호)으로 물류 시스템 연동

### 자주 쓰는 쿼리 패턴
- 수주별 품목 상세 (상품명 포함): `SELECT eg.goods_name, eo.option_name, a.*, b.* FROM erp_order_recv a LEFT JOIN erp_order_recv_item b ON a.recv_seq = b.recv_seq LEFT JOIN erp_goods eg ON b.category_code = eg.category_code AND b.brand_code = eg.brand_code AND b.goods_code = eg.goods_code LEFT JOIN erp_option eo ON b.option_code = eo.option_code WHERE a.recv_seq = #{recvSeq}`

---

## erp_order_recv_memo (수주 메모)

summary: 수주 거래처(biz_code)에 대한 메모를 기록하는 테이블. 거래처별 특이사항, 요청사항 등을 이력으로 관리한다.
domain: order
related_tables: erp_order_recv_biz

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| recv_memo_seq | int | PK, AUTO_INCREMENT | 메모 번호 | |
| biz_code | varchar(50) | NOT NULL | 거래처 코드 | erp_order_recv_biz.biz_code 참조 |
| recv_memo | text | NOT NULL | 메모 내용 | |
| recv_memo_writer | varchar(50) | NOT NULL | 메모 작성자 | |
| recv_memo_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 메모 일자 | |

### 관계
- erp_order_recv_memo.biz_code → erp_order_recv_biz.biz_code (N:1): 대상 거래처

### 비즈니스 맥락
- 거래처별 커뮤니케이션 이력이나 특이사항을 기록
- 수주 처리 시 참고할 주의사항, 배송 요청사항 등을 관리

### 자주 쓰는 쿼리 패턴
- 거래처별 메모 조회: `SELECT * FROM erp_order_recv_memo WHERE biz_code = #{bizCode} ORDER BY recv_memo_date DESC`

---

## erp_order_recv_biz (B2B 수주 거래처)

summary: B2B 수주 거래처(매출처) 마스터 정보를 관리하는 테이블. 사업자등록번호, 대표자, 업태/종목, 세금계산서 발행 타입, 정산 담당자 정보를 저장한다.
domain: order
related_tables: erp_order_recv, erp_order_recv_biz_item, erp_order_recv_biz_recipient, erp_order_recv_memo, erp_partner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| biz_code | char(7) | DEFAULT 'biz0000' | 업체 코드 | 'biz0001' |
| biz_name | varchar(30) | NOT NULL, DEFAULT '' | 업체명 | |
| biz_description | text | NULL | 업체 설명 | |
| partner_code | char(7) | NOT NULL | 매출처 코드 | erp_partner.partner_code 참조 |
| biz_ceo | varchar(10) | NOT NULL, DEFAULT '' | 대표자명 | |
| biz_num | varchar(12) | NOT NULL, DEFAULT '' | 사업자등록번호 | '123-45-67890' |
| biz_image | varchar(500) | NOT NULL, DEFAULT '' | 사업자등록증 이미지 | |
| biz_contact | varchar(20) | NOT NULL, DEFAULT '' | 업체 연락처 | |
| biz_email | varchar(50) | NOT NULL, DEFAULT '' | 업체 이메일 | |
| biz_address | varchar(250) | NOT NULL, DEFAULT '' | 업체 주소 | |
| biz_type | varchar(100) | NOT NULL, DEFAULT '' | 업태 | |
| biz_item | varchar(100) | NOT NULL, DEFAULT '' | 종목 | |
| tax_issue_type | varchar(7) | NOT NULL, DEFAULT 'fixed' | 세금계산서 발행 타입 | 'fixed' |
| chg_name | varchar(30) | NOT NULL, DEFAULT '' | 정산 담당자 | |
| chg_contact | varchar(15) | NOT NULL, DEFAULT '' | 정산 담당자 연락처 | |
| chg_email | varchar(50) | NOT NULL, DEFAULT '' | 정산 담당자 이메일 | |
| barcode_flag | varchar(1) | DEFAULT 'N' | 바코드 사용 여부 | 'Y', 'N' |
| branch_flag | varchar(1) | NOT NULL, DEFAULT 'N' | 지점 관리 여부 | 'Y', 'N' |

### 인덱스
- `idx_biz_code` — (biz_code)

### 관계
- erp_order_recv_biz.biz_code → erp_order_recv.biz_code (1:N): 이 업체의 수주 목록
- erp_order_recv_biz.biz_code → erp_order_recv_biz_item.biz_code (1:N): 이 업체의 취급 상품
- erp_order_recv_biz.biz_code → erp_order_recv_biz_recipient.biz_code (1:N): 이 업체의 배송지 목록
- erp_order_recv_biz.partner_code → erp_partner.partner_code (N:1): 매출처 파트너 정보

### 비즈니스 맥락
- B2B 거래처 마스터로 세금계산서 발행에 필요한 사업자 정보를 관리
- partner_code로 erp_partner와 연결하여 매출처 관리 체계와 통합
- branch_flag = 'Y'인 업체는 지점별 관리 기능 활성화

### 자주 쓰는 쿼리 패턴
- 전체 거래처 목록: `SELECT * FROM erp_order_recv_biz ORDER BY biz_code`
- 거래처 상세 (배송지 포함): `SELECT * FROM erp_order_recv_biz a INNER JOIN erp_order_recv_biz_recipient b ON a.biz_code = b.biz_code WHERE a.biz_code = #{bizCode}`

---

## erp_order_recv_biz_item (B2B 거래처 취급 상품)

summary: B2B 수주 거래처가 취급하는 상품 목록과 출고가, 재고 비율(%)을 관리한다. 거래처별 개별 상품 출고 조건을 설정한다.
domain: order
related_tables: erp_order_recv_biz

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| biz_item_seq | int unsigned | PK, AUTO_INCREMENT | 수주 상품 번호 | |
| biz_code | char(7) | NOT NULL, DEFAULT 'biz0000' | 거래처 코드 | erp_order_recv_biz.biz_code 참조 |
| category_code | char | NOT NULL, DEFAULT 'G' | 카테고리 코드 | |
| brand_code | char(7) | DEFAULT 'brd0000' | 브랜드 코드 | |
| goods_code | char(6) | DEFAULT 'gs0000' | 상품 코드 | |
| option_code | char(7) | NULL | 옵션 코드 | |
| export_price | int | NOT NULL, DEFAULT 0 | 출고가 | |
| export_perc | int | NOT NULL, DEFAULT 50 | 재고 수량 % | 50 (기본값) |

### 인덱스
- `idx_biz_product_code` — (biz_code, category_code, brand_code, goods_code, option_code)

### 관계
- erp_order_recv_biz_item.biz_code → erp_order_recv_biz.biz_code (N:1): 소속 거래처

### 비즈니스 맥락
- 각 B2B 거래처별로 취급 가능한 상품과 그 출고가를 사전 설정
- export_perc는 현재 재고 대비 해당 거래처에 할당 가능한 비율

### 자주 쓰는 쿼리 패턴
- 거래처별 취급 상품 조회: `SELECT * FROM erp_order_recv_biz_item WHERE biz_code = #{bizCode}`

---

## erp_order_recv_biz_recipient (B2B 거래처 배송지)

summary: B2B 수주 거래처의 배송지(수령인) 정보를 관리한다. 하나의 거래처에 여러 배송지를 등록할 수 있다.
domain: order
related_tables: erp_order_recv_biz

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 시퀀스 | |
| biz_code | char(7) | NOT NULL, DEFAULT '' | 거래처 코드 | erp_order_recv_biz.biz_code 참조 |
| name | varchar(30) | NOT NULL, DEFAULT '' | 수령인 이름 | |
| contact | varchar(30) | NOT NULL, DEFAULT '' | 연락처 | |
| address | varchar(250) | NOT NULL, DEFAULT '' | 주소 | |

### 관계
- erp_order_recv_biz_recipient.biz_code → erp_order_recv_biz.biz_code (N:1): 소속 거래처

### 비즈니스 맥락
- B2B 거래처의 다중 배송지(지점, 물류센터 등)를 관리
- 수주 시 출고 품목별로 배송지를 선택하여 배송 정보를 설정

### 자주 쓰는 쿼리 패턴
- 거래처별 배송지 목록: `SELECT * FROM erp_order_recv_biz_recipient WHERE biz_code = #{bizCode}`

---

## erp_customer_order_list (고객 주문 목록)

summary: 영업팀이 관리하는 고객 주문 통합 목록. 주문번호, 거래처명, 상품명, 수량, 금액, 결제 방법을 기록하며, 판매 분석 및 매출 집계에 활용된다.
domain: order
related_tables: erp_customer_order_partner_list, erp_customer_order_product_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | bigint | PK, AUTO_INCREMENT | 시퀀스 | |
| order_num | varchar(100) | NULL | 주문 번호 | |
| order_date | date | NULL | 주문 일자 | |
| partner_name | varchar(255) | NOT NULL | 거래처명 | erp_customer_order_partner_list.partner_name 참조 |
| product_id | varchar(100) | NULL | 상품 ID | |
| product_name | varchar(512) | NOT NULL | 상품명 | erp_customer_order_product_list.product_name 참조 |
| ea | int | NOT NULL | 수량 | |
| price | int | NOT NULL | 금액 | |
| payment_method | varchar(255) | NULL | 결제 방법 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일 | |

### 인덱스
- `idx_customer_order_list_order_date` — (order_date)
- `idx_customer_order_list_product_name` — (product_name)

### 관계
- erp_customer_order_list.partner_name → erp_customer_order_partner_list.partner_name (N:1): 거래처 분류
- erp_customer_order_list.product_name → erp_customer_order_product_list.product_name (N:1): 상품 분류

### 비즈니스 맥락
- 영업팀에서 관리하는 고객 주문의 통합 뷰 (B2B + B2C 포괄)
- 판매 분석 대시보드에서 모델별/거래처별 매출 집계에 사용
- AI 챗봇의 매출 분석 쿼리에서도 활용됨

### 자주 쓰는 쿼리 패턴
- 모델별 매출 분석: `SELECT product.model_name, SUM(o.ea) AS total_qty, SUM(o.price) AS total_revenue FROM erp_customer_order_list o LEFT JOIN erp_customer_order_product_list product ON o.product_name = product.product_name WHERE o.order_date >= #{startDate} AND o.order_date < #{endDate} GROUP BY product.model_name ORDER BY total_revenue DESC`
- 기간별 거래처 매출: `SELECT partner_name, SUM(price) AS total FROM erp_customer_order_list WHERE order_date BETWEEN #{start} AND #{end} GROUP BY partner_name`

---

## erp_customer_order_partner_list (고객 주문 거래처 마스터)

summary: 고객 주문에서 사용되는 거래처(파트너) 마스터 테이블. 거래처 유형, 카테고리, 담당자를 관리한다.
domain: order
related_tables: erp_customer_order_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| partner_name | varchar(255) | PK | 거래처명 | |
| type | varchar(100) | NOT NULL | 거래처 유형 | |
| category | varchar(50) | NOT NULL | 카테고리 | |
| manager_name | varchar(100) | NULL | 담당자명 | |

### 관계
- erp_customer_order_partner_list.partner_name → erp_customer_order_list.partner_name (1:N): 이 거래처의 주문 목록

### 비즈니스 맥락
- 고객 주문 분석에서 거래처를 유형(type)별, 카테고리별로 분류
- 영업 담당자(manager_name) 배정 관리

### 자주 쓰는 쿼리 패턴
- 전체 거래처 목록: `SELECT * FROM erp_customer_order_partner_list ORDER BY partner_name`

---

## erp_customer_order_product_list (고객 주문 상품 마스터)

summary: 고객 주문에서 사용되는 상품 마스터 테이블. 상품명을 기준으로 브랜드, 모델, 악세서리 타입, 레이아웃, 리퍼브 상태 등의 속성을 관리한다.
domain: order
related_tables: erp_customer_order_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_id | varchar(100) | NULL | 상품 ID | |
| product_name | varchar(512) | PK | 상품명 | |
| brand_name | varchar(100) | DEFAULT '' | 브랜드명 | 'Keychron', 'GTGear' |
| model_name | varchar(100) | DEFAULT '' | 모델명 | 'Q1', 'K2' |
| acc_type | varchar(100) | DEFAULT '' | 악세서리 타입 | |
| layout_name | varchar(100) | DEFAULT '' | 레이아웃명 | '풀배열', 'TKL' |
| refurb_status | varchar(100) | DEFAULT '' | 리퍼브 상태 | |

### 인덱스
- `idx_customer_order_product_list_product_name` — (product_name)

### 관계
- erp_customer_order_product_list.product_name → erp_customer_order_list.product_name (1:N): 이 상품이 포함된 주문 목록

### 비즈니스 맥락
- 고객 주문에서 상품을 브랜드/모델/레이아웃별로 분류하여 분석 가능
- model_name으로 그룹핑하여 모델별 매출 분석 수행
- refurb_status로 리퍼브/신품 구분 가능

### 자주 쓰는 쿼리 패턴
- 브랜드별 상품 목록: `SELECT * FROM erp_customer_order_product_list WHERE brand_name = #{brand}`
- 모델별 매출 분석에 JOIN: `SELECT product.model_name, SUM(o.price) FROM erp_customer_order_list o LEFT JOIN erp_customer_order_product_list product ON o.product_name = product.product_name GROUP BY product.model_name`

---

## erp_purchase_order (발주 PO 관리)

summary: PO(Purchase Order) 생명주기를 관리하는 테이블. PO 번호별로 상품, 수량, 납기일, 발주 연결 상태를 추적하며, DRAFT(임시발주) → ORDERED(발주확정) → PARTIAL(일부입고) → RECEIVED(입고완료) 상태를 order_seq 및 erp_stock 연동으로 판별한다.
domain: order
related_tables: erp_order, erp_order_item, erp_product_info, erp_stock, erp_partner, erp_item_shipment

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint unsigned | PK, AUTO_INCREMENT | 시퀀스 | |
| register_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록 일시 | |
| po_title | varchar(255) | NOT NULL | PO 제목 | |
| po_number | varchar(255) | NOT NULL | PO 번호 | 'PO-2024-001' |
| partner_code | varchar(20) | NOT NULL | 파트너 코드 | erp_partner.partner_code 참조 |
| description | varchar(255) | NOT NULL, DEFAULT '' | 비고 | |
| member_id | varchar(10) | NOT NULL | 생성자 ID | member.id 참조 |
| created_at | timestamp | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 | |
| modified_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일 | |
| brand_code | varchar(20) | NOT NULL | 브랜드 코드 | |
| product_code | varchar(255) | NOT NULL, DEFAULT '' | 상품코드 (G코드) | 'G-brd0001-gs0001' |
| quantity | decimal(10,2) | NOT NULL, DEFAULT 0.00 | 수량 | |
| due_date | datetime | NULL | 납기일 | |
| order_seq | bigint | NULL | 발주번호 (erp_order 연결) | NULL이면 DRAFT, 값이 있으면 ORDERED 이상 |

### 인덱스
- `idx_purchase_order_product_orderseq` — (product_code, order_seq)

### 관계
- erp_purchase_order.order_seq → erp_order.order_seq (N:1): 확정된 발주와 연결
- erp_purchase_order.product_code → erp_product_info.product_code (N:1): 상품 마스터
- erp_purchase_order.partner_code → erp_partner.partner_code (N:1): 공급 파트너

### 비즈니스 맥락
- PO 상태 판별 로직:
  - `order_seq IS NULL` → **DRAFT** (임시발주, 아직 발주 미확정)
  - `order_seq IS NOT NULL AND imported_qty = 0` → **ORDERED** (발주확정, 미입고)
  - `order_seq IS NOT NULL AND 0 < imported_qty < quantity` → **PARTIAL** (일부입고)
  - `order_seq IS NOT NULL AND imported_qty >= quantity` → **RECEIVED** (입고완료)
- 입고 수량 판별: `erp_order_item → erp_stock JOIN`으로 import_date IS NOT NULL인 재고 수를 카운트
- 하나의 po_number에 여러 상품(product_code)이 포함될 수 있음
- AI 재고 분석에서 미입고 발주 수량(pending_order_qty) 계산에 핵심 사용

### 자주 쓰는 쿼리 패턴
- PO 목록 조회 (상태 포함): `SELECT po.po_number, po.product_code, po.quantity, CASE WHEN po.order_seq IS NULL THEN 'DRAFT' WHEN IFNULL(imp.imported_qty, 0) >= po.quantity THEN 'RECEIVED' WHEN IFNULL(imp.imported_qty, 0) > 0 THEN 'PARTIAL' ELSE 'ORDERED' END AS status FROM erp_purchase_order po LEFT JOIN (SELECT oi.order_seq, oi.product_code, COUNT(s.stock_seq) AS imported_qty FROM erp_order_item oi INNER JOIN erp_stock s ON oi.item_seq = s.item_seq WHERE s.import_date IS NOT NULL GROUP BY oi.order_seq, oi.product_code) imp ON po.order_seq = imp.order_seq AND po.product_code = imp.product_code WHERE po.register_date >= DATE_SUB(CURDATE(), INTERVAL 180 DAY) ORDER BY po.register_date DESC`
- 미확정 PO 목록: `SELECT * FROM erp_purchase_order WHERE order_seq IS NULL ORDER BY register_date DESC`
- 전체 PO 조회 (발주/패킹/BL 연결): `SELECT a.*, SUM(d.ea) AS sum_quantity, GROUP_CONCAT(DISTINCT c.pi_number) AS pi_number FROM erp_purchase_order a LEFT JOIN erp_order_item b ON b.order_seq = a.order_seq LEFT JOIN erp_order c ON b.order_seq = c.order_seq LEFT JOIN erp_item_shipment d ON b.item_seq = d.item_seq GROUP BY a.seq`

---

## erp_coupang (쿠팡 SKU 마스터)

summary: 쿠팡 로켓그로스 연동을 위한 SKU 마스터 테이블. 쿠팡 SKU ID와 내부 상품코드(product_code)의 매핑, 납품가, 크롤링 설정 등을 관리한다.
domain: order
related_tables: erp_coupang_order, erp_product_info, coupang_inventory_daily_log

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 시퀀스 | |
| sku_id | varchar(10) | NOT NULL | 쿠팡 SKU ID | |
| sku_name | varchar(200) | NULL | SKU 상품명 | |
| sku_delivery_price | int | NULL | SKU 납품가 | |
| product_code | varchar(30) | NULL | 내부 상품 코드 | erp_product_info.product_code 참조 |
| category_code | char | NOT NULL, DEFAULT '' | 카테고리 코드 | |
| brand_code | char(7) | NOT NULL, DEFAULT '' | 브랜드 코드 | |
| goods_code | char(6) | NOT NULL, DEFAULT '' | 상품 코드 | |
| option_code | varchar(10) | NULL | 옵션 코드 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| crawl_price_collect_flag | char | DEFAULT 'N' | 크롤링 여부 | 'Y', 'N' |
| crawl_price_alert_flag | char | DEFAULT 'N' | 가격 알럿 여부 | 'Y', 'N' |
| status | varchar(1) | NOT NULL, DEFAULT 'Y' | 상태 | 'Y' (활성), 'N' (비활성) |

### 인덱스
- `idx_sku_id` — (sku_id)
- `idx_product_code` — (product_code)
- `idx_brand_code` — (brand_code)
- `idx_category_code` — (category_code)
- `idx_goods_code` — (goods_code)
- `idx_option_code` — (option_code)
- `idx_regdate_desc` — (regist_date DESC)

### 관계
- erp_coupang.product_code → erp_product_info.product_code (N:1): 내부 상품 정보
- erp_coupang.sku_id → erp_coupang_order.sku_id (1:N): 이 SKU의 발주 이력
- erp_coupang.sku_id → coupang_inventory_daily_log.sku_id (1:N): 일별 판매 로그

### 비즈니스 맥락
- 쿠팡 로켓그로스 채널에서 관리되는 SKU와 내부 ERP 상품코드를 매핑
- sku_delivery_price는 쿠팡에 납품하는 가격
- 쿠팡 발주 체크리스트(재고 현황 분석)에서 핵심 기준 테이블로 사용
- 쿠팡 판매량 분석에서 coupang_inventory_daily_log와 JOIN하여 출고/입고량 집계

### 자주 쓰는 쿼리 패턴
- SKU 목록 조회 (상품 정보 포함): `SELECT c.*, i.brand_name, i.goods_name, i.option_name FROM erp_coupang c LEFT JOIN erp_product_info i ON c.product_code = i.product_code WHERE IFNULL(c.status, 'Y') = 'Y'`
- SKU별 재고/판매 분석: `SELECT a.product_code, b.ea AS stock, e.one_week_order_ea, e.one_month_order_ea FROM erp_coupang d LEFT JOIN erp_stock_realtime b ON d.product_code = b.productCode LEFT JOIN erp_coupang_order e ON d.sku_id = e.sku_id`

---

## erp_coupang_order (쿠팡 발주 이력)

summary: 쿠팡 로켓그로스에서 발생한 발주(입고 요청) 이력을 기록하는 테이블. 발주번호(order_seq)별 SKU, 수량, 발주일자를 관리하며, 주/월/분기별 발주 추세 분석에 활용된다.
domain: order
related_tables: erp_coupang

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 시퀀스 | |
| order_seq | int | NOT NULL | 쿠팡 발주번호 | UK(order_seq, sku_id) |
| sku_id | varchar(10) | NOT NULL | SKU ID | erp_coupang.sku_id 참조, UK(order_seq, sku_id) |
| sku_name | varchar(255) | NULL | SKU 상품명 | |
| ea | int | NULL | 발주 수량 | |
| order_date | date | NULL | 발주 일자 | |

### 인덱스
- `uk_order_seq` — UNIQUE (order_seq, sku_id)
- `idx_sku_id` — (sku_id)

### 관계
- erp_coupang_order.sku_id → erp_coupang.sku_id (N:1): SKU 마스터

### 비즈니스 맥락
- 쿠팡에서 보내오는 발주(재고 보충 요청)를 기록
- ON DUPLICATE KEY UPDATE로 동일 발주번호+SKU 조합의 중복 방지
- 주/월/3개월 단위의 발주 추세를 집계하여 쿠팡 체크리스트에서 재고 분석에 활용

### 자주 쓰는 쿼리 패턴
- 발주 이력 조회: `SELECT b.product_code, a.order_seq, b.sku_id, a.sku_name, a.ea, a.order_date FROM erp_coupang_order a LEFT JOIN erp_coupang b ON a.sku_id = b.sku_id ORDER BY a.order_seq DESC`
- SKU별 주간/월간/분기 발주 집계: `SELECT sku_id, SUM(CASE WHEN order_date >= DATE_ADD(CURDATE(), INTERVAL -1 WEEK) THEN 1 ELSE 0 END) AS one_week_order_ea, SUM(CASE WHEN order_date >= DATE_ADD(CURDATE(), INTERVAL -1 MONTH) THEN 1 ELSE 0 END) AS one_month_order_ea, SUM(CASE WHEN order_date >= DATE_ADD(CURDATE(), INTERVAL -3 MONTH) THEN 1 ELSE 0 END) AS three_month_order_ea FROM erp_coupang_order GROUP BY sku_id`

---

## non_sabangnet_b2c_orders (사방넷 외 B2C 주문)

summary: 사방넷을 통하지 않는 B2C 채널(예: Shopify, 자사몰 등)의 주문 데이터를 수집하는 테이블. 채널명, 주문번호, 상품, 금액, 주문 상태를 관리한다.
domain: order
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 시퀀스 | |
| channel_name | varchar(20) | NOT NULL | 채널명 | UK(channel_name, order_number) |
| order_number | varchar(255) | NOT NULL | 주문 번호 | UK(channel_name, order_number) |
| order_date | datetime | NOT NULL | 주문 일시 | |
| product_code | varchar(255) | NOT NULL | 상품 코드 | |
| product_name | text | NULL | 상품명 | |
| option_name | varchar(255) | NOT NULL, DEFAULT '' | 옵션명 | |
| ea | int unsigned | NOT NULL, DEFAULT 0 | 수량 | |
| total_price | decimal(10,2) | NOT NULL, DEFAULT 0.00 | 총 금액 | |
| order_status | varchar(20) | NOT NULL | 주문 상태 | |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성일 | |
| modified_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일 | |

### 인덱스
- `unique_order_item` — UNIQUE (channel_name, order_number)
- `idx_channel_order` — (channel_name, order_number)
- `idx_order_date` — (order_date)
- `idx_product_code` — (product_code)

### 관계
- non_sabangnet_b2c_orders.product_code → erp_product_info.product_code (N:1): 상품 정보 (논리적)

### 비즈니스 맥락
- 사방넷에서 처리되지 않는 채널(Shopify Global 등)의 B2C 주문을 별도 수집
- channel_name + order_number의 유니크 제약으로 중복 주문 방지
- 전체 판매 채널 통합 분석 시 사방넷 주문 데이터와 함께 사용

### 자주 쓰는 쿼리 패턴
- 채널별 주문 목록: `SELECT * FROM non_sabangnet_b2c_orders WHERE channel_name = #{channel} AND order_date BETWEEN #{start} AND #{end}`
- 채널별 매출 집계: `SELECT channel_name, SUM(total_price) AS revenue, SUM(ea) AS qty FROM non_sabangnet_b2c_orders WHERE order_date BETWEEN #{start} AND #{end} GROUP BY channel_name`

---

## erp_target_order (브랜드별 매출 목표)

summary: 브랜드별 매출 목표 금액을 설정하는 테이블. 발주 목표 관리에 사용된다.
domain: order
related_tables: erp_brand

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 시퀀스 | |
| brand_code | varchar(10) | NOT NULL | 브랜드 코드 | erp_brand.brand_code 참조 |
| target_amount | bigint | DEFAULT 0 | 목표 금액 | |
| createdBy | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 | |

### 관계
- erp_target_order.brand_code → erp_brand.brand_code (N:1): 대상 브랜드

### 비즈니스 맥락
- 브랜드별로 매출(또는 발주) 목표 금액을 설정
- 실적 대비 목표 달성률을 분석하는 데 활용

### 자주 쓰는 쿼리 패턴
- 브랜드별 목표 조회: `SELECT * FROM erp_target_order WHERE brand_code = #{brandCode}`

---

## erp_target_amount (파트너별 발주 목표 금액)

summary: 년도별/파트너별 발주 목표 금액을 설정하는 테이블. 공급업체별 연간 발주 계획 금액을 관리한다.
domain: order
related_tables: erp_partner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 시퀀스 | |
| year | varchar(4) | NOT NULL | 년도 | '2024', '2025' |
| partner_code | varchar(20) | NOT NULL | 파트너 코드 | erp_partner.partner_code 참조 |
| order_target_amount | bigint | NOT NULL | 발주 목표 금액 | |

### 관계
- erp_target_amount.partner_code → erp_partner.partner_code (N:1): 대상 파트너

### 비즈니스 맥락
- 공급업체별로 연간 발주 목표 금액을 설정하여 구매 계획 관리
- 실제 발주 금액과 비교하여 파트너별 발주 달성률 분석 가능

### 자주 쓰는 쿼리 패턴
- 년도별 파트너 목표 조회: `SELECT * FROM erp_target_amount WHERE year = #{year}`
- 목표 대비 실적: `SELECT t.partner_code, t.order_target_amount, SUM(oi.buying_price * oi.ea * oi.cur2krw) AS actual_amount FROM erp_target_amount t LEFT JOIN erp_order o ON t.partner_code = o.partner_code LEFT JOIN erp_order_item oi ON o.order_seq = oi.order_seq WHERE t.year = #{year} GROUP BY t.partner_code`

---

## erp_customer_month_targert_sales (월별 브랜드 매출 목표)

summary: 년/월/브랜드별 월간 매출 목표를 설정하는 테이블. 영업팀의 월별 매출 목표 관리에 사용된다.
domain: order
related_tables: erp_customer_order_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| year | varchar(10) | UK(year, month, brand) | 년도 | '2024' |
| month | varchar(10) | UK(year, month, brand) | 월 | '01', '12' |
| brand | varchar(100) | UK(year, month, brand) | 브랜드명 | 'Keychron', 'GTGear' |
| target_sales | int | NULL | 목표 매출액 | |

### 인덱스
- `erp_customer_month_targert_sales_pk` — UNIQUE (year, month, brand)

### 관계
- 논리적으로 erp_customer_order_list와 연계하여 브랜드별 실적 대비 목표 분석

### 비즈니스 맥락
- 브랜드별 월간 매출 목표를 설정하고, erp_customer_order_list의 실적과 비교하여 달성률 분석
- year + month + brand 조합이 유니크 키로 중복 방지

### 자주 쓰는 쿼리 패턴
- 월별 목표 조회: `SELECT * FROM erp_customer_month_targert_sales WHERE year = #{year} AND month = #{month}`
- 브랜드별 목표 대비 실적: `SELECT t.brand, t.target_sales, SUM(o.price) AS actual_sales FROM erp_customer_month_targert_sales t LEFT JOIN erp_customer_order_list o ON t.brand = (SELECT brand_name FROM erp_customer_order_product_list WHERE product_name = o.product_name LIMIT 1) WHERE t.year = #{year} AND t.month = #{month} GROUP BY t.brand`
