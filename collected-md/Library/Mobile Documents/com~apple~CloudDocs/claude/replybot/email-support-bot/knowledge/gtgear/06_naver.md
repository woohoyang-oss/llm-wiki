# Naver 도메인 테이블 (24 tables)

> 네이버 스마트스토어/커머스 채널 연동을 위한 테이블 모음.
> 주문, 상품, 리뷰, 문의, 랭킹, 정산, 실시간 매출 등 네이버 커머스 운영 전반을 관리한다.

---

## naver_smartstore_orders (스마트스토어 주문)

summary: 네이버 스마트스토어의 개별 주문 데이터를 저장하는 핵심 매출 테이블. AI 매출 분석의 1순위 데이터소스이다.
domain: naver
related_tables: naver_smartstore_order_daily, naver_smartstore_product_daily, naver_commerce_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| account | varchar(50) | PK (복합) | 스토어 계정 식별자 | 'keychron', 'gtgear' |
| product_order_id | varchar(100) | PK (복합) | 상품 주문 ID (주문 내 개별 상품 단위) | |
| order_id | varchar(100) | | 주문 ID (하나의 주문에 여러 상품 가능) | |
| product_no | varchar(50) | | 네이버 상품 번호 | |
| product_name | varchar(500) | | 상품명 | |
| quantity | int | DEFAULT 0 | 주문 수량 | |
| total_price | bigint | DEFAULT 0 | 총 결제 금액 | |
| product_order_status | varchar(50) | | 주문 상태 | 'PAYED', 'DELIVERED', 'CANCELED', 'RETURNED', 'EXCHANGED' |
| order_date | datetime | | 주문 일시 | |
| pay_date | datetime | | 결제 일시 | |
| claim_type | varchar(50) | | 클레임 유형 | 'CANCEL', 'RETURN', 'EXCHANGE', 'ADMIN_CANCEL', NULL |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 레코드 생성 시각 | |

### 관계
- `account` + `product_order_id`가 복합 PK
- `account`로 naver_smartstore_order_daily, naver_smartstore_product_daily, naver_smartstore_qna 등과 연결
- `product_no`로 naver_smartstore_product_daily와 연결

### 비즈니스 맥락
- AI 매출 분석의 **1순위 데이터소스** (Layer 1). 가장 신뢰도 높은 실거래 데이터.
- `product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')`로 유효 주문만 필터링
- `claim_type`으로 클레임(취소/반품/교환) 현황 분석 가능
- 실거래가 = `total_price / quantity`

### 자주 쓰는 쿼리 패턴
```sql
-- SALES_OVERVIEW: 기간별 매출 요약
SELECT COUNT(*) AS order_count,
       IFNULL(SUM(quantity), 0) AS total_qty,
       IFNULL(SUM(total_price), 0) AS total_revenue,
       IFNULL(ROUND(AVG(total_price), 0), 0) AS avg_order_value
FROM naver_smartstore_orders
WHERE account = %s
  AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED');

-- SALES_DAILY: 일별 매출 추이
SELECT DATE(order_date) AS sale_date, COUNT(*) AS order_count,
       SUM(quantity) AS total_qty, SUM(total_price) AS revenue
FROM naver_smartstore_orders
WHERE account = %s AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY DATE(order_date) ORDER BY sale_date;

-- SALES_BY_PRODUCT: 상품별 매출 랭킹
SELECT product_name, product_no, SUM(quantity) AS total_qty,
       SUM(total_price) AS total_revenue,
       ROUND(AVG(total_price / quantity), 0) AS avg_unit_price
FROM naver_smartstore_orders
WHERE account = %s AND order_date >= %s AND order_date < %s
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY product_name, product_no ORDER BY total_revenue DESC;

-- CLAIMS_SUMMARY: 클레임 현황
SELECT claim_type, COUNT(*) AS cnt
FROM naver_smartstore_orders
WHERE account = %s AND order_date >= %s AND order_date < %s
  AND claim_type IS NOT NULL AND claim_type != ''
GROUP BY claim_type ORDER BY cnt DESC;

-- PROBLEM_PRODUCTS: 반품 다발 상품
SELECT product_name, COUNT(*) AS total_orders,
       SUM(CASE WHEN claim_type = 'RETURN' THEN 1 ELSE 0 END) AS returns,
       ROUND(SUM(CASE WHEN claim_type = 'RETURN' THEN 1 ELSE 0 END)*100.0/NULLIF(COUNT(*), 0), 1) AS return_rate
FROM naver_smartstore_orders
WHERE account = %s AND order_date >= %s AND order_date < %s
GROUP BY product_name HAVING COUNT(*) >= %s ORDER BY return_rate DESC;
```

---

## naver_smartstore_order_daily (스마트스토어 일별 주문 집계)

summary: 네이버 스마트스토어의 일별 주문/매출 집계 데이터. 일별 트렌드 분석에 최적화된 요약 테이블이다.
domain: naver
related_tables: naver_smartstore_orders, naver_smartstore_product_daily

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| account | varchar(50) | PK (복합) | 스토어 계정 식별자 | 'keychron', 'gtgear' |
| stat_date | date | PK (복합) | 집계 날짜 | |
| order_cnt | int | DEFAULT 0 | 주문 건수 | |
| paid_cnt | int | DEFAULT 0 | 결제 완료 건수 | |
| cancel_cnt | int | DEFAULT 0 | 취소 건수 | |
| return_cnt | int | DEFAULT 0 | 반품 건수 | |
| revenue | bigint | DEFAULT 0 | 매출액 | |
| cancel_amt | bigint | DEFAULT 0 | 취소 금액 | |
| avg_order_value | int | DEFAULT 0 | 평균 주문 금액 | |

### 관계
- `account` + `stat_date`가 복합 PK
- `account`로 naver_smartstore_orders와 동일 스토어 식별

### 비즈니스 맥락
- naver_smartstore_orders의 일별 요약 버전. 일별 추세 비교에 빠르게 사용 가능.
- 주문/결제/취소/반품 건수와 금액을 일별로 집계
- 순매출 = `revenue - cancel_amt`

### 자주 쓰는 쿼리 패턴
```sql
-- 일별 매출 트렌드
SELECT stat_date, order_cnt, revenue, cancel_cnt, cancel_amt
FROM naver_smartstore_order_daily
WHERE account = %s AND stat_date BETWEEN %s AND %s
ORDER BY stat_date;
```

---

## naver_commerce_order (네이버 커머스 API 주문)

summary: 네이버 커머스 API를 통해 수집한 상세 주문 데이터. 배송, 결제수단, 정산 등 상세 정보를 포함하는 Layer 2 데이터소스이다.
domain: naver
related_tables: naver_smartstore_orders, naver_commerce_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint | PK, AUTO_INCREMENT | 일련번호 | |
| orderId | varchar(30) | NOT NULL, UK (복합) | 주문 ID | |
| productOrderId | varchar(50) | NOT NULL, UK (복합) | 상품 주문 ID | |
| orderDate | datetime | | 주문 일시 | |
| paymentDate | datetime | | 결제 일시 | |
| paymentMeans | varchar(50) | | 결제 수단 | |
| productOrderStatus | varchar(50) | NOT NULL | 상품 주문 상태 | |
| lastChangedDate | datetime | NOT NULL | 마지막 변경 일시 | |
| lastChangedType | varchar(50) | NOT NULL | 마지막 변경 유형 | |
| claimType | varchar(30) | | 클레임 유형 | |
| claimStatus | varchar(50) | | 클레임 상태 | |
| generalPaymentAmount | int | | 일반 결제 금액 | |
| naverMileagePaymentAmount | int | | 네이버 마일리지 결제 금액 | |
| orderDiscountAmount | int | | 주문 할인 금액 | |
| ordererId | varchar(50) | | 주문자 ID | |
| ordererName | varchar(100) | | 주문자 이름 | |
| ordererTel | varchar(20) | | 주문자 전화번호 | |
| sellerProductCode | varchar(50) | | 판매자 상품 코드 | |
| commissionRatingType | varchar(50) | | 수수료 등급 유형 | |
| paymentCommission | int | | 결제 수수료 | |
| expectedSettlementAmount | int | | 예상 정산 금액 | |
| inflowPath | varchar(100) | | 유입 경로 | |
| quantity | int | | 수량 | |
| productId | varchar(50) | | 상품 ID | |
| productName | varchar(255) | | 상품명 | |
| productOption | varchar(255) | | 상품 옵션 | |
| unitPrice | int | | 단가 | |
| productDiscountAmount | int | | 상품 할인 금액 | |
| deliveryFeeAmount | int | | 배송비 | |
| sellerBurdenDiscountAmount | int | | 판매자 부담 할인 금액 | |
| product_order_status | varchar(50) | | 상품 주문 상태 (별도) | |
| decisionDate | datetime | | 구매 확정 일시 | |
| packageNumber | varchar(50) | | 묶음 번호 | |
| placeOrderDate | datetime | | 발주 일시 | |
| placeOrderStatus | varchar(50) | | 발주 상태 | |
| shippingDueDate | datetime | | 배송 예정 일시 | |
| totalPaymentAmount | int | | 총 결제 금액 | |
| totalProductAmount | int | | 총 상품 금액 | |
| deliveryAttributeType | varchar(50) | | 배송 속성 유형 | |
| merchantChannelId | varchar(50) | | 판매 채널 ID | |
| shippingAddressBase | varchar(255) | | 배송지 기본 주소 | |
| shippingAddressZipCode | varchar(20) | | 배송지 우편번호 | |
| shippingAddressDetail | varchar(255) | | 배송지 상세 주소 | |
| shippingAddressTel | varchar(20) | | 배송지 전화번호 | |
| shippingAddressName | varchar(100) | | 수취인명 | |
| deliveryStatus | varchar(50) | | 배송 상태 | |
| sendDate | datetime | | 발송 일시 | |
| trackingNumber | varchar(50) | | 운송장 번호 | |
| deliveredDate | datetime | | 배송 완료 일시 | |
| deliveryCompany | varchar(100) | | 택배사 | |
| deliveryMethod | varchar(50) | | 배송 방법 | |
| optionManageCode | varchar(200) | | 옵션 관리 코드 | |
| brandCode | varchar(20) | NOT NULL | 투비 브랜드 코드 | |
| is_stock_decreased | varchar(1) | DEFAULT 'N' | 재고 차감 여부 | 'Y', 'N' |
| stockDecreasedDate | datetime | | 재고 차감 일시 | |
| is_notification_send | varchar(1) | DEFAULT 'N' | 알림톡 발송 여부 | 'Y', 'N' |

### 관계
- `orderId` + `productOrderId` UNIQUE 복합키
- `productId`로 naver_commerce_product_info와 연결
- `brandCode`로 브랜드 필터링

### 비즈니스 맥락
- Layer 2 데이터소스. 네이버 커머스 API에서 직접 수집한 상세 주문 데이터.
- 배송 추적, 정산, 결제수단 등 상세 분석 가능
- `is_stock_decreased`로 ERP 재고 차감 연동 상태 확인
- `optionManageCode`로 사내 제품 코드 매핑

### 자주 쓰는 쿼리 패턴
```sql
-- 브랜드별 주문 조회
SELECT nco.*, ... FROM naver_commerce_order AS nco
WHERE nco.brandCode = %s
  AND DATE_FORMAT(nco.orderDate, '%%Y-%%m-%%d') BETWEEN %s AND %s
ORDER BY nco.orderDate DESC;
```

---

## naver_commerce_product_info (네이버 커머스 상품 정보)

summary: 네이버 커머스 API 상품 코드와 옵션 관리 코드 매핑 정보를 저장하는 테이블.
domain: naver
related_tables: naver_commerce_order

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| product_code | varchar(255) | NOT NULL | 상품 코드 | |
| product_name | text | NOT NULL | 상품명 | |
| option_manage_code | varchar(255) | NOT NULL | 옵션 관리 코드 | |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| modified_at | timestamp | DEFAULT CURRENT_TIMESTAMP, ON UPDATE | 수정일시 | |

### 관계
- naver_commerce_order의 `productId`, `optionManageCode`와 매핑

### 비즈니스 맥락
- 네이버 상품 코드와 사내 옵션 관리 코드 간 매핑 테이블
- naver_commerce_order 조회 시 상품 식별에 사용

### 자주 쓰는 쿼리 패턴
```sql
-- 상품 코드로 상품 정보 조회
SELECT * FROM naver_commerce_product_info WHERE product_code = %s;
```

---

## naver_smartstore_product_daily (스마트스토어 상품별 일별 실적)

summary: 네이버 스마트스토어 상품별 일별 판매 실적 데이터. 상품별 매출 트렌드 분석에 사용한다.
domain: naver
related_tables: naver_smartstore_orders, naver_smartstore_order_daily

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| account | varchar(50) | PK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| stat_date | date | PK (복합) | 집계 날짜 | |
| product_no | varchar(50) | PK (복합) | 네이버 상품 번호 | |
| channel_product_no | varchar(50) | | 채널 상품 번호 | |
| product_name | varchar(500) | | 상품명 | |
| order_cnt | int | DEFAULT 0 | 주문 건수 | |
| quantity | int | DEFAULT 0 | 주문 수량 | |
| revenue | bigint | DEFAULT 0 | 매출액 | |
| cancel_cnt | int | DEFAULT 0 | 취소 건수 | |
| cancel_amt | bigint | DEFAULT 0 | 취소 금액 | |

### 관계
- `account` + `stat_date` + `product_no` 복합 PK
- `product_no`로 naver_smartstore_orders와 연결

### 비즈니스 맥락
- 상품 단위 일별 성과 추적에 최적화
- 상품별 주문 건수, 수량, 매출, 취소 정보 포함

### 자주 쓰는 쿼리 패턴
```sql
-- 상품별 일별 매출 추이
SELECT stat_date, product_name, order_cnt, quantity, revenue
FROM naver_smartstore_product_daily
WHERE account = %s AND stat_date BETWEEN %s AND %s
ORDER BY stat_date, revenue DESC;
```

---

## naver_smartstore_qna (스마트스토어 Q&A)

summary: 네이버 스마트스토어 상품 Q&A 데이터. 질문/답변 내용 및 답변 상태를 관리한다.
domain: naver
related_tables: naver_store_product_qna, naver_smartstore_orders

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| account | varchar(50) | PK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| qna_id | varchar(100) | PK (복합) | Q&A 고유 ID | |
| product_no | varchar(50) | | 상품 번호 | |
| product_name | varchar(500) | | 상품명 | |
| question_content | text | | 질문 내용 | |
| answer_content | text | | 답변 내용 | |
| question_date | datetime | | 질문 등록 일시 | |
| answer_date | datetime | | 답변 등록 일시 | |
| answered | tinyint(1) | DEFAULT 0 | 답변 여부 | 0(미답변), 1(답변완료) |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 레코드 생성 시각 | |

### 관계
- `account` + `qna_id` 복합 PK
- 인덱스: `idx_qna_answered(account, answered)`, `idx_qna_date(account, question_date)`

### 비즈니스 맥락
- AI CS 자동 응답 시스템의 Q&A 데이터소스
- `answered = 0` 으로 미답변 Q&A 모니터링
- 스토어별(account) 미답변 집계에 활용

### 자주 쓰는 쿼리 패턴
```sql
-- QNA_UNANSWERED: 미답변 Q&A 목록
SELECT account, product_name AS title, question_content,
       question_date AS created_date,
       DATEDIFF(CURDATE(), DATE(question_date)) AS days_waiting
FROM naver_smartstore_qna
WHERE answered = 0
ORDER BY question_date ASC;

-- 스토어별 미답변 집계
SELECT account, COUNT(*) AS unanswered
FROM naver_smartstore_qna
WHERE answered = 0
GROUP BY account;
```

---

## naver_smartstore_settlement_daily (스마트스토어 일별 정산)

summary: 네이버 스마트스토어 일별 정산 데이터. 수수료, 서비스 이용료, 순정산 금액을 관리한다.
domain: naver
related_tables: naver_smartstore_order_daily

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| account | varchar(50) | PK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| stat_date | date | PK (복합) | 정산 날짜 | |
| settle_amount | bigint | DEFAULT 0 | 정산 금액 | |
| commission | bigint | DEFAULT 0 | 수수료 | |
| service_fee | bigint | DEFAULT 0 | 서비스 이용료 | |
| net_amount | bigint | DEFAULT 0 | 순정산 금액 | |
| order_count | int | DEFAULT 0 | 주문 건수 | |
| claim_deduction | bigint | DEFAULT 0 | 클레임 공제 금액 | |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성 시각 | |

### 관계
- `account` + `stat_date` 복합 PK
- naver_smartstore_order_daily와 동일 기간 비교 가능

### 비즈니스 맥락
- 실제 정산 금액과 수수료 구조 파악에 사용
- `net_amount` = 수수료, 서비스이용료, 클레임 공제 후 순수익

### 자주 쓰는 쿼리 패턴
```sql
-- 일별 정산 현황
SELECT stat_date, settle_amount, commission, service_fee, net_amount
FROM naver_smartstore_settlement_daily
WHERE account = %s AND stat_date BETWEEN %s AND %s
ORDER BY stat_date;
```

---

## naver_store_customer_inquiries (고객 문의)

summary: 네이버 스마트스토어 고객 문의 데이터. AI 자동 답변 생성 및 CS 검수 프로세스를 지원하는 핵심 CS 테이블이다.
domain: naver
related_tables: naver_store_inquiry_issue, naver_store_inquiry_greetings_template, naver_store_product_qna

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| inquiry_no | bigint | PK | 네이버 문의 고유 번호 | |
| brand_channel | varchar(20) | NOT NULL | 브랜드 채널 코드 | 'KEYCHRON', 'GTGEAR', 'AIPER' |
| internal_product_code | varchar(30) | | 사내 제품 코드 | 'K-0068-0001-0001' |
| category | varchar(50) | | 문의 유형 | '상품', '배송', '반품', '교환', '환불', '기타' |
| title | varchar(500) | NOT NULL | 문의 제목 | |
| inquiry_content | text | NOT NULL | 문의 내용 | |
| inquiry_registration_date_time | datetime | NOT NULL | 문의 등록 일시 | |
| customer_id | varchar(100) | | 고객 ID | |
| customer_name | varchar(100) | NOT NULL | 고객 이름 | |
| order_id | varchar(100) | NOT NULL | 주문 ID | |
| product_no | varchar(100) | | 네이버 상품 번호 | |
| product_order_id_list | varchar(500) | | 상품 주문 ID 목록 (쉼표 구분) | |
| product_name | varchar(500) | | 상품명 | |
| product_order_option | varchar(500) | | 주문 상품 옵션 | |
| answer_content | text | | 답변 내용 (AI 또는 CS 작성) | |
| answer_content_id | | | 답변 ID | |
| answer_template_no | int | | 답변 템플릿 번호 | |
| answer_registration_date_time | datetime | | 답변 등록 일시 | |
| answered | tinyint(1) | DEFAULT 0, NOT NULL | 답변 완료 여부 | 0(미답변), 1(답변완료) |
| ai_answer_generated | tinyint(1) | DEFAULT 0, NOT NULL | AI 답변 생성 완료 여부 | |
| cs_reviewed | tinyint(1) | DEFAULT 0, NOT NULL | CS 검수 완료 여부 | |
| processing_status | varchar(50) | DEFAULT 'pending', NOT NULL | 처리 상태 | 'pending', 'ai_processing', 'ai_completed', 'cs_reviewing', 'completed', 'failed' |
| agent_answer | text | | AI 생성 테스트용 답변 | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 레코드 생성 시각 | |
| updated_at | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE, NOT NULL | 레코드 수정 시각 | |

### 관계
- `inquiry_no`가 PK
- `inquiry_no`로 naver_store_inquiry_issue와 1:N 연결
- 인덱스: `idx_brand_ai_pending`, `idx_brand_cs_pending`, `idx_brand_category`, `idx_inquiry_date`

### 비즈니스 맥락
- AI 자동 답변 파이프라인: pending → ai_processing → ai_completed → cs_reviewing → completed
- 멀티 브랜드 지원 (KEYCHRON, GTGEAR, AIPER)
- `answered = 0`으로 미답변 문의 모니터링
- `agent_answer`는 AI가 생성한 답변, `answer_content`는 최종 등록된 답변

### 자주 쓰는 쿼리 패턴
```sql
-- UNANSWERED_INQUIRIES: 미답변 문의 목록
SELECT inquiry_no, brand_channel, category, title,
       inquiry_registration_date_time AS created_date,
       DATEDIFF(CURDATE(), DATE(inquiry_registration_date_time)) AS days_waiting
FROM naver_store_customer_inquiries
WHERE answered = 0
ORDER BY inquiry_registration_date_time ASC;

-- AI가 처리할 문의 조회
SELECT * FROM naver_store_customer_inquiries
WHERE brand_channel = %s
  AND ai_answer_generated = FALSE
  AND processing_status IN ('pending', 'ai_processing')
ORDER BY created_at ASC LIMIT %s;

-- CS 검수 대기 문의 조회
SELECT * FROM naver_store_customer_inquiries
WHERE brand_channel = %s
  AND ai_answer_generated = TRUE AND cs_reviewed = FALSE
  AND processing_status IN ('ai_completed', 'cs_reviewing')
ORDER BY created_at ASC LIMIT %s;

-- 브랜드별 처리 상태별 문의 통계
SELECT brand_channel, processing_status, COUNT(*) AS count
FROM naver_store_customer_inquiries
GROUP BY brand_channel, processing_status;
```

---

## naver_store_inquiry_greetings_template (문의 답변 인사말 템플릿)

summary: 네이버 스토어 문의 답변 시 사용하는 시작/마무리 인사말 템플릿을 관리하는 테이블.
domain: naver
related_tables: naver_store_customer_inquiries, naver_store_product_qna

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| greeting_type | char | DEFAULT '', NOT NULL | 인사말 타입 | 'o'(시작), 'c'(마무리) |
| greeting_text | varchar(255) | DEFAULT '', NOT NULL | 인사말 내용 | |
| is_active | char | DEFAULT 'N' | 사용 여부 | 'Y', 'N' |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 생성 시각 | |

### 관계
- naver_store_customer_inquiries, naver_store_product_qna 답변 생성 시 참조

### 비즈니스 맥락
- AI 답변 생성 시 시작 인사말(greeting_type='o')과 마무리 인사말(greeting_type='c')을 자동 삽입
- `is_active = 'Y'`인 템플릿만 사용

### 자주 쓰는 쿼리 패턴
```sql
-- 활성 인사말 조회
SELECT
  (SELECT greeting_text FROM naver_store_inquiry_greetings_template
   WHERE greeting_type = 'o' AND is_active = 'Y') AS greetingText,
  (SELECT greeting_text FROM naver_store_inquiry_greetings_template
   WHERE greeting_type = 'c' AND is_active = 'Y') AS closingText;
```

---

## naver_store_inquiry_issue (문의 이슈 관리)

summary: 고객 문의에 대한 이슈(실패, 개입 필요) 상태를 추적하는 테이블.
domain: naver
related_tables: naver_store_customer_inquiries

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| inquiry_no | bigint | NOT NULL | 네이버 문의 번호 (FK) | |
| issue_type | char | | 이슈 타입 | 'F'(실패), 'I'(개입필요) |
| status | char | DEFAULT 'P' | 처리 상태 | 'P'(대기), 'C'(완료) |

### 관계
- `inquiry_no`로 naver_store_customer_inquiries와 N:1 연결

### 비즈니스 맥락
- AI 답변 생성 실패 또는 CS 개입이 필요한 문의를 별도 추적
- 문의 목록 조회 시 issue_type, status를 GROUP_CONCAT하여 표시

### 자주 쓰는 쿼리 패턴
```sql
-- 문의별 이슈 조회
SELECT GROUP_CONCAT(issue_type SEPARATOR ', ') AS issue_types,
       GROUP_CONCAT(status SEPARATOR ', ') AS statuses
FROM naver_store_inquiry_issue
WHERE inquiry_no = %s;
```

---

## naver_store_product_qna (상품 문의)

summary: 네이버 스마트스토어 상품별 Q&A 데이터. AI 자동 답변 생성과 CS 검수를 지원하며 브랜드별로 관리된다.
domain: naver
related_tables: naver_store_customer_inquiries, naver_store_inquiry_greetings_template, naver_smartstore_qna

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| question_id | bigint | PK | 네이버 상품 문의 고유 번호 | |
| brand_channel | varchar(20) | NOT NULL | 브랜드 채널 코드 | 'KEYCHRON', 'GTGEAR', 'AIPER' |
| internal_product_code | varchar(50) | | 사내 제품 코드 | |
| question | text | NOT NULL | 질문 내용 | |
| create_date | datetime | NOT NULL | 문의 등록 일시 | |
| masked_writer_id | varchar(100) | | 마스킹된 작성자 ID | |
| product_id | varchar(50) | | 네이버 상품 번호 | |
| product_name | varchar(500) | | 상품명 | |
| answer | text | | 답변 내용 | |
| answered | tinyint(1) | DEFAULT 0 | 답변 여부 | 0(미답변), 1(답변완료) |
| ai_answer_generated | tinyint(1) | DEFAULT 0 | AI 답변 생성 완료 여부 | |
| cs_reviewed | tinyint(1) | DEFAULT 0 | CS 검수 완료 여부 | |
| processing_status | varchar(30) | DEFAULT 'pending' | 처리 상태 | 'pending', 'ai_processing', 'ai_completed', 'cs_reviewing', 'completed', 'failed' |
| agent_answer | text | | AI 생성 답변 | |
| usability | tinyint(1) | | AI 답변 활용 가능 여부 | |
| created_at | timestamp | DEFAULT CURRENT_TIMESTAMP | 레코드 생성 시각 | |
| updated_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 레코드 수정 시각 | |

### 관계
- `question_id`가 PK
- 인덱스: `idx_brand_channel`, `idx_brand_answered`, `idx_brand_status`, `idx_product_id`

### 비즈니스 맥락
- naver_store_customer_inquiries와 동일한 AI 자동 답변 파이프라인 적용
- 상품별 Q&A이므로 product_id/product_name으로 상품별 분석 가능
- Weaviate 벡터 DB 학습 데이터로도 활용 (answer != agent_answer인 건)
- `usability` 필드로 AI 답변 품질 추적

### 자주 쓰는 쿼리 패턴
```sql
-- AI가 처리할 상품 문의 조회
SELECT question_id, question, product_id, product_name
FROM naver_store_product_qna
WHERE agent_answer IS NULL
  AND question NOT LIKE '%%개발 테스트%%'
ORDER BY question_id DESC LIMIT 2;

-- 답변 프리뷰 조회 (기간별)
SELECT question_id, product_id, product_name, question,
       create_date, answer, agent_answer, usability
FROM naver_store_product_qna
WHERE DATE(create_date) >= %s AND %s >= DATE(create_date)
ORDER BY created_at DESC;

-- Weaviate 학습 데이터 추출
SELECT question, product_id, product_name, answer
FROM naver_store_product_qna
WHERE DATE(updated_at) = CURRENT_DATE
  AND answer IS NOT NULL AND answer != agent_answer;
```

---

## naver_reviews (네이버 리뷰)

summary: 네이버 스마트스토어 상품 리뷰 데이터. 리뷰 내용, 평점, AI 답변을 관리한다.
domain: naver
related_tables: naver_store_review_crawler, naver_store_review_item

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| productNo | varchar(150) | NOT NULL | 상품 번호 | |
| productName | varchar(255) | NOT NULL | 상품명 | |
| reviewType | varchar(30) | NOT NULL | 리뷰 유형 | |
| reviewScore | varchar(5) | NOT NULL | 리뷰 평점 | '1'~'5' |
| reviewContent | text | NOT NULL | 리뷰 내용 | |
| helpCount | int unsigned | DEFAULT 0 | 도움이 됨 수 | |
| writerId | varchar(100) | NOT NULL | 작성자 ID | |
| createDate | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 작성일시 | |
| modifyDate | varchar(30) | | 수정일시 | |
| replyContent | text | | 답변 내용 | |
| brandCode | varchar(10) | NOT NULL | 브랜드 코드 | |
| updatedAt | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 수정 시각 | |
| confirmDate | datetime | | 네이버 등록 일시 | |
| reviewId | varchar(255) | NOT NULL, UNIQUE | 리뷰 고유 번호 | |

### 관계
- `reviewId` UNIQUE
- 인덱스: `idx_createDate`, `idx_productNo`, `idx_writerId`

### 비즈니스 맥락
- 네이버 스마트스토어 공식 리뷰 데이터
- `replyContent`에 AI 생성 답변을 저장, `confirmDate`로 네이버 등록 여부 확인
- `brandCode`로 브랜드별 리뷰 관리

### 자주 쓰는 쿼리 패턴
```sql
-- 전체 리뷰 최신순 조회
SELECT * FROM naver_reviews ORDER BY reviewId DESC;

-- AI 답변 확정
UPDATE naver_reviews SET replyContent = %s WHERE seq = %s;
```

---

## naver_ranking_tracker_list (네이버 랭킹 추적 목록)

summary: 네이버 쇼핑 검색 랭킹 모니터링 대상 목록. 키워드-상품 조합별 랭킹을 추적한다.
domain: naver
related_tables: naver_shopping_ranking

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| mid | varchar(50) | NOT NULL, UK (복합) | 타겟 아이디 (상품 식별) | |
| product_name | varchar(255) | | 제품 이름 | |
| keyword | varchar(255) | NOT NULL, UK (복합) | 검색 키워드 | |
| ranking | int | | 탐지 랭킹 | |
| timer | int | | 타이머 | |
| status | char | DEFAULT 'Y', NOT NULL | 상태 | 'Y', 'N' |
| title | varchar(255) | | 검색 결과 제목 | |
| link | varchar(255) | | 상품 링크 | |
| image | varchar(255) | | 상품 이미지 URL | |
| lprice | varchar(100) | | 최저가 | |
| hprice | varchar(100) | | 최고가 | |
| mallName | varchar(255) | | 쇼핑몰명 | |
| productType | varchar(100) | | 상품 유형 | |
| brand | varchar(255) | | 브랜드 | |
| maker | varchar(255) | | 제조사 | |
| category1~4 | varchar(255) | | 카테고리 1~4단계 | |
| update_datetime | datetime | | 마지막 업데이트 일시 | |

### 관계
- `mid` + `keyword` UNIQUE 복합키
- `seq`로 naver_shopping_ranking의 `tracker_seq`와 1:N 연결

### 비즈니스 맥락
- 네이버 쇼핑 검색 랭킹 모니터링의 대상 목록
- 하나의 상품(mid)에 여러 키워드를 등록하여 키워드별 랭킹 추적
- naver_shopping_ranking에 시계열 랭킹 데이터 저장

### 자주 쓰는 쿼리 패턴
```sql
-- 모니터링 목록 조회 (현재/전일 랭킹 포함)
SELECT t.seq, t.mid, t.keyword, t.product_name,
       pr.current_last_ranking, pr.yesterday_last_ranking
FROM naver_ranking_tracker_list AS t
LEFT JOIN pivoted_rankings AS pr ON t.seq = pr.tracker_seq;
```

---

## naver_shopping_ranking (네이버 쇼핑 랭킹)

summary: 네이버 쇼핑 검색 키워드별 랭킹 시계열 데이터. 시간대별 순위 변동을 추적한다.
domain: naver
related_tables: naver_ranking_tracker_list

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| datetime | datetime | NOT NULL | 감지 일시 | |
| tracker_seq | int | NOT NULL | 모니터링 번호 (FK) | |
| ranking | varchar(100) | NOT NULL | 순위 | '1'~'100', '-'(미검출) |
| flag | char | DEFAULT 'N', NOT NULL | 변동 여부 | 'Y', 'N' |
| title | varchar(255) | | 상품 제목 | |
| link | varchar(255) | | 상품 링크 | |
| image | varchar(255) | | 상품 이미지 URL | |
| lprice | varchar(100) | | 최저가 | |
| hprice | varchar(255) | | 최고가 | |
| category1~4 | varchar(255) | | 카테고리 1~4단계 | |
| tag | varchar(255) | | 판매자 태그 | |

### 관계
- `tracker_seq`로 naver_ranking_tracker_list의 `seq`와 N:1 연결
- 인덱스: `idx_naver_ranking_timeline(tracker_seq ASC, datetime DESC)`, `idx_tracker_datetime`

### 비즈니스 맥락
- 시간대별 검색 랭킹 변동 추적 (시계열)
- `ranking = '-'`는 검색 결과에 미표출을 의미
- 차트 조회 시 `ranking = '-'`인 경우 100으로 변환하여 시각화

### 자주 쓰는 쿼리 패턴
```sql
-- 키워드별 랭킹 차트 데이터
SELECT seq, datetime, tracker_seq, IF(ranking = '-', 100, ranking) AS ranking,
       title, lprice, hprice, category1, category2, category3, category4, tag
FROM naver_shopping_ranking
WHERE tracker_seq IN (SELECT seq FROM naver_ranking_tracker_list)
  AND datetime >= %s AND datetime <= %s;
```

---

## naver_store_review_crawler (리뷰 크롤러 설정)

summary: 네이버 스토어 리뷰 크롤링 대상 설정을 관리하는 테이블. 크롤링할 상점과 상품을 정의한다.
domain: naver
related_tables: naver_store_review_item, naver_store_review_photo

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| store_type | varchar(30) | | 상점 종류 | |
| remark | text | | 비고 | |
| store_name | varchar(255) | NOT NULL | 상점명 | |
| product_id | varchar(255) | NOT NULL | 상품 식별코드 | |
| page_url | text | NOT NULL | 페이지 URL | |
| regist_date | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 등록 일시 | |

### 관계
- `seq`로 naver_store_review_item의 `crawler_seq`와 1:N 연결 (CASCADE)

### 비즈니스 맥락
- 경쟁사/자사 스토어 리뷰를 크롤링하기 위한 대상 설정
- 크롤링된 리뷰는 naver_store_review_item에 저장

### 자주 쓰는 쿼리 패턴
```sql
-- 크롤러 설정 목록 조회
SELECT seq AS crawler_seq, store_type, remark, store_name, product_id, page_url, regist_date
FROM naver_store_review_crawler;
```

---

## naver_store_review_item (크롤링 리뷰 아이템)

summary: 크롤링된 네이버 스토어 리뷰 데이터. 닉네임, 옵션, 별점, 내용 등을 저장한다.
domain: naver
related_tables: naver_store_review_crawler, naver_store_review_photo

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| crawler_seq | int unsigned | NOT NULL, FK | 크롤러 번호 | |
| nickname | varchar(100) | | 작성자 닉네임 | |
| options | varchar(255) | | 선택 옵션 | |
| date | varchar(20) | | 작성일 | |
| rating | int unsigned | DEFAULT 0, NOT NULL | 별점 | 0~5 |
| content | text | | 리뷰 내용 | |
| like | int unsigned | DEFAULT 0, NOT NULL | 좋아요 수 | |

### 관계
- `crawler_seq`로 naver_store_review_crawler와 N:1 연결 (CASCADE DELETE)
- `seq`로 naver_store_review_photo와 1:N 연결

### 비즈니스 맥락
- 크롤링된 리뷰 아이템. 경쟁사/자사 리뷰 분석에 활용.

### 자주 쓰는 쿼리 패턴
```sql
-- 크롤러별 리뷰 조회
SELECT seq AS review_seq, crawler_seq, nickname, options, date, rating, content, `like`
FROM naver_store_review_item
WHERE crawler_seq = %s;
```

---

## naver_store_review_photo (크롤링 리뷰 사진)

summary: 크롤링된 리뷰에 첨부된 사진 URL을 저장하는 테이블.
domain: naver
related_tables: naver_store_review_item

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| review_seq | int unsigned | NOT NULL, FK | 리뷰 번호 | |
| url | text | NOT NULL | 사진 URL | |

### 관계
- `review_seq`로 naver_store_review_item과 N:1 연결 (CASCADE DELETE)

### 비즈니스 맥락
- 리뷰 상세 조회 시 첨부 사진 표시에 사용

### 자주 쓰는 쿼리 패턴
```sql
-- 리뷰별 사진 조회
SELECT seq AS photo_seq, review_seq, url
FROM naver_store_review_photo
WHERE review_seq = %s;
```

---

## naver_tracking_item (추적 상품)

summary: 네이버 쇼핑 랭킹 상위 노출 상품 중 추적 대상으로 등록된 상품 목록.
domain: naver
related_tables: naver_ranking_tracker_list, naver_shopping_ranking

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| channelProductNo | varchar(200) | NOT NULL | 채널 상품 번호 | |
| is_tracking | char | DEFAULT 'N', NOT NULL | 추적 여부 | 'Y', 'N' |
| product_name | varchar(255) | NOT NULL | 상품명 | |
| createdBy | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 등록 일시 | |

### 관계
- `channelProductNo`는 naver_ranking_tracker_list의 `mid`와 대응

### 비즈니스 맥락
- 랭킹 50위 이내 노출 상품 중 자동 추적 대상 등록
- `is_tracking = 'Y'`인 상품만 활성 추적

### 자주 쓰는 쿼리 패턴
```sql
-- 추적 미등록 상품 매칭
SELECT t.mid, r.title AS product_name
FROM naver_shopping_ranking AS r
JOIN naver_ranking_tracker_list AS t ON r.tracker_seq = t.seq
WHERE r.ranking != '-' AND r.ranking <= 50
  AND t.mid NOT IN (SELECT channelProductNo FROM naver_tracking_item);
```

---

## naver_trend_daily (네이버 데이터랩 트렌드)

summary: 네이버 데이터랩 검색 트렌드 일별 데이터. 키워드 그룹별 상대적 검색량(ratio)을 저장한다.
domain: naver
related_tables: (독립 테이블)

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | bigint | PK, AUTO_INCREMENT | 일련번호 | |
| group_name | varchar(50) | NOT NULL, UK (복합) | 키워드 그룹명 | |
| account | varchar(20) | NOT NULL | 연관 계정 | 'keychron', 'gtgear', 'common' |
| stat_date | date | NOT NULL, UK (복합) | 집계 날짜 | |
| ratio | decimal(10,5) | DEFAULT 0.00000 | 상대적 검색량 (0~100) | |
| keywords | varchar(500) | | 포함 키워드 목록 | |
| collected_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수집 시각 | |

### 관계
- `group_name` + `stat_date` UNIQUE 복합키
- 인덱스: `idx_account_date(account, stat_date)`, `idx_date(stat_date)`

### 비즈니스 맥락
- 네이버 데이터랩 API에서 수집한 검색 트렌드 데이터
- `ratio`는 최대값 100 기준의 상대적 검색량
- 계정별(keychron/gtgear/common) 키워드 그룹 트렌드 분석

### 자주 쓰는 쿼리 패턴
```sql
-- 키워드 그룹별 트렌드 조회
SELECT stat_date, group_name, ratio
FROM naver_trend_daily
WHERE account = %s AND stat_date BETWEEN %s AND %s
ORDER BY stat_date;
```

---

## smartstore_channel_detail (채널별 유입 상세)

summary: 스마트스토어 채널별(검색, 직접유입, 광고 등) 유입 및 구매 성과 데이터.
domain: naver
related_tables: smartstore_keyword_daily, naver_smartstore_order_daily

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int | PK, AUTO_INCREMENT | 일련번호 | |
| account | varchar(50) | NOT NULL, UK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| start_date | date | NOT NULL, UK (복합) | 시작 날짜 | |
| end_date | date | NOT NULL | 종료 날짜 | |
| device_category | varchar(50) | DEFAULT '', UK (복합) | 기기 유형 | 'mobile', 'pc' |
| channel_group | varchar(100) | DEFAULT '' | 채널 그룹 | '검색', '직접유입', '외부유입' |
| channel_name | varchar(200) | NOT NULL, UK (복합) | 채널명 | '네이버쇼핑검색', '직접URL' |
| num_users | int | DEFAULT 0 | 사용자 수 | |
| num_interactions | int | DEFAULT 0 | 인터랙션 수 | |
| pv | int | DEFAULT 0 | 페이지뷰 | |
| num_purchases | int | DEFAULT 0 | 구매 건수 | |
| pay_amount | bigint | DEFAULT 0 | 결제 금액 | |
| collected_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수집 시각 | |

### 관계
- `account` + `start_date` + `end_date` + `device_category` + `channel_name` UNIQUE 복합키

### 비즈니스 맥락
- 어떤 채널에서 유입되어 구매까지 이어지는지 분석
- 기기 유형(mobile/pc)별 성과 비교 가능
- 구매 전환율 = `num_purchases / num_users`

### 자주 쓰는 쿼리 패턴
```sql
-- 채널별 성과 조회
SELECT channel_name, SUM(num_users) AS users, SUM(pv) AS pv,
       SUM(num_purchases) AS purchases, SUM(pay_amount) AS revenue
FROM smartstore_channel_detail
WHERE account = %s AND start_date BETWEEN %s AND %s
GROUP BY channel_name ORDER BY revenue DESC;
```

---

## smartstore_keyword_daily (키워드별 일별 성과)

summary: 스마트스토어 유입 키워드별 일별 성과(인터랙션, 구매, 결제 금액) 데이터.
domain: naver
related_tables: smartstore_channel_detail

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int | PK, AUTO_INCREMENT | 일련번호 | |
| account | varchar(50) | NOT NULL, UK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| start_date | date | NOT NULL, UK (복합) | 시작 날짜 | |
| end_date | date | NOT NULL | 종료 날짜 | |
| keyword | varchar(200) | NOT NULL, UK (복합) | 검색 키워드 | |
| num_interactions | int | DEFAULT 0 | 인터랙션 수 | |
| num_purchases | int | DEFAULT 0 | 구매 건수 | |
| pay_amount | bigint | DEFAULT 0 | 결제 금액 | |
| collected_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수집 시각 | |

### 관계
- `account` + `start_date` + `end_date` + `keyword` UNIQUE 복합키

### 비즈니스 맥락
- 어떤 검색 키워드가 매출에 기여하는지 분석
- 키워드별 구매 전환율 = `num_purchases / num_interactions`

### 자주 쓰는 쿼리 패턴
```sql
-- 키워드별 매출 기여도 조회
SELECT keyword, SUM(num_interactions) AS interactions,
       SUM(num_purchases) AS purchases, SUM(pay_amount) AS revenue
FROM smartstore_keyword_daily
WHERE account = %s AND start_date BETWEEN %s AND %s
GROUP BY keyword ORDER BY revenue DESC;
```

---

## smartstore_realtime_hourly (실시간 시간별 매출)

summary: 스마트스토어 시간별 실시간 매출 및 인터랙션 데이터. GFA 광고 효과 분석에도 활용된다.
domain: naver
related_tables: smartstore_realtime_products, naver_smartstore_order_daily

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int | PK, AUTO_INCREMENT | 일련번호 | |
| account | varchar(50) | NOT NULL, UK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| stat_date | date | NOT NULL, UK (복합) | 집계 날짜 | |
| hour_of_day | tinyint | NOT NULL, UK (복합) | 시간대 (0~23) | |
| pay_amount | bigint | DEFAULT 0 | 결제 금액 | |
| num_interactions | int | DEFAULT 0 | 인터랙션 수 (주문 건수) | |
| collected_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수집 시각 | |

### 관계
- `account` + `stat_date` + `hour_of_day` UNIQUE 복합키

### 비즈니스 맥락
- 실시간 매출 현황 모니터링. 시간대별 매출 패턴 분석 가능.
- GFA 광고 효과 분석에서 SS(스마트스토어) 시간별 매출과 GFA 광고비 상관관계 분석에 사용

### 자주 쓰는 쿼리 패턴
```sql
-- SS_HOURLY_TODAY: 오늘 시간별 매출 (GFA 상관 분석용)
SELECT hour_of_day AS hour_slot, pay_amount AS revenue, num_interactions AS order_cnt
FROM smartstore_realtime_hourly
WHERE account = %s AND stat_date = CURDATE()
ORDER BY hour_of_day;
```

---

## smartstore_realtime_products (실시간 상품별 매출)

summary: 스마트스토어 시간별 상품 매출 랭킹 데이터. 실시간 인기 상품 순위를 추적한다.
domain: naver
related_tables: smartstore_realtime_hourly

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int | PK, AUTO_INCREMENT | 일련번호 | |
| account | varchar(50) | NOT NULL, UK (복합) | 스토어 계정 | 'keychron', 'gtgear' |
| stat_date | date | NOT NULL, UK (복합) | 집계 날짜 | |
| hour_of_day | tinyint | DEFAULT 0, NOT NULL, UK (복합) | 시간대 (0~23) | |
| rank_position | tinyint | NOT NULL, UK (복합) | 순위 | 1, 2, 3... |
| product_name | varchar(500) | NOT NULL | 상품명 | |
| pay_amount | bigint | DEFAULT 0 | 결제 금액 | |
| collected_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수집 시각 | |

### 관계
- `account` + `stat_date` + `hour_of_day` + `rank_position` UNIQUE 복합키
- smartstore_realtime_hourly와 같은 시간 기준으로 조인 가능

### 비즈니스 맥락
- 시간대별 인기 상품 실시간 모니터링
- 특정 시간대에 어떤 상품이 가장 많이 팔리는지 분석

### 자주 쓰는 쿼리 패턴
```sql
-- 현재 시간대 인기 상품 순위
SELECT rank_position, product_name, pay_amount
FROM smartstore_realtime_products
WHERE account = %s AND stat_date = CURDATE()
  AND hour_of_day = (SELECT MAX(hour_of_day) FROM smartstore_realtime_products
                     WHERE account = %s AND stat_date = CURDATE())
ORDER BY rank_position;
```

---

## brandstore_realtime_daily (브랜드스토어 실시간 현황)

summary: 네이버 브랜드스토어 실시간 현황을 JSON 형태로 저장하는 테이블. 일별/시간별 브랜드스토어 성과를 추적한다.
domain: naver
related_tables: smartstore_realtime_hourly

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | int unsigned | PK, AUTO_INCREMENT | 일련번호 | |
| realtime_json | json | | 실시간 현황 JSON 데이터 | |
| create_time | datetime | DEFAULT CURRENT_TIMESTAMP | 생성 시각 | |

### 관계
- (독립 테이블, JSON 형태로 비정규화 저장)

### 비즈니스 맥락
- 브랜드스토어 실시간 현황을 JSON으로 통째로 저장
- 00시 데이터가 전날 최종 현황, 그 외는 오늘 실시간 현황
- 기간 비교(현재 기간 vs 직전 동일 기간) 분석에 활용
- 요일별 패턴 분석(예측)에도 사용

### 자주 쓰는 쿼리 패턴
```sql
-- 최신 브랜드스토어 현황
SELECT realtime_json FROM brandstore_realtime_daily
ORDER BY id DESC LIMIT 1;

-- 요일별 예측 (같은 요일의 과거 00시 데이터)
SELECT realtime_json FROM brandstore_realtime_daily
WHERE HOUR(create_time) = '00' AND MINUTE(create_time) = '00'
  AND DAYOFWEEK(create_time) = DAYOFWEEK(NOW());
```
