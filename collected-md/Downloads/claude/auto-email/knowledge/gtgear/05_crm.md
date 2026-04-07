# CRM 도메인 테이블 정의서

---

## crm_as (A/S 테이블)

summary: A/S(애프터서비스) 접수 건을 관리하는 핵심 테이블. 고객 정보, 담당자, 접수/시작/완료 일시를 기록하며, 상태는 start_date와 complete_date의 NULL 여부로 판단한다.
domain: crm
related_tables: crm_as_product, crm_as_history, crm_as_memo, crm_as_notify, crm_as_export, crm_return_management, crm_report, as_board_rel, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | A/S 접수 번호 | 1, 2, 3... |
| customer_name | varchar(30) | NULL | 구매자명 | '홍길동' |
| customer_phone | varchar(20) | NULL | 구매자 전화번호 | '010-1234-5678' |
| customer_email | varchar(255) | NULL | 구매자 이메일 | 'test@email.com' |
| customer_address | varchar(255) | NULL | 구매자 주소 | '서울시 강남구...' |
| customer_postal_code | varchar(10) | NULL | 구매자 우편번호 | '06100' |
| customer_environment | varchar(100) | NULL | 구매자 주 사용환경 | '사무실', '가정' |
| description | text | NULL | 비고 | 자유 텍스트 |
| charge_member_id | int unsigned | NULL, FK→member(id) | 담당자 멤버 ID (정) | |
| sub_charge_member_id | int unsigned | NULL | 담당자 멤버 ID (부) | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| start_date | datetime | NULL | 처리 시작일시 | NULL=미접수 |
| complete_date | datetime | NULL | 처리 종료일시 | NULL=미완료 |
| feedback_flag | char | NULL, DEFAULT 'N' | 만족도조사 메세지 전송 여부 | 'Y', 'N' |

### 관계
- `crm_as_product.as_seq` → `crm_as.seq` (1:N) - A/S 건당 여러 제품
- `crm_as_export.as_seq` → `crm_as.seq` - 출고 관리
- `crm_return_management.as_seq` → `crm_as.seq` - 회수 관리
- `crm_as_history.as_seq` → `crm_as.seq` - 히스토리
- `crm_as_memo.as_seq` → `crm_as.seq` - 메모
- `as_board_rel.as_seq` → `crm_as.seq` - 젠데스크 티켓 연결
- `charge_member_id` → `member.id` - 담당자

### 비즈니스 맥락
- **상태 판별 로직**: status 컬럼이 없음. `start_date`와 `complete_date`의 NULL 여부로 상태를 판단한다.
  - `start_date IS NULL AND complete_date IS NULL` → 접수(대기)
  - `start_date IS NOT NULL AND complete_date IS NULL` → 진행중
  - `complete_date IS NOT NULL` → 완료
- 담당자는 정/부 2명까지 배정 가능
- 젠데스크 티켓과 연결하여 외부 CS 시스템과 통합 관리

### 자주 쓰는 쿼리 패턴
```sql
-- AS 처리율 분석 (AI 쿼리: AS_STATUS)
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN start_date IS NOT NULL AND complete_date IS NULL THEN 1 ELSE 0 END) AS in_progress,
    SUM(CASE WHEN start_date IS NULL AND complete_date IS NULL THEN 1 ELSE 0 END) AS pending,
    ROUND(SUM(CASE WHEN complete_date IS NOT NULL THEN 1 ELSE 0 END)*100.0/NULLIF(COUNT(*), 0), 1) AS completion_rate
FROM crm_as
WHERE start_date >= %s OR complete_date IS NULL;

-- AS 전체 백로그 (AI 쿼리: AS_BACKLOG)
SELECT COUNT(*) AS backlog_count
FROM crm_as
WHERE complete_date IS NULL;

-- AS 목록 조회 (담당자 JOIN, 젠데스크 티켓 포함)
SELECT crm_as.*, charge_member.name AS charge_member_name,
       GROUP_CONCAT(board.ticket_id) AS ticketList
FROM crm_as
LEFT JOIN member AS charge_member ON crm_as.charge_member_id = charge_member.id
LEFT JOIN (as_board_rel rel JOIN gtgear.gtgear_forum_board board ON rel.board_id = board.id) AS board
    ON crm_as.seq = board.as_seq
GROUP BY crm_as.seq;
```

---

## crm_as_alimtalk_template (A/S 알림톡 템플릿)

summary: A/S 진행 단계별 알림톡 발송에 사용할 템플릿 매핑 정보를 관리하는 테이블. 카테고리, 브랜드, 유/무상 여부에 따라 다른 템플릿을 적용한다.
domain: crm
related_tables: crm_as, crm_as_notify, alimtalk_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| category_code | char | NULL | 카테고리 코드 | 'F', 'G' |
| brand_code | char(10) | NULL | 브랜드 코드 | 'brd0001' |
| notify_phase | tinyint unsigned | NOT NULL, DEFAULT 0 | 알림톡 단계 | 0, 1, 2, 3 |
| charge_flag | char | NOT NULL, DEFAULT 'N' | 무상/유상 플래그 (무상='N') | 'Y'=유상, 'N'=무상 |
| template_code | varchar(10) | NULL, FK→alimtalk_template | 알림톡 템플릿 코드 | |

### 관계
- `template_code` → `alimtalk_template.template_code` - 실제 알림톡 템플릿

### 비즈니스 맥락
- 카테고리+브랜드+유무상+단계 조합으로 적절한 알림톡 템플릿을 자동 선택
- notify_phase는 A/S 처리 단계 (접수확인, 진단완료, 수리완료 등)에 대응

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 조건에 맞는 알림톡 템플릿 조회
SELECT template_code
FROM crm_as_alimtalk_template
WHERE category_code = ? AND brand_code = ? AND notify_phase = ? AND charge_flag = ?;
```

---

## crm_as_export (A/S 출고 관리)

summary: A/S 처리 후 교환품/수리품을 고객에게 출고(발송)하는 정보를 관리하는 테이블. 택배사, 송장번호, 출고 유형 등을 기록한다.
domain: crm
related_tables: crm_as, crm_as_export_product, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| as_seq | int unsigned | NULL, FK→crm_as(seq) | A/S 번호 | |
| export_type | varchar(30) | NULL | 분류 | '새상품교환', '리퍼브' 등 |
| customer_name | varchar(50) | NULL | 고객명 | |
| customer_phone | varchar(30) | NULL | 고객 전화번호 | |
| customer_address | varchar(255) | NULL | 고객 주소 | |
| customer_postal_code | varchar(10) | NULL | 고객 우편번호 | |
| description | varchar(255) | NULL | 비고 | |
| courier | char(2) | NULL | 택배사 | 'lg', 'lt', 'cj', 'kd' |
| invoice | varchar(20) | NULL | 출고 송장 번호 | |
| is_invoice_requested | char | NOT NULL, DEFAULT 'N' | 택배사 시스템 업로드 여부(수동) | 'Y', 'N' |
| member_id | int unsigned | NULL, FK→member(id) | 등록자 member id | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| complete_date | datetime | NULL | 완료일시 | |

### 관계
- `as_seq` → `crm_as.seq` (N:1) - A/S 건과 연결
- `crm_as_export_product.as_export_seq` → `crm_as_export.seq` (1:N) - 출고 제품 목록
- `member_id` → `member.id` - 등록자

### 비즈니스 맥락
- A/S 처리 결과 고객에게 새상품/리퍼브 등을 발송할 때 생성
- 택배사 코드: lg=로젠, lt=롯데, cj=CJ대한통운, kd=경동
- complete_date가 NULL이면 출고 미완료 상태

### 자주 쓰는 쿼리 패턴
```sql
-- A/S 건별 출고 내역 조회
SELECT * FROM crm_as_export WHERE as_seq = ?;
```

---

## crm_as_export_product (A/S 출고 제품)

summary: A/S 출고 건에 포함된 개별 제품 정보를 관리하는 테이블. 출고 재고, 리퍼브 등급, 출고요청, 취소 사유 등을 기록한다.
domain: crm
related_tables: crm_as_export, crm_as_product, erp_stock, erp_export_request

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| as_export_seq | int unsigned | NOT NULL, FK→crm_as_export(seq) | A/S 출고 번호 | |
| as_product_seq | int unsigned | NULL, FK→crm_as_product(seq) | 접수된 A/S 제품 번호 | |
| product_code | varchar(30) | NOT NULL | 제품 코드 | 'G-brd0001-gs0001-opt0001' |
| refurbish_grade | char | NULL | 리퍼브 등급 | 'A', 'B', 'C' |
| export_stock_seq | int unsigned | NULL, FK→erp_stock(stock_seq) | 출고된 재고 번호 | |
| export_request_seq | int | NULL, FK→erp_export_request(request_id) | 출고요청 번호 | |
| export_date | datetime | NULL | 출고일시 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| cancel_date | datetime | NULL | 출고 취소일시 | |
| cancel_reason | varchar(100) | NULL | 취소 사유 | 'CUSTOMER_CANCEL', 'STOCK_NOT_FOUND' |

### 관계
- `as_export_seq` → `crm_as_export.seq` (N:1, CASCADE DELETE)
- `as_product_seq` → `crm_as_product.seq` - 접수 제품과 연결
- `export_stock_seq` → `erp_stock.stock_seq` - 출고 재고
- `export_request_seq` → `erp_export_request.request_id` - 출고 요청

### 비즈니스 맥락
- 하나의 A/S 출고 건에 여러 제품이 포함될 수 있음
- cancel_reason: CUSTOMER_CANCEL(고객 요청 취소), STOCK_NOT_FOUND(실재고 없음) 등
- 리퍼브 등급이 있으면 리퍼브 재고에서 출고

### 자주 쓰는 쿼리 패턴
```sql
-- 출고 건별 제품 목록
SELECT * FROM crm_as_export_product WHERE as_export_seq = ?;
```

---

## crm_as_history (A/S 히스토리 테이블)

summary: A/S 건의 처리 과정을 시간순으로 기록하는 히스토리 테이블. 상태 변경, 담당자 변경, 제품 추가 등 모든 변경 이력을 추적한다.
domain: crm
related_tables: crm_as, crm_as_product, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 기록일시 | |
| member_id | int unsigned | NULL | 작업자 멤버 ID | |
| as_seq | int unsigned | NULL | A/S 번호 | |
| product_seq | int | NULL | A/S 제품 번호 | |
| history_type | varchar(50) | NULL | 히스토리 분류 | |
| content | varchar(255) | NULL | 히스토리 내용 | |
| remark | varchar(255) | NULL | 히스토리 비고 | |
| updated_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 업데이트 일시(비고 수정일시) | |
| update_member_id | int unsigned | NULL | 업데이트 멤버 ID(비고 수정자 ID) | |

### 관계
- `as_seq` → `crm_as.seq` - A/S 건 참조
- `member_id` → 작업자 (member 테이블)

### 비즈니스 맥락
- A/S 처리 전 과정의 감사 추적(audit trail) 용도
- history_type으로 변경 종류를 분류
- remark는 나중에 수정 가능하며, 수정 시 updated_at과 update_member_id가 갱신됨

### 자주 쓰는 쿼리 패턴
```sql
-- A/S 건의 히스토리 조회
SELECT h.*, m.name AS member_name
FROM crm_as_history h
LEFT JOIN member m ON h.member_id = m.id
WHERE h.as_seq = ?
ORDER BY h.created_at DESC;
```

---

## crm_as_memo (A/S 메모 테이블)

summary: A/S 건에 대한 내부 메모를 관리하는 테이블. 담당자 간 커뮤니케이션 및 메모 기록 용도로 사용한다.
domain: crm
related_tables: crm_as, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| as_seq | int unsigned | NOT NULL | as seq | |
| member_id | int unsigned | NOT NULL | 등록자 member ID | |
| content | varchar(255) | NOT NULL | 메모 내용 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| update_date | datetime | NULL | 수정일시 | |

### 관계
- `as_seq` → `crm_as.seq` - A/S 건 참조
- `member_id` → `member.id` (INDEX) - 등록자

### 비즈니스 맥락
- A/S 건 상세 화면에서 담당자가 작성하는 내부 메모
- 히스토리(crm_as_history)와 별개로, 자유 형식의 메모 기록

### 자주 쓰는 쿼리 패턴
```sql
-- A/S 건의 메모 목록 조회
SELECT m.*, member.name AS member_name
FROM crm_as_memo m
LEFT JOIN member ON m.member_id = member.id
WHERE m.as_seq = ?
ORDER BY m.regist_date DESC;
```

---

## crm_as_notify (A/S 알림 발송 내역)

summary: A/S 처리 단계별로 고객에게 발송한 알림톡/SMS 내역을 기록하는 테이블. 발송 단계, 수신 번호, 템플릿 코드 등을 저장한다.
domain: crm
related_tables: crm_as, crm_as_product, crm_as_alimtalk_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시(전송요청일시) | |
| as_seq | int unsigned | NULL | A/S 번호 | |
| product_seq | int | NULL | A/S 제품 번호 | |
| notify_phase | tinyint unsigned | NULL | A/S 알림 단계 | 0, 1, 2, 3... |
| to_num | varchar(20) | NULL | 수신 전화번호 | '01012345678' |
| template_code | varchar(50) | NULL | 전송 템플릿 코드 | |
| sms_result_seq | int unsigned | NULL | 전송 내역 seq | |

### 관계
- `as_seq` → `crm_as.seq` - A/S 건 참조
- `product_seq` → `crm_as_product.seq` - A/S 제품 참조
- `template_code` → 알림톡 템플릿 코드

### 비즈니스 맥락
- A/S 진행 과정에서 고객에게 자동/수동 발송한 알림톡 이력
- notify_phase와 crm_as_product.notify_phase가 연동되어 현재 발송 단계를 추적
- created_at에 INDEX가 있어 날짜별 발송 내역 조회에 최적화

### 자주 쓰는 쿼리 패턴
```sql
-- A/S 건별 알림 발송 이력
SELECT * FROM crm_as_notify WHERE as_seq = ? ORDER BY created_at DESC;
```

---

## crm_as_product (A/S 상품 테이블)

summary: A/S 접수 건에 포함된 개별 제품 정보를 관리하는 핵심 테이블. 진행 상태, 조치 유형, 진단 내용, 수리 비용, 시리얼 번호 등 A/S 제품별 상세 정보를 기록한다.
domain: crm
related_tables: crm_as, erp_product_info, erp_stock, crm_report, crm_as_export_product, crm_return_product

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| as_seq | int unsigned | NOT NULL, FK→crm_as(seq) | as seq | |
| product_code | varchar(30) | NOT NULL | 제품 코드 | 'G-brd0001-gs0001' |
| quantity | int unsigned | NULL, DEFAULT 1 | 수량 | 1 |
| process_type | varchar(50) | NULL | 진행 상태 | |
| resolution_type | varchar(50) | NULL | 조치 유형 | |
| retrieve_courier | char(2) | NULL | (deprecated) 회수 택배사 | 'cj', 'lt', 'lg', 'kt', 'qi' |
| retrieve_invoice | varchar(50) | NULL | (deprecated) 회수 송장 번호 | |
| retrieve_memo | text | NULL | (deprecated) 회수 고객 배송 메모 | |
| import_date | datetime | NULL | (deprecated) 입고 일시 | |
| expected_diagnosis_days | tinyint unsigned | NOT NULL, DEFAULT 3 | (deprecated) 진단 소요일 | 3 |
| customer_serial_num | varchar(50) | NULL | (deprecated) 고객 입력 시리얼 번호 | |
| serial_num | varchar(50) | NULL | 제품 실물 시리얼 번호 | |
| option_model | varchar(100) | NULL | (deprecated) 옵션 모델 정보 | |
| stock_seq | int unsigned | NULL | 제품 stock seq | |
| repair_period | datetime | NULL | (deprecated) 예상 수리 완료 일시 | |
| is_free_repair | char | NOT NULL, DEFAULT 'N' | 무상수리 여부 | 'Y', 'N' |
| user_cost | int unsigned | NULL | 고객 부담금(원화) | 50000 |
| company_cost | int unsigned | NULL | 투비 부담금(원화) | 30000 |
| diagnose | text | NULL | 진단 내용 | 자유 텍스트 |
| reference_content | text | NULL | 참고 자료(링크 등) | URL 등 |
| return_flag | char | NULL, DEFAULT 'N' | (deprecated) 완료 후 회송 여부 | 'Y', 'N' |
| notify_phase | int unsigned | NOT NULL, DEFAULT 0 | 현재 알림톡 발송 단계 | 0, 1, 2... |

### 관계
- `as_seq` → `crm_as.seq` (N:1, CASCADE DELETE) - A/S 건
- `product_code` → `erp_product_info.product_code` (JOIN) - 제품 상세 정보(goods_name, brand_name 등)
- `stock_seq` → `erp_stock.stock_seq` - 재고 정보
- `crm_report.as_product_seq` → `crm_as_product.seq` - 리포트
- `crm_as_export_product.as_product_seq` → `crm_as_product.seq` - 출고 제품
- `crm_return_product.as_product_seq` → `crm_as_product.seq` - 회수 제품
- `as_material_parts_rel.as_product_seq` → `crm_as_product.seq` - 부품 사용
- `as_material_stock_rel.as_product_seq` → `crm_as_product.seq` - 제품 사용

### 비즈니스 맥락
- **중요**: product_name 컬럼이 없음. 제품명 조회 시 `erp_product_info` JOIN 필수
- diagnose(진단 내용)는 고장 패턴 분석의 핵심 필드
- is_free_repair로 무상/유상 수리를 구분하며, user_cost/company_cost로 비용 분담 관리
- deprecated 컬럼이 많아 새 기능에서는 crm_return_management/crm_as_export를 사용

### 자주 쓰는 쿼리 패턴
```sql
-- AS 고장 패턴 분석 (AI 쿼리: AS_DIAGNOSIS_PATTERNS)
SELECT p.diagnose, COUNT(*) AS cnt
FROM crm_as_product p
JOIN crm_as a ON p.as_seq = a.seq
WHERE a.start_date >= %s
  AND p.diagnose IS NOT NULL AND p.diagnose != ''
GROUP BY p.diagnose
ORDER BY cnt DESC;

-- AS 다발 상품 (AI 쿼리: AS_BY_PRODUCT, erp_product_info JOIN 필수)
SELECT pi.goods_name, pi.option_name, COUNT(*) AS as_count
FROM crm_as_product p
JOIN erp_product_info pi ON p.product_code = pi.product_code
JOIN crm_as a ON p.as_seq = a.seq
WHERE a.start_date >= %s
GROUP BY pi.goods_name, pi.option_name
ORDER BY as_count DESC;

-- A/S 제품 상세 조회 (재고 + 리포트 + 매출처 JOIN)
SELECT product.*, erp_product_info.goods_name, erp_product_info.brand_name,
       report.seq AS report_seq, stock.export_date, stock.selling_price
FROM crm_as_product AS product
LEFT JOIN erp_product_info ON product.product_code = erp_product_info.product_code
LEFT JOIN crm_report AS report ON product.seq = report.as_product_seq
LEFT JOIN erp_stock AS stock ON product.stock_seq = stock.stock_seq
WHERE product.as_seq = ?;
```

---

## crm_call_info (전화 수신 정보)

summary: 전화 수발신 기록을 저장하는 기본 테이블. 발신자, 수신자, 통화 일시만 기록하는 간단한 구조이다.
domain: crm
related_tables: crm_call_memo

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | int unsigned | PK, AUTO_INCREMENT | 인덱스 | |
| sender | varchar(11) | NULL | 발신자 번호 | '01012345678' |
| receiver | varchar(11) | NULL | 수신자 번호 | '0215551234' |
| call_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 통화 일시 | |

### 관계
- `sender` 값으로 `crm_call_memo.sender`와 연결 (전화번호 기반)

### 비즈니스 맥락
- 전화 시스템(Cisco)과 연동하여 자동으로 수신 기록 저장
- 상세 상담 내용은 crm_call_memo에 별도 기록

### 자주 쓰는 쿼리 패턴
```sql
-- 고객 전화번호로 수발신 이력 조회
SELECT idx, receiver, call_date
FROM crm_call_info
WHERE sender = ?;
```

---

## crm_call_memo (전화 상담 메모)

summary: 고객 전화 상담 내용을 기록하는 핵심 CRM 테이블. 인바운드/아웃바운드 구분, 상담 분류, 담당자, 완료 여부, 마케팅 동의 등 상담 관련 전체 정보를 관리한다.
domain: crm
related_tables: crm_call_info, crm_as, customer

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | int unsigned | PK, AUTO_INCREMENT | 인덱스 | |
| channel | varchar(50) | NULL | 채널 | |
| sender | varchar(30) | NULL | 보낸사람(고객 전화번호) | '01012345678' |
| sender_name | varchar(30) | NULL, DEFAULT '' | 고객이름 | '홍길동' |
| receiver | varchar(11) | NULL | 받는사람(상담원 번호) | |
| memo | text | NULL | 메모 | 자유 텍스트 |
| writer | varchar(20) | NULL | 작성자 | |
| finish_flag | char | NULL, DEFAULT 'N' | 완료 플래그 | 'Y', 'N' |
| call_type | char | NULL, DEFAULT 'I' | Inbound / Outbound | 'I'=인바운드, 'O'=아웃바운드 |
| call_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 통화날짜 | |
| call_reason | varchar(20) | NOT NULL, DEFAULT '' | 상담분류 | '문자발송' 등 |
| director | varchar(30) | NULL, DEFAULT '' | 담당자 | |
| important_flag | char | NULL, DEFAULT 'N' | 중요 플래그 | 'Y', 'N' |
| finish_time | datetime | NULL | 완료 날짜 | |
| marketing_flag | char | NULL, DEFAULT 'Y' | 마케팅 문자 플래그 | 'Y', 'N' |
| appointment_time | datetime | NULL | 예약 시간 | |
| appointed_flag | char | NOT NULL, DEFAULT 'N' | 예약 플래그 | 'Y', 'N' |

### 관계
- `sender` 값으로 `crm_call_info.sender`와 연결 (전화번호 기반)
- `sender`로 `crm_as.customer_phone`, `erp_as.user_phone`과 교차 조회 가능

### 비즈니스 맥락
- CS팀의 주요 업무 도구. 전화 상담 시 실시간으로 메모 작성
- call_type: 'I'=고객이 전화한 인바운드, 'O'=상담원이 건 아웃바운드
- marketing_flag: 고객의 마케팅 수신 동의 여부를 가장 최근 메모에서 관리
- appointed_flag: 콜백 예약이 잡혀 있는지 여부
- call_reason이 '문자발송'인 건은 통화 통계에서 제외
- sender(전화번호) 기반으로 crm_as, erp_as 이력을 함께 조회하여 고객 이력을 한눈에 파악

### 자주 쓰는 쿼리 패턴
```sql
-- 날짜 범위별 상담 메모 목록
SELECT * FROM crm_call_memo
WHERE LEFT(call_date, 10) >= ? AND LEFT(call_date, 10) <= ?;

-- 고객 전화번호로 최근 메모 조회
SELECT * FROM crm_call_memo WHERE sender = ? ORDER BY idx DESC LIMIT 1;

-- 마케팅 수신 동의 확인
SELECT marketing_flag FROM crm_call_memo WHERE sender = ? ORDER BY idx DESC LIMIT 1;

-- 콜백 예약 목록
SELECT * FROM crm_call_memo WHERE appointed_flag = 'Y';

-- 상담 통계 차트 (일/주/월별)
SELECT DATE_FORMAT(call_date, ?) AS period,
       COUNT(*) AS total_count,
       SUM(CASE WHEN call_type='o' THEN 1 ELSE 0 END) AS outbound_count,
       SUM(CASE WHEN call_type='i' THEN 1 ELSE 0 END) AS inbound_count
FROM crm_call_memo
WHERE call_reason != '문자발송'
GROUP BY period;
```

---

## crm_refund (교환/반품)

summary: CRM에서 관리하는 교환/반품 처리 테이블. 네이버 주문번호, 제품 정보, 바코드, 시리얼, 개봉 여부, 처리 유형 등 교환/반품의 전체 프로세스를 기록한다.
domain: crm
related_tables: erp_product_info, erp_stock, erp_refurb_stock, erp_parts, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| order_id | varchar(20) | NULL | 네이버 주문번호 | |
| customer_name | varchar(20) | NULL | 고객명 | |
| customer_contact | varchar(30) | NULL | 고객 연락처 | |
| customer_address | varchar(500) | NULL | 고객 주소 | |
| barcode | varchar(30) | NULL | 바코드 번호 | |
| product_code | varchar(20) | NULL | 제품 코드 | |
| serial_num | varchar(30) | NULL | 시리얼 번호 | |
| original_stock_seq | int unsigned | NULL | 최초 출고품 재고 번호 | |
| original_refurb_stock_seq | int unsigned | NULL | 최초 출고 리퍼브 번호 | |
| product_type | char | NOT NULL, DEFAULT 'C' | 제품 타입 | 'C'=일반, 'R'=리퍼브 |
| import_date | datetime | NULL | 입고일시 | |
| is_opened | char | NOT NULL, DEFAULT 'N' | 개봉 여부 | 'Y', 'N' |
| resolution_type | varchar(20) | NULL | 처리 유형 | |
| description | text | NULL | 비고 | |
| refunded_stock_seq | int unsigned | NULL, FK→erp_stock(stock_seq) | 회수 입고된 재고 번호 | |
| refunded_refurb_stock_seq | int unsigned | NULL, FK→erp_refurb_stock(seq) | 회수 입고된 리퍼브 번호 | |
| refunded_parts_seq | int unsigned | NULL, FK→erp_parts(seq) | 회수된 부품 번호 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| complete_date | datetime | NULL | 완료일시 | |
| registrant_id | int unsigned | NULL, FK→member(id) | 등록자 멤버 ID | |

### 관계
- `product_code` → `erp_product_info.product_code` (JOIN)
- `original_stock_seq` → `erp_stock.stock_seq` - 원래 출고된 재고
- `refunded_stock_seq` → `erp_stock.stock_seq` - 회수 입고된 재고
- `refunded_refurb_stock_seq` → `erp_refurb_stock.seq` - 회수 리퍼브 재고
- `refunded_parts_seq` → `erp_parts.seq` - 회수된 부품
- `registrant_id` → `member.id` - 등록자

### 비즈니스 맥락
- A/S와 별개의 독립적인 교환/반품 프로세스
- product_type으로 일반 제품(C)과 리퍼브 제품(R)을 구분
- 회수된 제품은 상태에 따라 재고(erp_stock), 리퍼브(erp_refurb_stock), 부품(erp_parts)으로 입고
- complete_date가 NULL이면 처리 진행 중

### 자주 쓰는 쿼리 패턴
```sql
-- 교환/반품 목록 (제품 정보, 등록자 JOIN)
SELECT refund.*, product.goods_name, product.brand_name, member.name AS registrant_name
FROM crm_refund AS refund
LEFT JOIN erp_product_info AS product ON refund.product_code = product.product_code
LEFT JOIN member ON refund.registrant_id = member.id;
```

---

## crm_report (A/S 리포트 테이블)

summary: A/S 처리 결과를 리포트로 작성하는 테이블. 손실금액, 보상 여부, RMA/CRM 메모, 제조사 보상 대상 여부 등 A/S 완료 후 정산 관련 정보를 관리한다.
domain: crm
related_tables: crm_as, crm_as_product, erp_stock, erp_product_info, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| as_product_seq | int unsigned | NULL, FK→crm_as_product(seq) | as 제품 seq | |
| as_seq | int unsigned | NULL | as seq | |
| customer_name | varchar(30) | NULL | 고객명 | |
| customer_phone | varchar(30) | NULL | 고객 전화번호 | |
| product_code | varchar(20) | NULL | 제품 코드 | |
| serial_num | varchar(100) | NULL | 시리얼 번호 | |
| original_stock_seq | int unsigned | NULL | 최초출고품 재고번호 | |
| lost_amount | double unsigned | NOT NULL, DEFAULT 0 | 손실금액 | 50000.0 |
| compensation_target_flag | char | NOT NULL, DEFAULT 'N' | 제조사 전액 보상 대상 상품 여부 | 'Y', 'N' |
| rma | varchar(255) | NULL | RMA 메모 | |
| crm | varchar(255) | NULL | CRM 메모 | |
| compensation_stock_seq | int unsigned | NULL, FK→erp_stock(stock_seq) | 배상 상품 재고 번호 | |
| compensation_amount | double unsigned | NOT NULL, DEFAULT 0 | 보상금액 | |
| reply | text | NULL | 회신 내용 (deprecated) | |
| description | text | NULL | 비고 | |
| member_id | int unsigned | NOT NULL | 담당자 멤버 ID | |
| report_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 리포트 일시 | |
| complete_date | datetime | NULL | 완료일시 | |

### 관계
- `as_product_seq` → `crm_as_product.seq` (N:1) - A/S 제품
- `compensation_stock_seq` → `erp_stock.stock_seq` - 배상 상품 재고
- `member_id` → `member.id` - 담당자

### 비즈니스 맥락
- A/S 처리 완료 후 제조사 보상/손실 정산을 위한 리포트
- compensation_target_flag='Y'이면 제조사가 전액 보상하는 대상 제품
- lost_amount: 회사 측 손실금액, compensation_amount: 제조사 보상금액
- RMA(Return Merchandise Authorization): 제조사 반품 승인 관련 메모

### 자주 쓰는 쿼리 패턴
```sql
-- A/S 리포트 목록 (제품 정보, 재고 정보, 담당자 JOIN)
SELECT r.*, m.name AS member_name,
       product.goods_name, product.brand_name,
       stock.buying_price AS original_stock_buying_price
FROM crm_report AS r
LEFT JOIN member AS m ON r.member_id = m.id
LEFT JOIN erp_product_info AS product ON r.product_code = product.product_code
LEFT JOIN erp_stock AS stock ON r.original_stock_seq = stock.stock_seq;
```

---

## crm_return_management (회수 관리)

summary: A/S 제품의 회수(수거) 프로세스를 관리하는 테이블. 택배사, 송장번호, 회수 요청 여부 등 고객으로부터 제품을 회수하는 과정을 추적한다.
domain: crm
related_tables: crm_as, crm_return_product, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| as_seq | int unsigned | NULL, FK→crm_as(seq) | A/S 번호 | |
| customer_name | varchar(30) | NULL | 고객명 | |
| customer_phone | varchar(30) | NULL | 전화번호 | |
| customer_address | text | NULL | 주소 | |
| customer_postal_code | varchar(10) | NULL | 우편번호 | |
| courier | char(2) | NULL | 택배사 | 'cj', 'lt', 'lg' |
| original_invoice | varchar(255) | NULL | 원출고송장 번호 | |
| return_invoice | varchar(255) | NULL | 회수송장 번호 | |
| requested_flag | char | NOT NULL, DEFAULT 'N' | 회수 요청 여부 | 'Y', 'N' |
| description | text | NULL | 비고 | |
| member_id | int unsigned | NULL, FK→member(id) | 등록자 멤버 ID | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- `as_seq` → `crm_as.seq` (N:1) - A/S 건
- `crm_return_product.return_seq` → `crm_return_management.seq` (1:N) - 회수 제품 목록
- `member_id` → `member.id` - 등록자

### 비즈니스 맥락
- A/S 접수 후 고객에게서 제품을 회수할 때 생성
- requested_flag: 택배사에 회수 요청을 한 상태인지 여부
- original_invoice: 원래 고객에게 보낸 송장, return_invoice: 회수용 송장
- crm_as_product의 deprecated 회수 관련 컬럼을 대체하는 신규 테이블

### 자주 쓰는 쿼리 패턴
```sql
-- 회수 관리 목록 (A/S별, 제품 수/회수 수 포함)
SELECT management.*, member.name AS member_name,
       COUNT(product.seq) AS product_count,
       SUM(CASE WHEN product.import_date IS NULL THEN 0 ELSE 1 END) AS returned_count
FROM crm_return_management AS management
LEFT JOIN member ON management.member_id = member.id
LEFT JOIN crm_return_product AS product ON management.seq = product.return_seq
WHERE management.as_seq = ?
GROUP BY management.seq;
```

---

## crm_return_product (회수 관리 대상 제품)

summary: 회수 관리 건에 포함된 개별 제품 정보를 관리하는 테이블. 제품 코드, 수량, 실제 회수일시 등을 기록한다.
domain: crm
related_tables: crm_return_management, crm_as_product, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| return_seq | int unsigned | NULL, FK→crm_return_management(seq) | 회수 관리 번호 | |
| as_product_seq | int unsigned | NULL, FK→crm_as_product(seq) | A/S 제품 번호 | |
| product_code | varchar(30) | NOT NULL | 제품 코드 | |
| ea | int unsigned | NOT NULL, DEFAULT 1 | 수량 | 1 |
| import_date | datetime | NULL | 회수일시 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- `return_seq` → `crm_return_management.seq` (N:1, CASCADE DELETE) - 회수 관리 건
- `as_product_seq` → `crm_as_product.seq` - A/S 제품과 연결
- `product_code` → `erp_product_info.product_code` (JOIN) - 제품 상세 정보

### 비즈니스 맥락
- import_date가 NULL이면 아직 회수되지 않은 상태
- import_date가 있으면 실제 제품이 입고된 상태
- 회수 진행률은 `SUM(import_date IS NOT NULL) / COUNT(*)` 로 계산

### 자주 쓰는 쿼리 패턴
```sql
-- 회수 건별 제품 목록 (제품명 JOIN)
SELECT product.*, erp_product_info.goods_name, erp_product_info.option_name
FROM crm_return_product AS product
LEFT JOIN erp_product_info ON product.product_code = erp_product_info.product_code
WHERE product.return_seq = ?;
```

---

## customer (고객 DB)

summary: B2C 채널의 고객 마스터 데이터를 관리하는 테이블. 고객명, 전화번호, 이메일, 주소 등 기본 정보를 저장하며, 전화번호로 유니크 식별한다.
domain: crm
related_tables: crm_call_memo, crm_as, vip_history, customer_refund

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | id | |
| name | varchar(20) | NULL | 이름 | '홍길동' |
| phone | varchar(30) | NOT NULL, UNIQUE | 휴대전화번호 | '010-1234-5678' |
| email | varchar(255) | NULL | 이메일 | |
| address | varchar(255) | NULL | 주소 | |
| postal_code | varchar(10) | NULL | 우편번호 | |

### 관계
- `customer.id` → `vip_history.customer_id` - VIP 이력
- `customer.phone` → `crm_call_memo.sender` (전화번호 기반 JOIN)
- `customer.phone` → `crm_as.customer_phone` (전화번호 기반 JOIN)

### 비즈니스 맥락
- 전화번호(phone)가 고객 식별의 핵심 키 (UNIQUE 제약)
- CRM 전화 상담 시 phone으로 고객 정보 자동 매칭
- VIP 고객 여부는 vip_history 테이블과 JOIN하여 확인
- 전화번호 비교 시 하이픈 제거 필요: `REPLACE(phone, "-", "")`

### 자주 쓰는 쿼리 패턴
```sql
-- 전화번호로 고객 조회 (VIP 여부 확인)
SELECT selection_date, purchase_amount
FROM vip_history
WHERE customer_id = (
    SELECT id FROM customer
    WHERE REPLACE(phone, "-", "") LIKE CONCAT("%", REPLACE(?, "-", ""), "%")
);

-- 고객 ID 조회 (전화번호 기반)
SELECT id FROM customer WHERE phone = ? LIMIT 1;
```

---

## customer_refund (고객 환불 기록)

summary: 고객 단위의 환불 요청을 관리하는 테이블. 주문번호, 환불 금액, 계좌 정보, 환불 사유, 현금영수증 취소 여부 등 환불 처리에 필요한 전체 정보를 기록한다.
domain: crm
related_tables: member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | seq | |
| mUid | varchar(30) | NOT NULL, DEFAULT '' | 등록자 멤버 UID | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| title | varchar(255) | NOT NULL, DEFAULT '' | 제목 | |
| order_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 주문일시 | |
| order_num | varchar(250) | NOT NULL, DEFAULT '' | 주문번호 | |
| amount | int | NOT NULL | 환불 금액 | 50000 |
| bank | varchar(15) | NOT NULL, DEFAULT '' | 은행 | '국민', '신한' |
| account_holder | varchar(30) | NOT NULL, DEFAULT '' | 예금주 | |
| account_number | varchar(30) | NOT NULL, DEFAULT '' | 계좌번호 | |
| refund_requester | varchar(30) | NOT NULL, DEFAULT '' | 환불요청자 | |
| refund_reason | text | NOT NULL | 환불사유 | |
| cash_receipt_cancel_confirm | char | NOT NULL, DEFAULT 'N' | 현금영수증 취소 여부 | 'Y', 'N' |
| is_completed | char | NULL, DEFAULT 'N' | 완료 여부 | 'Y', 'N' |

### 관계
- `mUid` → `member.uid` (JOIN) - 등록자 멤버 정보

### 비즈니스 맥락
- crm_refund(교환/반품)와는 별개의 순수 환불 처리 테이블
- is_completed 토글로 환불 완료/미완료 상태를 관리
- 현금영수증 취소 여부도 함께 관리

### 자주 쓰는 쿼리 패턴
```sql
-- 환불 목록 조회 (등록자 JOIN)
SELECT a.*, b.name AS member_name
FROM customer_refund AS a
LEFT JOIN member AS b ON a.mUid = b.uid;

-- 환불 완료 상태 토글
UPDATE customer_refund
SET is_completed = CASE WHEN is_completed = 'Y' THEN 'N' WHEN is_completed = 'N' THEN 'Y' END
WHERE seq = ?;
```

---

## dw_grade (제품 등급/가용월 데이터)

summary: 제품별 재고 가용 개월수를 일자별로 기록하는 데이터 웨어하우스 테이블. 재고 소진 예측 및 발주 판단에 활용된다.
domain: crm
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| grade_date | date | NOT NULL | 기준 일자 | '2026-03-01' |
| product_code | varchar(100) | NOT NULL | 제품 코드 | 'G-brd0001-gs0001' |
| available_month | double | NULL | 가용 개월수 | 3.5, -1(재고 없음) |

### 관계
- `product_code` → `erp_product_info.product_code` (JOIN) - 제품 상세 정보
- INDEX: (grade_date, product_code), (product_code)

### 비즈니스 맥락
- 매일 배치로 제품별 재고 가용 개월수를 계산하여 적재
- available_month = 현재 재고 / 월평균 판매량
- -1은 판매 이력 없음을 의미
- 가용 개월수가 낮은 제품은 긴급 발주 대상

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 제품의 가용 개월수 추이
SELECT grade_date, available_month
FROM dw_grade
WHERE product_code = ?
ORDER BY grade_date DESC;

-- 최신 날짜 기준 가용 개월수 낮은 제품 (발주 필요)
SELECT product_code, available_month
FROM dw_grade
WHERE grade_date = (SELECT MAX(grade_date) FROM dw_grade)
  AND available_month > 0
ORDER BY available_month ASC;
```

---

## erp_as (ERP A/S 테이블)

summary: ERP 시스템에서 직접 접수된 A/S 건을 관리하는 레거시 테이블. CRM A/S(crm_as)와 별개로 운영되며, 제품 코드 체계(category/brand/goods/option)를 직접 포함한다.
domain: crm
related_tables: erp_brand, erp_goods, erp_option, crm_call_memo

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| as_seq | int unsigned | PK, AUTO_INCREMENT | A/S 번호 | |
| export_seq | int | NULL | 출고 번호 | |
| category_code | char | NOT NULL, DEFAULT '' | 카테고리 코드 | 'F', 'G' |
| brand_code | char(7) | NOT NULL, DEFAULT 'brd0000' | 브랜드 코드 | 'brd0001' |
| goods_code | char(6) | NOT NULL, DEFAULT 'gs0000' | 상품 코드 | 'gs0001' |
| option_code | char(7) | NULL | 옵션 코드 | 'opt0001' |
| user_name | varchar(30) | NOT NULL, DEFAULT '' | 고객명 | |
| user_phone | varchar(30) | NOT NULL, DEFAULT '' | 고객 전화번호 | |
| user_address | varchar(250) | NOT NULL | 고객 주소 | |
| user_postal_code | varchar(250) | NULL | 고객 우편번호 | |
| purchase_date | date | NULL | 구매일 | |
| reason | text | NOT NULL | 접수 사유 | |
| zendesk_link | varchar(250) | NULL, DEFAULT '' | 젠데스크 링크 | |
| import_confirm_call | char | NOT NULL, DEFAULT 'N' | 입고 확인 전화 | 'Y', 'N' |
| import_confirm_sms | char | NOT NULL, DEFAULT 'N' | 입고 확인 SMS | 'Y', 'N' |
| start_sms | char | NOT NULL, DEFAULT 'N' | 수리 시작 SMS | 'Y', 'N' |
| reason_confirm_call | char | NOT NULL, DEFAULT 'N' | 사유 확인 전화 | 'Y', 'N' |
| report | char | NOT NULL, DEFAULT 'N' | 리포트 여부 | 'Y', 'N' |
| user_fee | int | NOT NULL, DEFAULT 0 | 고객 부담금 | |
| company_fee | int | NOT NULL, DEFAULT 0 | 회사 부담금 | |
| regist_date | date | NOT NULL | 등록일 | |
| start_date | date | NULL | 수리 시작일 | |
| end_date | date | NULL | 수리 완료일 | |
| result | varchar(30) | NULL, DEFAULT '' | 결과 | |
| description | varchar(250) | NOT NULL, DEFAULT '' | 비고 | |
| report_seq | int | NULL | 리포트 번호 | |
| as_muid | varchar(30) | NOT NULL, DEFAULT '' | 담당자 UID | |
| return_flag | char | NOT NULL, DEFAULT 'N' | 반품 여부 | 'Y', 'N' |
| return_seq | int | NULL | 반품 번호 | |
| as_export_flag | char | NULL, DEFAULT 'N' | A/S 출고 여부 | 'Y', 'N' |
| as_export_seq | int | NULL | A/S 출고 번호 | |
| director | varchar(30) | NULL | 담당자 | |

### 관계
- `category_code`, `brand_code`, `goods_code`, `option_code`로 erp_brand, erp_goods, erp_option JOIN
- `user_phone`으로 `crm_call_memo.sender`와 교차 조회 (하이픈 제거 필요)

### 비즈니스 맥락
- CRM 시스템 도입 이전의 레거시 A/S 관리 테이블
- crm_as와 달리 제품 코드를 분해된 형태(category/brand/goods/option)로 저장
- 전화 상담 시 고객의 과거 ERP A/S 이력 조회에 활용
- 상태 판별: start_date/end_date의 NULL 여부

### 자주 쓰는 쿼리 패턴
```sql
-- 고객 전화번호로 ERP A/S 이력 조회 (전화 상담 시)
SELECT as_seq, start_date, reason
FROM erp_as
WHERE REPLACE(user_phone, '-', '') = ?
  AND reason != ''
ORDER BY as_seq DESC;
```

---

## erp_blacklist (블랙리스트)

summary: 출고 제한 대상 고객(블랙리스트)을 관리하는 테이블. 고객명, 전화번호, 주소, 사유를 기록하며, 출고 시 경고를 표시할지 여부를 설정한다.
domain: crm
related_tables: erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| user_num | int | PK, AUTO_INCREMENT | 번호 | |
| user_name | varchar(20) | NULL | 고객명 | |
| user_postal_code | varchar(255) | NULL | 우편번호 | |
| user_address | text | NULL | 주소 | |
| user_phone | varchar(30) | NULL | 전화번호 | |
| reason_text | text | NULL | 사유 | |
| export_alert_flag | char | NULL | 출고 경고 플래그 | 'Y', 'N' |

### 관계
- 출고 프로세스에서 고객 정보(이름, 전화번호, 주소)와 대조

### 비즈니스 맥락
- 악성 고객, 상습 반품자 등을 등록하여 출고 시 경고
- export_alert_flag='Y'이면 출고 화면에서 경고 메시지 표시
- 전체 목록을 조회하여 출고 전 체크

### 자주 쓰는 쿼리 패턴
```sql
-- 블랙리스트 전체 목록 조회
SELECT * FROM erp_blacklist;

-- 블랙리스트 등록
INSERT INTO erp_blacklist
SET user_name = ?, user_postal_code = ?, user_address = ?,
    user_phone = ?, reason_text = ?, export_alert_flag = ?;
```

---

## erp_smartStore (스마트스토어 주문 상태)

summary: 네이버 스마트스토어 주문의 상태 변경 이력을 관리하는 테이블. 주문ID, 상품주문ID, 결제일, 주문 상태, 클레임 유형/상태 등을 기록한다.
domain: crm
related_tables: naver_smartstore_orders

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint | PK, AUTO_INCREMENT | 번호 | |
| orderId | varchar(30) | NOT NULL | 주문 ID | |
| productOrderId | varchar(30) | NOT NULL | 상품주문 ID | |
| paymentDate | datetime | NULL | 결제일시 | |
| productOrderStatus | varchar(50) | NOT NULL | 상품주문 상태 | |
| lastChangedDate | datetime | NOT NULL | 마지막 변경일시 | |
| receiverAddressChanged | tinyint(1) | NOT NULL | 수령인 주소 변경 여부 | 0, 1 |
| lastChangedType | varchar(50) | NOT NULL | 마지막 변경 유형 | |
| giftReceivingStatus | varchar(30) | NULL | 선물 수령 상태 | |
| claimType | varchar(30) | NULL | 클레임 유형 | 'CANCEL', 'RETURN', 'EXCHANGE' |
| claimStatus | varchar(50) | NULL | 클레임 상태 | |

### 관계
- `orderId`, `productOrderId`로 네이버 스마트스토어 API와 연동

### 비즈니스 맥락
- 네이버 스마트스토어 API에서 주문 변경 이벤트를 수신하여 저장
- claimType: CANCEL(취소), RETURN(반품), EXCHANGE(교환)
- 주문 상태 변경을 실시간으로 추적하여 CS 대응

### 자주 쓰는 쿼리 패턴
```sql
-- 클레임 유형별 현황 조회
SELECT claimType, COUNT(*) AS cnt
FROM erp_smartStore
WHERE claimType IS NOT NULL
GROUP BY claimType;

-- 특정 주문의 상태 변경 이력
SELECT * FROM erp_smartStore WHERE orderId = ? ORDER BY lastChangedDate DESC;
```
