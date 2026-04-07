# 재무(Finance) / 이벤트(Event) / 사은품(Free Gift) / 쿠폰(Coupon) 도메인 테이블 정의

---

## finance_accounts (재무 게시판/장부)

summary: 재무 관련 게시글 및 장부 데이터를 관리하는 테이블. 작성자가 제목, 내용, 첨부파일과 함께 재무 관련 기록을 등록한다.
domain: finance
related_tables: member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| member_id | varchar(30) | NULL | 작성자 (member.id 참조) | |
| title | varchar(100) | NOT NULL, DEFAULT '' | 제목 | |
| content | mediumtext | NOT NULL | 내용 | |
| upload_file | text | NOT NULL | 업로드 파일 경로 | |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| modified_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | |

### 관계
- member 테이블과 member_id로 JOIN하여 작성자명(member.name) 조회

### 비즈니스 맥락
- 재무팀이 장부, 회계 관련 문서를 등록/관리하는 게시판 형태의 테이블
- 파일 첨부 기능 지원

### 자주 쓰는 쿼리 패턴
```sql
-- 장부 목록 조회 (작성자명 포함)
SELECT a.*, b.name AS member_name
FROM finance_accounts AS a
LEFT JOIN member AS b ON a.member_id = b.id;
```

---

## finance_estimate (견적서)

summary: 견적서(Estimate) 정보를 관리하는 테이블. 발행일자, 수신자 정보, 타입(카테고리), 출고요청일자, 세금계산서 연동 번호 등을 저장한다.
domain: finance
related_tables: finance_estimate_item, finance_tax_invoice, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| muid | varchar(30) | NOT NULL | 작성자 UID | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 발행일자 | |
| title | varchar(100) | NULL | 제목 | |
| chg_company_name | varchar(100) | NULL | 상호명 | |
| chg_phone | varchar(100) | NULL | 핸드폰번호 | |
| chg_email | varchar(100) | NULL | 이메일 | |
| chg_address | varchar(255) | NULL | 주소 | |
| chg_postal_code | varchar(20) | NULL | 우편번호 | |
| chg_description | text | NULL | 비고 | |
| delivery_message | varchar(255) | NULL | 배송메세지 | |
| type | char | DEFAULT 'G' | 타입(카테고리) | 'G' |
| export_schedule_date | datetime | NULL | 출고요청일자 | |
| taxInvoiceSeq | varchar(10) | NULL | 세금계산서 번호 | |

### 관계
- finance_estimate_item: est_seq로 1:N 관계 (견적서 항목 목록)
- finance_tax_invoice: taxInvoiceSeq로 세금계산서와 연동
- member: muid로 작성자 조회

### 비즈니스 맥락
- 고객 또는 거래처에 보내는 견적서를 관리
- type 컬럼으로 카테고리 분류 (예: 'G' = 일반)
- 견적서 발행 후 세금계산서로 연동 가능
- 인덱스: (type, regist_date, seq) 복합 인덱스

### 자주 쓰는 쿼리 패턴
```sql
-- 견적서 목록 조회 (타입별 필터)
SELECT * FROM finance_estimate
WHERE type = #{type} AND regist_date >= #{startDate} AND regist_date <= #{endDate}
ORDER BY seq DESC;
```

---

## finance_estimate_item (견적서 항목)

summary: 견적서에 포함되는 개별 상품 항목을 관리하는 테이블. 카테고리, 브랜드, 상품, 옵션 코드와 수량, 출고가를 저장한다.
domain: finance
related_tables: finance_estimate, erp_goods, erp_option

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| est_seq | int | NOT NULL | 견적서번호 (finance_estimate.seq FK) | |
| category_code | char | NOT NULL, DEFAULT '' | 카테고리코드 | |
| brand_code | varchar(7) | NOT NULL | 브랜드코드 | |
| goods_code | varchar(6) | NOT NULL | 상품코드 | |
| option_code | varchar(7) | NOT NULL | 옵션코드 | |
| ea | varchar(20) | NOT NULL | 상품개수 | |
| export_price | varchar(10) | NOT NULL | 상품출고가 | |
| description | varchar(250) | NULL, DEFAULT '' | 비고 | |
| export_request_seq | int unsigned | NULL | 출고요청번호 | |

### 관계
- finance_estimate: est_seq로 N:1 관계
- erp_goods: category_code, brand_code, goods_code로 상품 정보 참조
- erp_option: option_code로 옵션 정보 참조

### 비즈니스 맥락
- 하나의 견적서(finance_estimate)에 여러 상품 항목이 포함됨
- 출고요청번호(export_request_seq)와 연동하여 실제 출고 처리 가능
- 인덱스: est_seq

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 견적서의 상품 항목 조회
SELECT * FROM finance_estimate_item WHERE est_seq = #{estSeq};
```

---

## finance_etc_remit (기타 송금 신청)

summary: 기타 비용(마케팅비, 기타결제 등)에 대한 송금 신청 및 승인 프로세스를 관리하는 테이블. 팀장 승인 -> 대표 승인 -> 송금 처리의 다단계 결재 워크플로우를 지원한다.
domain: finance
related_tables: member, department

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| finance_etc_seq | int unsigned | PK, AUTO_INCREMENT | 송금 신청 번호 | |
| muid | varchar(30) | NOT NULL, DEFAULT '' | 신청자 UID | |
| type | varchar(10) | NOT NULL, DEFAULT '' | 구분(카테고리) | '마케팅비', '기타결제' 등 |
| title | varchar(100) | NOT NULL, DEFAULT '' | 제목 | |
| content | text | NOT NULL | 내용 | |
| bank | varchar(15) | NOT NULL, DEFAULT '' | 은행명 | |
| account_holder | varchar(30) | NOT NULL, DEFAULT '' | 예금주 | |
| account_number | varchar(30) | NOT NULL, DEFAULT '' | 계좌번호 | |
| amount | int | NOT NULL | 금액 | |
| upload_file | text | NULL | 첨부파일 | |
| description | varchar(250) | NOT NULL, DEFAULT '' | 비고 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| approval_flag | char | NULL | 팀장 승인 여부 | 'Y' / 'N' / NULL(미처리) |
| approval_muid | varchar(30) | NULL | 팀장 승인자 UID | |
| approval_date | datetime | NULL | 팀장 승인일시 | |
| approval_flag_owner | char | NULL | 대표 승인 여부 | 'Y' / 'N' / NULL |
| approval_muid_owner | varchar(30) | NULL | 대표 UID | |
| approval_date_owner | datetime | NULL | 대표 승인일 | |
| reject_reason | varchar(100) | NULL | 팀장 반려 사유 | |
| reject_reason_owner | varchar(100) | NULL | 대표 반려 사유 | |
| remit_muid | varchar(30) | NULL | 송금 처리자 UID | |
| remit_date | datetime | NULL | 송금 처리일시 | |
| cancel_flag | char | NOT NULL, DEFAULT 'N' | 취소 여부 | 'Y' / 'N' |
| cancel_muid | varchar(30) | NULL | 취소 처리자 UID | |
| cash_receipt_cancel_confirm | char | NOT NULL, DEFAULT 'N' | 현금영수증 취소 확인 여부 | 'Y' / 'N' |
| approval_depth | int | DEFAULT 1 | 승인 단계 | 1(팀장만), 2(팀장+대표) |
| day_flag | char | DEFAULT 'N' | 당일 처리 플래그 | 'Y' / 'N' |
| day_description | varchar(250) | NULL | 당일 처리 사유 | |
| remit_flag | char(2) | NULL | 송금 승인 상태 | NULL(미승인), 'Y'(승인), 'N'(반려) |

### 관계
- member: muid, approval_muid, approval_muid_owner, remit_muid, cancel_muid로 각각 신청자/승인자/송금자/취소자 조회
- department: member를 통해 부서명 조회

### 비즈니스 맥락
- 다단계 결재 워크플로우: 신청 -> 팀장 승인(approval_flag) -> 대표 승인(approval_flag_owner) -> 송금 처리(remit_flag)
- approval_depth=1이면 팀장 승인만, approval_depth=2이면 팀장+대표 2단계 승인
- cancel_flag='Y'이면 취소된 건
- type으로 '마케팅비', '기타결제' 등 카테고리 분류
- 6개월 이내 동일 계좌번호 사용 이력 확인 기능 존재
- 알리고 문자 서비스 충전 등 반복 결제 체크 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 기간별 기타송금 목록 조회 (부서명, 작성자명 포함)
SELECT d.department_name, m.name, f.*
FROM finance_etc_remit AS f
INNER JOIN member AS m ON m.uid = f.muid
INNER JOIN department AS d ON m.department_id = d.id
WHERE f.regist_date >= #{startDate} AND f.regist_date <= #{endDate}
ORDER BY f.finance_etc_seq DESC;

-- 자주 사용하는 계좌 목록
SELECT COUNT(*) AS count, bank, account_holder, account_number, MAX(regist_date) AS regist_date
FROM finance_etc_remit
GROUP BY bank, account_number
ORDER BY count DESC;

-- 동일 계좌 최근 6개월 사용 횟수 확인
SELECT COUNT(*) FROM finance_etc_remit
WHERE REPLACE(REPLACE(account_number, '-', ''), ' ', '') = #{accountNumber}
AND regist_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH);
```

---

## finance_fixtures_remit (비품 구매 신청)

summary: 비품(사무용품, 장비 등) 구매 신청 및 승인을 관리하는 테이블. 승인 -> 구매 처리 워크플로우를 지원하며, 달러 결제 여부도 관리한다.
domain: finance
related_tables: member, department

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| finance_fixtures_seq | int unsigned | PK, AUTO_INCREMENT | 비품 신청 번호 | |
| muid | varchar(30) | NOT NULL, DEFAULT '' | 신청자 UID | |
| type | varchar(10) | NOT NULL, DEFAULT '' | 구분 | |
| title | varchar(100) | NOT NULL, DEFAULT '' | 제목 | |
| price | varchar(100) | NOT NULL | 가격 | |
| link | text | NOT NULL | 구매 링크 | |
| description | varchar(900) | NOT NULL, DEFAULT '' | 상세 설명 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| approval_flag | char | NULL | 승인 여부 | 'Y' / 'N' / NULL |
| approval_muid | varchar(30) | NULL | 승인자 UID | |
| approval_date | datetime | NULL | 승인일시 | |
| purchase_flag | char | NULL | 구매 처리 여부 | 'Y' / 'N' / NULL |
| purchase_muid | varchar(30) | NULL | 구매 처리자 UID | |
| purchase_date | datetime | NULL | 구매 처리일시 | |
| purpose | varchar(100) | NOT NULL, DEFAULT '' | 구매 사유 | |
| purchase_option | varchar(900) | NULL | 구매 옵션 상세 | |
| is_usd_yn | varchar(1) | DEFAULT 'N' | 달러 결제 여부 | 'Y' / 'N' |

### 관계
- member: muid, approval_muid, purchase_muid로 신청자/승인자/구매자 조회
- department: member를 통해 부서명 조회

### 비즈니스 맥락
- 비품 구매 워크플로우: 신청 -> 승인(approval_flag) -> 구매 처리(purchase_flag)
- 구매 링크와 가격을 함께 등록하여 효율적으로 관리
- 달러 결제 여부(is_usd_yn)로 해외 구매 건 구분
- 기존 비품 신청 건 복제(clone) 기능 지원

### 자주 쓰는 쿼리 패턴
```sql
-- 기간별 비품 구매 신청 목록 (부서, 신청자, 승인자, 구매자 정보 포함)
SELECT a.*,
  (SELECT d.department_name FROM member AS m LEFT JOIN department AS d ON m.department_id = d.id WHERE m.uid = a.muid) AS department_name,
  (SELECT name FROM member WHERE uid = a.muid) AS name,
  (SELECT name FROM member WHERE uid = a.purchase_muid) AS purchase_name,
  (SELECT name FROM member WHERE uid = a.approval_muid) AS approval_name
FROM finance_fixtures_remit AS a
WHERE regist_date >= #{startDate} AND regist_date <= #{endDate};
```

---

## finance_tax_invoice (세금계산서)

summary: 전자세금계산서 발행 정보를 관리하는 테이블. 공급자(invoicer)와 공급받는자(invoicee) 사업자 정보, 발행 상태, 국세청 승인번호 등을 저장하며 POPBiLL 연동으로 전자 발행한다.
domain: finance
related_tables: finance_tax_invoice_detail, finance_tax_invoice_biz, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| taxInvoiceSeq | int unsigned | PK, AUTO_INCREMENT | 세금계산서 번호 | |
| categoryCode | varchar(5) | NULL | 카테고리코드 | |
| issueType | varchar(7) | DEFAULT 'fixed' | 발행 유형 | 'fixed'(정기), 'order'(주문) 등 |
| otherSystemIssued | char | DEFAULT 'N' | 타시스템 발행 여부 | 'Y' / 'N' |
| recvSeq | varchar(250) | NULL | 수주 번호 (쉼표 구분 복수 가능) | |
| memberUid | varchar(30) | NOT NULL, DEFAULT '' | 작성자 UID | |
| kwon | int | NOT NULL, DEFAULT 0 | 권 | |
| ho | int | NOT NULL, DEFAULT 0 | 호 | |
| invoicerCorpNum | varchar(30) | NOT NULL, DEFAULT '' | 공급자 사업자번호 | |
| invoicerTaxRegID | varchar(30) | NOT NULL, DEFAULT '' | 공급자 종사업장번호 | |
| invoicerCorpName | varchar(30) | NOT NULL, DEFAULT '' | 공급자 상호 | |
| invoicerCEOName | varchar(30) | NOT NULL, DEFAULT '' | 공급자 대표자명 | |
| invoicerAddr | varchar(150) | NOT NULL, DEFAULT '' | 공급자 주소 | |
| invoicerBizType | varchar(100) | NOT NULL, DEFAULT '' | 공급자 업태 | |
| invoicerBizClass | varchar(100) | NOT NULL, DEFAULT '' | 공급자 종목 | |
| invoicerContactName | varchar(30) | NOT NULL, DEFAULT '' | 공급자 담당자명 | |
| invoicerTEL | varchar(30) | NOT NULL, DEFAULT '' | 공급자 전화번호 | |
| invoicerEmail | varchar(50) | NOT NULL, DEFAULT '' | 공급자 이메일 | |
| invoiceeCorpNum | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 사업자번호 | |
| invoiceeTaxRegID | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 종사업장번호 | |
| invoiceeCorpName | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 상호 | |
| invoiceeCEOName | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 대표자명 | |
| invoiceeAddr | varchar(150) | NOT NULL, DEFAULT '' | 공급받는자 주소 | |
| invoiceeBizType | varchar(100) | NOT NULL, DEFAULT '' | 공급받는자 업태 | |
| invoiceeBizClass | varchar(100) | NOT NULL, DEFAULT '' | 공급받는자 종목 | |
| invoiceeContactName1 | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 담당자1 이름 | |
| invoiceeTEL1 | varchar(30) | NOT NULL, DEFAULT '' | 공급받는자 담당자1 전화번호 | |
| invoiceeEmail1 | varchar(50) | NOT NULL, DEFAULT '' | 공급받는자 담당자1 이메일 | |
| invoiceeContactName2 | varchar(30) | NULL | 공급받는자 담당자2 이름 | |
| invoiceeTEL2 | varchar(30) | NULL | 공급받는자 담당자2 전화번호 | |
| invoiceeEmail2 | varchar(50) | NULL | 공급받는자 담당자2 이메일 | |
| cash | varchar(50) | NOT NULL, DEFAULT '' | 현금 | |
| chkBill | varchar(50) | NOT NULL, DEFAULT '' | 수표 | |
| note | varchar(10) | NOT NULL, DEFAULT '' | 어음 | |
| credit | varchar(50) | NOT NULL, DEFAULT '' | 외상미수금 | |
| remark1 | varchar(1000) | NOT NULL, DEFAULT '' | 비고 | |
| purposeType | varchar(10) | NOT NULL, DEFAULT '영수' | 영수/청구 구분 | '영수', '청구' |
| writeDate | varchar(10) | NOT NULL, DEFAULT '' | 작성일자 | 'YYYYMMDD' |
| registDate | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| issueDate | datetime | NULL | 발행일시 | |
| ntsConfirmNum | varchar(50) | NULL | 국세청 승인번호 | |
| cancelIssueDate | datetime | NULL | 발행취소일시 | |

### 관계
- finance_tax_invoice_detail: taxInvoiceSeq로 1:N 관계 (세금계산서 품목 상세)
- finance_tax_invoice_biz: invoiceeCorpNum으로 거래처 정보 참조
- member: memberUid로 작성자 조회
- recvSeq로 B2B 수주(recv) 연동

### 비즈니스 맥락
- POPBiLL API와 연동하여 전자세금계산서 발행
- 발행 상태: issueDate NOT NULL이면 발행완료, cancelIssueDate NOT NULL이면 발행취소, 둘 다 NULL이면 미발행
- recvSeq로 수주 건과 연동 가능 (B2BCRM 도메인)
- 여러 세금계산서를 동일 거래처 기준으로 합산 발행 가능 (combine 기능)
- issueType으로 정기/주문별 발행 구분

### 자주 쓰는 쿼리 패턴
```sql
-- 세금계산서 목록 조회 (합계 포함)
SELECT c.name AS memberName,
  CASE WHEN a.issueDate IS NOT NULL THEN '발행완료'
       WHEN a.cancelIssueDate IS NOT NULL THEN '발행취소'
       ELSE '미발행' END AS ext2,
  a.*,
  IFNULL(CONCAT(SUM(b.supplyCost)), '0') AS supplyCostTotal,
  IFNULL(CONCAT(SUM(b.tax)), '0') AS taxTotal,
  IFNULL(CONCAT(SUM(b.supplyCost) + SUM(b.tax)), '0') AS totalAmount
FROM finance_tax_invoice AS a
LEFT JOIN finance_tax_invoice_detail AS b ON a.taxInvoiceSeq = b.taxInvoiceSeq
INNER JOIN member AS c ON a.memberUid = c.uid
GROUP BY a.taxInvoiceSeq;

-- 수주 번호로 세금계산서 조회
SELECT * FROM finance_tax_invoice WHERE recvSeq = #{recvSeq};
```

---

## finance_tax_invoice_biz (세금계산서 거래처)

summary: 세금계산서 발행 시 자주 사용하는 공급받는자(거래처) 사업자 정보를 별도로 관리하는 테이블. 거래처 즐겨찾기 역할을 한다.
domain: finance
related_tables: finance_tax_invoice

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| invoiceeCorpNum | varchar(30) | NOT NULL, DEFAULT '' | 사업자번호 | |
| invoiceeTaxRegID | varchar(50) | NOT NULL, DEFAULT '' | 종사업장번호 | |
| invoiceeCorpName | varchar(50) | NOT NULL, DEFAULT '' | 상호명 | |
| invoiceeCEOName | varchar(30) | NOT NULL, DEFAULT '' | 대표자명 | |
| invoiceeAddr | varchar(150) | NOT NULL, DEFAULT '' | 주소 | |
| invoiceeBizType | varchar(100) | NOT NULL, DEFAULT '' | 업태 | |
| invoiceeBizClass | varchar(150) | NOT NULL, DEFAULT '' | 종목 | |
| invoiceeContactName1 | varchar(30) | NOT NULL, DEFAULT '' | 담당자명 | |
| invoiceeTEL1 | varchar(50) | NOT NULL, DEFAULT '' | 전화번호 | |
| invoiceeEmail1 | varchar(100) | NOT NULL, DEFAULT '' | 이메일 | |

### 관계
- finance_tax_invoice: invoiceeCorpNum으로 세금계산서 발행 시 거래처 정보 참조

### 비즈니스 맥락
- 세금계산서 발행 시 거래처 정보를 빠르게 불러오기 위한 즐겨찾기 테이블
- finance_tax_invoice에 저장된 기존 거래처와 UNION하여 통합 거래처 목록을 제공

### 자주 쓰는 쿼리 패턴
```sql
-- 거래처 통합 목록 (기존 세금계산서 거래처 + 등록 거래처)
SELECT * FROM (
  SELECT invoiceeCorpNum, invoiceeCorpName, invoiceeEmail1, COUNT(1) AS count
  FROM finance_tax_invoice GROUP BY invoiceeCorpNum, invoiceeEmail1
  UNION ALL
  SELECT REPLACE(invoiceeCorpNum, '-', ''), invoiceeCorpName, invoiceeEmail1, 0 AS count
  FROM finance_tax_invoice_biz GROUP BY invoiceeCorpNum, invoiceeEmail1
  ORDER BY count DESC
) AS a GROUP BY invoiceeCorpNum, invoiceeEmail1;
```

---

## finance_tax_invoice_detail (세금계산서 품목 상세)

summary: 세금계산서에 포함되는 개별 품목 상세 정보를 관리하는 테이블. 품목명, 규격, 수량, 단가, 공급가액, 세액 등을 저장한다.
domain: finance
related_tables: finance_tax_invoice

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int | PK, AUTO_INCREMENT | 고유 ID | |
| taxInvoiceSeq | int | NOT NULL | 세금계산서 번호 (FK) | |
| taxInvoiceDetailSeq | int | NOT NULL | 품목 순번 | |
| recvSeq | varchar(100) | NULL | 수주 번호 | |
| purchaseDT | varchar(50) | DEFAULT '' | 거래일자 | |
| itemName | varchar(255) | NOT NULL, DEFAULT '' | 품목명 | |
| spec | varchar(50) | NOT NULL, DEFAULT '' | 규격 | |
| qty | varchar(10) | NOT NULL, DEFAULT '' | 수량 | |
| unitCost | varchar(10) | NOT NULL, DEFAULT '' | 단가 | |
| supplyCost | varchar(50) | NOT NULL, DEFAULT '' | 공급가액 | |
| tax | varchar(10) | NOT NULL, DEFAULT '' | 세액 | |
| remark | varchar(150) | NOT NULL, DEFAULT '' | 비고 | |

### 관계
- finance_tax_invoice: taxInvoiceSeq로 N:1 관계
- UNIQUE 제약조건: (taxInvoiceSeq, taxInvoiceDetailSeq) 조합 유일

### 비즈니스 맥락
- 하나의 세금계산서에 여러 품목이 포함됨
- supplyCost(공급가액)과 tax(세액)를 합산하여 총 금액 계산
- 수주 건(recvSeq)과 연동하여 주문 기반 세금계산서 자동 생성 가능
- 세금계산서 합산 발행 시 여러 세금계산서의 품목을 하나로 통합

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 세금계산서 품목 조회
SELECT *, taxInvoiceDetailSeq AS serialNum
FROM finance_tax_invoice_detail
WHERE taxInvoiceSeq = #{taxInvoiceSeq}
ORDER BY taxInvoiceDetailSeq;

-- 세금계산서 합계 조회
SELECT IFNULL(SUM(supplyCost), 0) AS supplyCostTotal,
       IFNULL(SUM(tax), 0) AS taxTotal,
       IFNULL(SUM(supplyCost) + SUM(tax), 0) AS totalAmount
FROM finance_tax_invoice_detail
WHERE taxInvoiceSeq = #{taxInvoiceSeq};
```

---

## event_best_review (베스트리뷰 이벤트)

summary: 네이버 스마트스토어 베스트리뷰 이벤트를 관리하는 테이블. 이벤트 채널명, 제목, 시작/종료일을 저장한다.
domain: event
related_tables: event_best_review_winner, event_best_review_prize

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | index | |
| channel_name | varchar(100) | NULL | 이벤트 채널명 | |
| title | varchar(255) | NULL | 이벤트 제목 | |
| start_at | datetime | NULL | 시작일시 | |
| end_at | datetime | NULL | 종료일시 | |

### 관계
- event_best_review_winner: event_seq로 1:N 관계 (당첨자 목록)
- event_best_review_prize: 사은품 목록 (독립 테이블, 당첨자가 prize_seq로 선택)

### 비즈니스 맥락
- 네이버 스마트스토어에서 베스트 리뷰를 작성한 고객에게 사은품을 지급하는 이벤트
- 이벤트 기간(start_at ~ end_at) 내 작성된 리뷰 중 선정

### 자주 쓰는 쿼리 패턴
```sql
-- 베스트리뷰 이벤트 목록 조회
SELECT * FROM event_best_review ORDER BY seq DESC;
```

---

## event_best_review_prize (베스트리뷰 사은품 목록)

summary: 베스트리뷰 이벤트에서 당첨자가 선택할 수 있는 사은품 목록을 관리하는 테이블.
domain: event
related_tables: event_best_review_winner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | index | |
| product_code | varchar(30) | NULL | 사은품 제품코드 | |
| prize_name | varchar(255) | NULL | 노출용 사은품명 | |
| is_active | char | NOT NULL, DEFAULT 'N' | 노출 여부 | 'Y' / 'N' |

### 관계
- event_best_review_winner: prize_seq FK로 당첨자가 선택한 사은품 참조

### 비즈니스 맥락
- is_active='Y'인 사은품만 당첨자 선택 화면에 노출
- 제품코드를 통해 ERP 상품 정보와 연동 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 활성 사은품 목록
SELECT * FROM event_best_review_prize WHERE is_active = 'Y';
```

---

## event_best_review_winner (베스트리뷰 당첨자)

summary: 스마트스토어 베스트리뷰 이벤트의 당첨자 정보를 관리하는 테이블. 리뷰 정보, 당첨자/수령자 정보, 사은품 선택, 승인 상태, 출고 연동까지 포함한다.
domain: event
related_tables: event_best_review, event_best_review_prize, member

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | index | |
| event_seq | int unsigned | NULL, FK | 베스트리뷰 이벤트 번호 | |
| review_no | varchar(30) | NULL | 네이버 스마트스토어 리뷰글번호 | |
| order_no | varchar(30) | NULL | 네이버 스마트스토어 주문번호 | |
| product_name | varchar(500) | NULL | 네이버 스마트스토어 상품명 | |
| rating | tinyint unsigned | NULL | 고객 평점 | 1~5 |
| winner_name | varchar(30) | NULL | 당첨자명 | |
| winner_phone | varchar(30) | NULL | 당첨자 연락처 | |
| recipient_name | varchar(30) | NULL | 수령자명 | |
| recipient_phone | varchar(30) | NULL | 수령자 연락처 | |
| recipient_postal_code | varchar(20) | NULL | 수령지 우편번호 | |
| recipient_address | varchar(500) | NULL | 수령지 주소 | |
| delivery_message | text | NULL | 배송 요청사항 | |
| recipient_input_at | datetime | NULL | 수령 정보 입력일시 | |
| prize_seq | int unsigned | NULL, FK | 고객 선택 경품 (event_best_review_prize.seq) | |
| engrave_text | varchar(255) | NULL | 각인 문구 | |
| is_approved | char | NOT NULL, DEFAULT 'N' | 승인 여부 | 'Y' / 'N' |
| approved_at | datetime | NULL | 승인일시 | |
| approved_by | int unsigned | NULL, FK | 승인자 ID (member.id) | |
| noti_winner_sent_at | datetime | NULL | 당첨 안내 발송일시 | |
| noti_ship_sent_at | datetime | NULL | 사은품 발송 안내 발송일시 | |
| export_request_id | int | NULL | 출고요청 ID | |
| description | text | NULL | 비고 | |
| created_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| updated_at | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE | 마지막 수정일시 | |

### 관계
- event_best_review: event_seq FK (ON DELETE SET NULL)
- event_best_review_prize: prize_seq FK (ON DELETE SET NULL)
- member: approved_by FK (ON DELETE SET NULL)
- UNIQUE 제약조건: (event_seq, winner_name, winner_phone)

### 비즈니스 맥락
- 당첨자 관리 프로세스: 당첨 등록 -> 당첨 안내 발송(noti_winner_sent_at) -> 수령 정보 입력(recipient_input_at) -> 사은품 선택(prize_seq) -> 승인(is_approved) -> 출고(export_request_id) -> 발송 안내(noti_ship_sent_at)
- 각인 문구(engrave_text) 지원: 커스텀 사은품 제공 가능
- 출고요청(export_request_id)으로 ERP 출고 프로세스와 연동

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 이벤트 당첨자 목록
SELECT * FROM event_best_review_winner WHERE event_seq = #{eventSeq};
```

---

## event_giveaway (기브어웨이 이벤트)

summary: 기브어웨이(경품 추첨) 이벤트를 관리하는 테이블. 공홈/인스타그램 등 플랫폼별로 이벤트를 생성하고 참여 페이지, 리워드, 당첨 정보를 관리한다.
domain: event
related_tables: event_giveaway_applicant, event_giveaway_image, event_giveaway_mission, event_message

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| channel | varchar(30) | NULL | 채널 | |
| id | varchar(255) | NOT NULL, UNIQUE | 고유 ID | |
| round_no | int | NOT NULL, DEFAULT 0 | 회차 | |
| platform | varchar(50) | NOT NULL, DEFAULT 'homepage' | 이벤트 플랫폼 | 'homepage', 'instagram' 등 |
| title | varchar(255) | NOT NULL | 제목 | |
| event_page_link | text | NULL | 이벤트 참여 페이지 링크 | |
| reward_link | text | NULL | 리워드 상품 상세 링크 | |
| reward_text | text | NULL | 리워드 상품 상세정보 내용 | |
| result_page_link | text | NULL | 결과 페이지 링크 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| banner_image_link | text | NULL | 대표 배너 이미지 링크 | |
| start_date | datetime | NULL | 시작일시 | |
| end_date | datetime | NULL | 종료일시 | |
| reward_date | datetime | NULL | 당첨자 발표일시 | |
| winner_num | int | NOT NULL, DEFAULT 0 | 당첨자 수 | |
| success_text | text | NULL | 신청 성공시 출력 메세지(HTML) | |

### 관계
- event_giveaway_applicant: giveaway_seq FK로 1:N (신청자 목록)
- event_giveaway_image: giveaway_seq FK로 1:N (이벤트 이미지)
- event_giveaway_mission: giveaway_seq FK로 1:N (미션 목록)
- event_message: event_id로 이벤트 메시지 연동

### 비즈니스 맥락
- 공홈, 인스타그램 등 다양한 플랫폼에서 진행하는 경품 추첨 이벤트
- 회차(round_no)로 동일 이벤트의 반복 진행 관리
- 당첨자 수(winner_num)를 사전 설정하고 참여자 중 추첨
- id 컬럼이 UNIQUE로 이벤트 고유 식별자 역할

### 자주 쓰는 쿼리 패턴
```sql
-- 기브어웨이 목록 (참여자 수 포함)
SELECT giveaway.seq AS giveaway_seq, giveaway.platform, giveaway.title,
  giveaway.start_date, giveaway.end_date, giveaway.reward_date,
  COUNT(applicant.seq) AS applicant_count
FROM event_giveaway AS giveaway
LEFT JOIN event_giveaway_applicant AS applicant ON giveaway.seq = applicant.giveaway_seq
GROUP BY giveaway.seq
ORDER BY end_date DESC, start_date DESC;

-- 기브어웨이 상세 (신규 참여자 수 포함)
SELECT giveaway.*, COUNT(applicant.seq) AS totalCount,
  SUM(IF(applicant.is_new = 'Y', 1, 0)) AS newParticipantCount
FROM event_giveaway AS giveaway
LEFT JOIN event_giveaway_applicant AS applicant ON giveaway.seq = applicant.giveaway_seq
WHERE giveaway.seq = #{seq}
GROUP BY giveaway.seq;
```

---

## event_giveaway_applicant (기브어웨이 신청자)

summary: 기브어웨이 이벤트에 참여한 신청자 정보를 관리하는 테이블. 전화번호, 이메일, 신규 여부, 당첨 여부를 저장한다.
domain: event
related_tables: event_giveaway, erp_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| giveaway_seq | int unsigned | NOT NULL, FK | 기브어웨이 번호 | |
| phone | varchar(30) | NOT NULL | 전화번호 | |
| email | varchar(255) | NOT NULL | 이메일 주소 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| is_new | char | NOT NULL, DEFAULT 'N' | 신규 등록 여부 | 'Y' / 'N' |
| prize_flag | char | NOT NULL, DEFAULT 'N' | 당첨 여부 | 'Y' / 'N' |

### 관계
- event_giveaway: giveaway_seq FK (ON DELETE CASCADE)
- UNIQUE 제약조건: (giveaway_seq, email), (giveaway_seq, phone) - 이벤트별 이메일/전화번호 중복 참여 방지
- 인덱스: email, phone, regist_date, (phone, email, regist_date) 복합

### 비즈니스 맥락
- 동일 이벤트에 같은 이메일 또는 전화번호로 중복 참여 불가
- is_new='Y'이면 신규 참여자(기존에 참여 이력 없는 사용자)
- prize_flag='Y'로 당첨자 표시, 추첨 시 일괄 업데이트
- erp_stock 테이블과 JOIN하여 참여자의 최근 6개월 구매 이력(B2C) 조회 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 기브어웨이 참여자 목록 (최근 구매 이력 포함)
SELECT applicant.*, IFNULL(export.cnt, 0) AS recentlyPurchaseAmount
FROM event_giveaway_applicant AS applicant
LEFT JOIN (
  SELECT REPLACE(user_contact, '-', '') AS phone, COUNT(*) AS cnt
  FROM erp_stock
  WHERE export_date >= DATE_SUB((SELECT end_date FROM event_giveaway WHERE seq = #{seq}), INTERVAL 6 MONTH)
  GROUP BY REPLACE(user_contact, '-', '')
) AS export ON applicant.phone = export.phone
WHERE giveaway_seq = #{seq};
```

---

## event_giveaway_image (기브어웨이 이미지)

summary: 기브어웨이 이벤트에 사용되는 이미지를 관리하는 테이블.
domain: event
related_tables: event_giveaway

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| giveaway_seq | int unsigned | NOT NULL, FK | 기브어웨이 번호 | |
| link | text | NOT NULL | 이미지 링크 | |
| main_flag | char | NOT NULL, DEFAULT 'N' | 대표 이미지 여부 | 'Y' / 'N' |

### 관계
- event_giveaway: giveaway_seq FK (ON DELETE CASCADE)

### 비즈니스 맥락
- 기브어웨이 이벤트 페이지에 노출되는 이미지 관리
- main_flag='Y'인 이미지가 대표 이미지로 사용됨
- 이벤트 삭제 시 이미지도 CASCADE 삭제

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 기브어웨이의 이미지 목록
SELECT seq AS image_seq, giveaway_seq, link, main_flag
FROM event_giveaway_image WHERE giveaway_seq = #{seq};
```

---

## event_giveaway_mission (기브어웨이 미션)

summary: 기브어웨이 이벤트 참여를 위한 미션(과제) 목록을 관리하는 테이블. 필수/선택 미션을 정의한다.
domain: event
related_tables: event_giveaway

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| giveaway_seq | int unsigned | NOT NULL, FK | 기브어웨이 번호 | |
| title | varchar(255) | NOT NULL | 미션명 | |
| icon_link | text | NULL | 아이콘 링크 | |
| mission_link | text | NULL | 미션 수행처 링크 | |
| required_flag | char | NOT NULL, DEFAULT 'Y' | 필수 여부 | 'Y'(필수) / 'N'(선택) |

### 관계
- event_giveaway: giveaway_seq FK (ON DELETE CASCADE)

### 비즈니스 맥락
- 기브어웨이 참여 조건으로 미션 수행 요구 가능 (예: 인스타 팔로우, 댓글 작성 등)
- required_flag='Y'이면 반드시 수행해야 하는 필수 미션

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 기브어웨이의 미션 목록
SELECT seq AS mission_seq, giveaway_seq, title, icon_link, mission_link, required_flag
FROM event_giveaway_mission WHERE giveaway_seq = #{seq};
```

---

## event_message (이벤트 메세지 템플릿 릴레이션)

summary: 기브어웨이/프리런칭 이벤트에서 사용하는 메시지(SMS/이메일)를 관리하는 테이블. 이벤트별, 메시지 유형별로 발송할 내용을 정의한다.
domain: event
related_tables: event_giveaway, event_prelaunching, message_template

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| event_id | varchar(100) | NOT NULL | 이벤트 고유 ID | |
| template_seq | int unsigned | NULL, FK | 메세지 템플릿 번호 (message_template.seq) | |
| event_type | varchar(100) | NOT NULL | 이벤트 종류 | 'giveaway', 'prelaunching' |
| send_channel | varchar(30) | NOT NULL | 전송 채널 | 'sms', 'email' |
| message_type | varchar(30) | NOT NULL | 메시지 종류 | 'open', 'close', 'success', 'prize' |
| title | varchar(255) | NULL | 제목 | |
| content | text | NOT NULL | 내용 | |
| reference | text | NULL | 참조 JSON(개발용) | |

### 관계
- message_template: template_seq FK (ON DELETE CASCADE)
- event_giveaway / event_prelaunching: event_id로 논리적 연결

### 비즈니스 맥락
- 이벤트 라이프사이클에 따라 다양한 메시지 유형 정의:
  - open: 이벤트 오픈 안내
  - close: 이벤트 종료 안내
  - success: 참여 완료 확인
  - prize: 당첨 안내
- 각 메시지는 SMS 또는 이메일 채널로 발송 가능
- message_template과 JOIN하여 공통 템플릿 활용

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 이벤트의 메시지 목록 (템플릿 정보 포함)
SELECT message.seq AS message_seq, message.event_id, message.event_type,
  message.send_channel, message.message_type, message.title, message.content,
  template.template_name, template.brand
FROM event_message AS message
LEFT JOIN message_template AS template ON message.template_seq = template.seq
WHERE message.event_id = #{eventId};
```

---

## event_prelaunching (프리런칭 이벤트)

summary: 신제품 출시 전 프리런칭(사전 관심 등록) 이벤트를 관리하는 테이블. 브랜드별 프리런칭 페이지, 상품 연동, 예약구매 기간, 태그, 페이지 본문 등 풍부한 정보를 포함한다.
domain: event
related_tables: event_prelaunching_applicant, event_prelaunching_image, event_prelaunching_message, event_prelaunching_message_log, event_prelaunching_option, erp_goods

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| id | varchar(255) | NULL | 고유 ID (URL slug 등) | |
| title | varchar(255) | NULL | 프리런칭 제목 | |
| description | varchar(500) | NULL | 프리런칭 설명 문구 | |
| brand | varchar(30) | NOT NULL | 프리런칭 브랜드 | |
| class_names | varchar(500) | DEFAULT 'all' | 워드프레스 클래스네임 분류 | |
| product_code | varchar(30) | NULL | 대상 상품 코드(상품까지) | |
| product_link | text | NULL | 프리런칭 상품 URL | |
| subscriber_short_url | varchar(300) | NULL | 구독자 페이지 단축 URL | |
| naver_store_link | text | NULL | 브랜드스토어 상품 URL | |
| shopify_store_link | text | NULL | 쇼피파이 상품 URL | |
| start_date | datetime | NULL | 프리런칭 시작일시 | |
| end_date | datetime | NULL | 프리런칭 종료일시 | |
| preorder_start_date | datetime | NULL | 예약구매 시작일시 | |
| preorder_end_date | datetime | NULL | 예약구매 종료일시 | |
| type_label | varchar(500) | NULL | 제품 타입 라벨 (chip 텍스트) | |
| tags | json | NULL | 태그 목록 (JSON 배열) | |
| page_content | longtext | NULL | 프리런칭 페이지 본문(HTML) | |
| memo | text | NULL | 메모 | |
| post_flag | char | NOT NULL, DEFAULT 'N' | 게시 여부 | 'Y' / 'N' |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- event_prelaunching_applicant: prelaunching_seq로 1:N (참여자 목록)
- event_prelaunching_image: prelaunching_seq FK로 1:N (이미지)
- event_prelaunching_message: prelaunching_seq FK로 1:N (메시지 설정)
- event_prelaunching_message_log: prelaunching_seq로 1:N (메시지 발송 로그)
- event_prelaunching_option: prelaunching_seq FK로 1:N (옵션)
- erp_goods: product_code로 ERP 상품 연동

### 비즈니스 맥락
- 신제품 출시 전 사전 관심 등록을 받는 마케팅 이벤트
- post_flag='Y'인 프리런칭만 공홈에 노출
- 프리런칭 기간(start_date~end_date)과 예약구매 기간(preorder_start_date~preorder_end_date)을 별도로 관리
- 네이버/쇼피파이 스토어 링크로 실제 구매 페이지 연동
- 구독자 전용 페이지 별도 제공 (subscriber_short_url)
- 태그(tags)는 JSON 배열로 관리

### 자주 쓰는 쿼리 패턴
```sql
-- 프리런칭 목록 (참여자 통계 포함)
SELECT p.*, a.total_participant_count, a.new_participant_count, a.subscriber_count
FROM event_prelaunching AS p
LEFT JOIN (
  SELECT prelaunching_seq, COUNT(*) AS total_participant_count,
    SUM(IF(is_new = 'Y', 1, 0)) AS new_participant_count,
    SUM(IF(is_subscribed = 'Y', 1, 0)) AS subscriber_count
  FROM event_prelaunching_applicant GROUP BY prelaunching_seq
) AS a ON p.seq = a.prelaunching_seq
ORDER BY p.seq DESC;
```

---

## event_prelaunching_applicant (프리런칭 참여자)

summary: 프리런칭 이벤트에 참여(사전 등록)한 사용자 정보를 관리하는 테이블. 선택한 옵션, 신규 여부, 구독 여부를 저장한다.
domain: event
related_tables: event_prelaunching

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_seq | int unsigned | NOT NULL | 프리런칭 번호 (FK) | |
| phone | varchar(30) | NOT NULL | 참여자 전화번호 | |
| email | varchar(255) | NOT NULL | 참여자 이메일 | |
| option_name | varchar(255) | NULL | 선택한 옵션명 | |
| second_option_name | varchar(255) | NULL | 선택한 하위 옵션명 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |
| is_new | char | NOT NULL, DEFAULT 'N' | 신규 참여 여부 | 'Y' / 'N' |
| is_subscribed | char | NOT NULL, DEFAULT 'N' | 구독자 여부 | 'Y' / 'N' |

### 관계
- event_prelaunching: prelaunching_seq로 N:1
- 인덱스: prelaunching_seq

### 비즈니스 맥락
- 프리런칭 참여 시 관심 있는 옵션(색상, 사양 등) 선택 가능
- is_new='Y'면 이전에 참여한 적 없는 신규 사용자
- is_subscribed='Y'면 기존 구독자(뉴스레터 등)
- 참여자 수 통계로 제품 수요 예측에 활용

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 프리런칭의 참여자 목록
SELECT * FROM event_prelaunching_applicant WHERE prelaunching_seq = #{seq};

-- 프리런칭별 참여 통계
SELECT prelaunching_seq, COUNT(*) AS total,
  SUM(IF(is_new = 'Y', 1, 0)) AS new_count,
  SUM(IF(is_subscribed = 'Y', 1, 0)) AS subscriber_count
FROM event_prelaunching_applicant GROUP BY prelaunching_seq;
```

---

## event_prelaunching_image (프리런칭 이미지)

summary: 프리런칭 이벤트에 사용되는 이미지(제품 페이지용/구독자 페이지용)를 관리하는 테이블.
domain: event
related_tables: event_prelaunching

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_seq | int unsigned | NOT NULL, FK | 프리런칭 번호 | |
| page_type | varchar(30) | NULL | 페이지 구분 | 'PRODUCT'(제품), 'SUBSCRIBER'(구독자) |
| file_path | varchar(500) | NULL | S3 파일 경로 (파일명 제외) | |
| file_name | varchar(200) | NULL | 원본 파일명 | |
| is_thumbnail | char | DEFAULT 'N' | 썸네일 여부 | 'Y' / 'N' |
| sort_order | tinyint | NOT NULL, DEFAULT 0 | 정렬 순위 (0이 대표 이미지) | |
| post_flag | char | NOT NULL, DEFAULT 'Y' | 게시 여부 | 'Y' / 'N' |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- event_prelaunching: prelaunching_seq FK (ON DELETE CASCADE)

### 비즈니스 맥락
- 제품 페이지용(PRODUCT)과 구독자 페이지용(SUBSCRIBER)으로 이미지를 구분 관리
- sort_order=0인 이미지가 대표 이미지
- S3에 저장된 이미지의 경로와 파일명을 별도 관리

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 프리런칭의 이미지 목록 (정렬순)
SELECT seq, file_path, file_name, is_thumbnail, sort_order, post_flag,
  IFNULL(page_type, 'PRODUCT') AS page_type
FROM event_prelaunching_image
WHERE prelaunching_seq = #{seq}
ORDER BY sort_order ASC;
```

---

## event_prelaunching_message (프리런칭 메시지 설정)

summary: 프리런칭 이벤트의 자동 메시지(SMS/이메일) 발송 설정을 관리하는 테이블. 트리거 유형별로 메시지 내용과 발송 채널을 정의한다.
domain: event
related_tables: event_prelaunching

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_seq | int unsigned | NOT NULL, FK | 프리런칭 번호 | |
| trigger_type | varchar(30) | NOT NULL | 트리거 유형 | 'EVENT_OPEN', 'PARTICIPATE_COMPLETE', 'EVENT_CLOSE' |
| channel_type | varchar(10) | NOT NULL | 발송 채널 | 'SMS', 'EMAIL' |
| subject | varchar(200) | NULL | 제목 | |
| content | text | NULL | 본문 내용 | |
| template_code | varchar(30) | NULL | 알리고 템플릿 번호 | |
| use_flag | char | DEFAULT 'N' | 사용 여부 | 'Y' / 'N' |

### 관계
- event_prelaunching: prelaunching_seq FK (ON DELETE CASCADE)
- UNIQUE 제약조건: (prelaunching_seq, trigger_type, channel_type) - 프리런칭별 트리거+채널 조합 유일

### 비즈니스 맥락
- 프리런칭 이벤트 라이프사이클에 따른 자동 메시지 발송 설정:
  - EVENT_OPEN: 이벤트 오픈 시 참여자에게 안내
  - PARTICIPATE_COMPLETE: 참여 완료 시 확인 메시지
  - EVENT_CLOSE: 이벤트 종료 시 안내
- use_flag='Y'인 경우에만 실제 발송

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 프리런칭의 메시지 설정 조회
SELECT seq, trigger_type, channel_type, subject, content, template_code, use_flag
FROM event_prelaunching_message
WHERE prelaunching_seq = #{seq};
```

---

## event_prelaunching_message_log (프리런칭 메시지 발송 로그)

summary: 프리런칭 이벤트 메시지 발송 이력을 기록하는 로그 테이블. 발송 수신자 수, 성공/실패 상태, 오류 메시지를 저장한다.
domain: event
related_tables: event_prelaunching, event_prelaunching_message

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_seq | int | NOT NULL | 프리런칭 번호 | |
| trigger_type | varchar(30) | NOT NULL | 트리거 유형 | 'EVENT_OPEN', 'PARTICIPATE_COMPLETE', 'EVENT_CLOSE' |
| channel_type | varchar(20) | NOT NULL | 발송 채널 | 'SMS', 'EMAIL' |
| recipient_count | int | DEFAULT 0 | 수신자 수 | |
| status | varchar(20) | DEFAULT 'SUCCESS' | 발송 상태 | 'SUCCESS', 'FAIL' 등 |
| error_message | text | NULL | 오류 메시지 | |
| sent_at | datetime | DEFAULT CURRENT_TIMESTAMP | 발송일시 | |

### 관계
- event_prelaunching: prelaunching_seq로 N:1
- UNIQUE 제약조건: (prelaunching_seq, trigger_type, channel_type) - 동일 조건 중복 발송 방지

### 비즈니스 맥락
- 메시지 발송 결과를 기록하여 발송 이력 추적
- UNIQUE 제약으로 동일 트리거+채널 조합의 중복 발송 방지
- 실패 시 error_message에 원인 기록

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 프리런칭의 메시지 발송 이력
SELECT * FROM event_prelaunching_message_log WHERE prelaunching_seq = #{seq};
```

---

## event_prelaunching_option (프리런칭 옵션)

summary: 프리런칭 이벤트에서 참여자가 선택할 수 있는 제품 옵션(색상, 사양 등)을 관리하는 테이블. 구독자 전용 옵션 노출 및 선택 카운트도 관리한다.
domain: event
related_tables: event_prelaunching, erp_option

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| prelaunching_seq | int unsigned | NOT NULL, FK | 프리런칭 번호 | |
| product_code | varchar(30) | NULL | 내부 제품코드 | |
| option_name | varchar(255) | NOT NULL | 옵션명 | |
| second_option_name | varchar(255) | NULL | 하위 옵션명 | |
| sort_order | tinyint | NOT NULL, DEFAULT 0 | 정렬 순위 | |
| post_flag | char | DEFAULT 'Y' | 공개(선택 가능) 여부 | 'Y' / 'N' |
| subscriber_post_flag | char | NOT NULL, DEFAULT 'Y' | 구독자 페이지 노출 여부 | 'Y' / 'N' |
| subscriber_selection_count | int unsigned | NOT NULL, DEFAULT 0 | 구독자 옵션 선택 카운트 | |

### 관계
- event_prelaunching: prelaunching_seq FK (ON DELETE CASCADE)
- erp_option: product_code로 ERP 옵션과 연동
- UNIQUE 제약조건: (prelaunching_seq, option_name, second_option_name)

### 비즈니스 맥락
- 프리런칭 참여 시 관심 옵션을 선택하게 하여 수요 예측에 활용
- post_flag: 일반 페이지에서의 노출 여부
- subscriber_post_flag: 구독자 전용 페이지에서의 노출 여부 (일반/구독자 페이지 분리 운영)
- subscriber_selection_count: 구독자의 옵션 선택 횟수 추적
- UPSERT 방식으로 옵션 업데이트 시 subscriber_selection_count 보존

### 자주 쓰는 쿼리 패턴
```sql
-- 프리런칭 옵션 목록 (ERP 옵션명 포함)
SELECT o.seq, o.product_code, o.option_name, o.second_option_name,
  o.sort_order, o.post_flag, o.subscriber_post_flag,
  IFNULL(o.subscriber_selection_count, 0) AS subscriber_selection_count,
  i.option_name AS product_option_name
FROM event_prelaunching_option o
LEFT JOIN erp_option i ON o.product_code = CONCAT(i.category_code, '-', i.brand_code, '-', i.goods_code, '-', i.option_code)
WHERE o.prelaunching_seq = #{seq}
ORDER BY o.sort_order ASC;
```

---

## coupon_list (쿠폰 관리)

summary: 이벤트용 쿠폰을 관리하는 테이블. 이벤트 제목별로 쿠폰을 등록하고 발급/사용 상태를 추적한다.
domain: event
related_tables: (없음 - 독립 테이블)

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint | PK, AUTO_INCREMENT | 쿠폰 시퀀스 | |
| event_title | varchar(50) | NOT NULL, DEFAULT '' | 이벤트 제목 | |
| coupon | varchar(100) | NOT NULL, DEFAULT '' | 쿠폰번호 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일자 | |
| is_send | char | NOT NULL, DEFAULT 'N' | 발급 여부 | 'Y' / 'N' |
| use_date | datetime | NULL | 사용일자 | |
| is_used | char | NOT NULL, DEFAULT 'N' | 사용 여부 | 'Y' / 'N' |

### 관계
- 독립 테이블. event_title로 이벤트를 논리적으로 구분

### 비즈니스 맥락
- 이벤트별로 쿠폰 코드를 대량 등록(엑셀 업로드 지원)
- is_send: 고객에게 발급되었는지 여부
- is_used: 실제 사용되었는지 여부, 사용 시 use_date 기록
- 랜덤 쿠폰 발급 기능: 미사용 쿠폰 중 랜덤으로 1개 선택하여 발급

### 자주 쓰는 쿼리 패턴
```sql
-- 이벤트별 쿠폰 목록 조회
SELECT * FROM coupon_list WHERE event_title = #{title} ORDER BY regist_date DESC;

-- 미사용 쿠폰 랜덤 추첨
SELECT seq FROM coupon_list
WHERE event_title = #{title} AND is_used = 'N' AND use_date IS NULL
ORDER BY RAND() LIMIT 1;

-- 쿠폰 사용 처리
UPDATE coupon_list SET is_used = 'Y', use_date = NOW() WHERE seq = #{seq};
```

---

## erp_free_gift (ERP 사은품 - 레거시)

summary: 몰별 사은품 매핑 정보를 관리하는 레거시 테이블. 쇼핑몰 상품 ID에 대응하는 사은품 제품코드와 MD 이벤트 정보를 저장한다.
domain: event
related_tables: erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| mall_name | varchar(50) | NULL | 쇼핑몰명 | |
| mall_product_id | varchar(100) | NULL | 쇼핑몰 상품 ID | |
| free_gift_product_code | varchar(100) | NULL | 사은품 제품코드 | |
| createdDate | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| modifyDate | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정일시 | |
| md_event_name | varchar(255) | NOT NULL | MD 이벤트명 | |
| md_event_start_date | datetime | NOT NULL | MD 이벤트 시작일 | |
| md_event_end_date | datetime | NOT NULL | MD 이벤트 종료일 | |
| isFinish | varchar(3) | NOT NULL, DEFAULT 'N' | 종료 여부 | 'Y' / 'N' |

### 관계
- erp_product_info: free_gift_product_code로 사은품 상품 정보 참조

### 비즈니스 맥락
- 특정 몰(쇼핑몰)에서 특정 상품 구매 시 사은품을 증정하는 프로모션 관리
- 레거시 테이블로, 신규 사은품 이벤트는 erp_free_gift_event 사용 권장
- isFinish='Y'이면 종료된 이벤트

### 자주 쓰는 쿼리 패턴
```sql
-- 진행 중인 사은품 이벤트 조회
SELECT * FROM erp_free_gift WHERE isFinish = 'N' AND NOW() BETWEEN md_event_start_date AND md_event_end_date;
```

---

## erp_free_gift_event (사은품 이벤트)

summary: 사은품 증정 이벤트를 관리하는 테이블. 이벤트명, 기간, 대상 몰, 사은품 제품코드를 정의한다.
domain: event
related_tables: erp_free_gift_target, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| event_name | varchar(200) | NOT NULL | 이벤트명 | |
| start_date | datetime | NULL | 시작일시 | |
| end_date | datetime | NULL | 종료일시 | |
| event_status | char | NOT NULL, DEFAULT 'N' | 이벤트 상태 | 'Y'(활성) / 'N'(비활성) |
| mall_id_list | varchar(255) | NOT NULL | 대상 몰 ID 목록 (쉼표 구분) | |
| free_gift_product_code | varchar(20) | NOT NULL | 사은품 제품코드 | |
| free_gift_product_name | varchar(200) | NULL | 사은품 제품명 | |
| description | varchar(255) | NULL | 설명 | |
| regist_date | datetime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 등록일시 | |

### 관계
- erp_free_gift_target: event_seq로 1:N (대상 상품 목록)
- erp_product_info: free_gift_product_code로 사은품 상품 정보 참조

### 비즈니스 맥락
- 특정 상품 구매 시 사은품을 자동 증정하는 이벤트 관리
- mall_id_list로 대상 쇼핑몰 지정 (여러 몰에 동시 적용 가능)
- erp_free_gift_target에 등록된 대상 상품을 구매하면 사은품 증정
- free_gift_product_name은 erp_product_info에서 자동 조합하여 저장

### 자주 쓰는 쿼리 패턴
```sql
-- 사은품 이벤트 목록 (대상 상품 수 포함)
SELECT *,
  (SELECT COUNT(*) FROM erp_free_gift_target WHERE event_seq = f_g_e.seq) AS target_product_count
FROM erp_free_gift_event AS f_g_e;
```

---

## erp_free_gift_target (사은품 이벤트 대상 상품)

summary: 사은품 이벤트의 대상 상품(구매 시 사은품이 지급되는 상품)을 관리하는 테이블.
domain: event
related_tables: erp_free_gift_event, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| event_seq | int | NOT NULL | 이벤트 번호 (erp_free_gift_event.seq FK) | |
| target_product_code | varchar(20) | NOT NULL | 대상 제품코드 | |

### 관계
- erp_free_gift_event: event_seq로 N:1
- erp_product_info: target_product_code로 상품 정보 참조
- 인덱스: event_seq, target_product_code

### 비즈니스 맥락
- 특정 이벤트(erp_free_gift_event)에 연결된 대상 상품 목록
- 고객이 대상 상품(target_product_code)을 구매하면 해당 이벤트의 사은품(free_gift_product_code)이 자동 증정됨
- 하나의 이벤트에 여러 대상 상품 등록 가능

### 자주 쓰는 쿼리 패턴
```sql
-- 특정 이벤트의 대상 상품 목록 (상품명 포함)
SELECT CONCAT(CONCAT('[', brand_name, '] '), goods_name,
  IF(option_name IS NULL, '', CONCAT(' - ', option_name))) AS product_name,
  t.target_product_code
FROM erp_free_gift_target AS t
LEFT JOIN erp_product_info AS p ON t.target_product_code = p.product_code
WHERE event_seq = #{seq};
```
