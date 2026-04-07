# ToBe Networks 데이터베이스 스키마 전체 문서

> 최종 업데이트: 2026-02-28
> 분석 소스: tbnws_admin_back (Spring Boot), admin-v3 (Prisma), tbnws_admin_front (React)

---

## 시스템 개요

| 항목 | 내용 |
|------|------|
| **백엔드** | Spring Boot 3.4.8 (Java 17) |
| **ORM** | MyBatis (XML 매퍼) |
| **주 데이터베이스** | MySQL 8.0+ |
| **벡터 DB** | PostgreSQL (pgvector) - AI용 |
| **총 테이블 수** | **231개** (Prisma 기준 315 모델) |
| **데이터소스** | 9개 MySQL DB + 1개 PostgreSQL |

---

## 멀티 데이터소스 구성

| DB명 | Bean Name | 용도 |
|------|-----------|------|
| **TBNWS_ADMIN** | sqlSessionTemplate (Primary) | 메인 어드민 시스템 |
| **gtgear** | sqlSessionTemplateGTGEAR | GTGear 이커머스 |
| **WMS** | sqlSessionTemplateWMS | 창고 관리 |
| **AI_CHATBOT** | sqlSessionTemplateAI | AI 챗봇 |
| **attendance** | sqlSessionTemplateSone | 근태 관리 |
| **tbnws** | sqlSessionTemplateTbnws | 레거시 데이터 |
| **yourls** | sqlSessionTemplateYourls | URL 단축 서비스 |
| **TPT** | sqlSessionTemplateTPT | TPT 서비스 |
| **shopify** | sqlSessionTemplateShopify | Shopify 연동 |

---

## 도메인별 테이블 구조

### 1. 상품 마스터 (Product Hierarchy) - 11 테이블

```
erp_category (카테고리: G=GTGear, F=Fulfillment)
  └─ erp_brand (브랜드)
      └─ erp_goods (상품)
          └─ erp_option (옵션)
              └─ erp_product_info (상품 마스터: product_code = C-B-G-O)
```

#### erp_category
| 컬럼 | 타입 | 설명 |
|------|------|------|
| category_code | VARCHAR (PK) | 카테고리 코드 (G, F 등) |
| category_name | VARCHAR | 카테고리명 |

#### erp_brand
| 컬럼 | 타입 | 설명 |
|------|------|------|
| category_code | VARCHAR (PK, FK) | 카테고리 코드 |
| brand_code | VARCHAR (PK) | 브랜드 코드 |
| brand_name | VARCHAR | 브랜드명 |

#### erp_goods
| 컬럼 | 타입 | 설명 |
|------|------|------|
| category_code | VARCHAR (PK, FK) | 카테고리 코드 |
| brand_code | VARCHAR (PK, FK) | 브랜드 코드 |
| goods_code | VARCHAR (PK) | 상품 코드 |
| goods_name | VARCHAR | 상품명 |

#### erp_option
| 컬럼 | 타입 | 설명 |
|------|------|------|
| category_code | VARCHAR (PK, FK) | 카테고리 코드 |
| brand_code | VARCHAR (PK, FK) | 브랜드 코드 |
| goods_code | VARCHAR (PK, FK) | 상품 코드 |
| option_code | VARCHAR (PK) | 옵션 코드 |
| option_name | VARCHAR | 옵션명 |

#### erp_product_info (핵심 마스터 - 55+ 컬럼)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| product_code | VARCHAR (PK) | 상품코드 (C-B-G-O 조합) |
| category_code | VARCHAR (FK) | 카테고리 |
| brand_code | VARCHAR (FK) | 브랜드 |
| goods_code | VARCHAR (FK) | 상품 |
| option_code | VARCHAR (FK) | 옵션 |
| product_name | VARCHAR | 상품 전체명 |
| type | CHAR | G=일반, F=풀필먼트 |
| sell_status | CHAR | Y=판매중, N=미판매, D=삭제 |
| option_model | VARCHAR | 옵션 모델명 |
| series | VARCHAR | 시리즈 |
| barcode | VARCHAR | 바코드 |
| kc_num | VARCHAR | KC 인증번호 |
| width/depth/height | DECIMAL | 사이즈 |
| weight | DECIMAL | 무게 |
| units_per_box | INT | 박스당 수량 |
| loadage | INT | 적재량 |
| safe_ea | INT | 안전재고 수량 |
| msrp | DECIMAL | 권장소비자가 |
| currency | VARCHAR | 통화 |
| purchase_price | DECIMAL | 매입가 |
| actual_price | DECIMAL | 실제가 |
| actual_price_channel | DECIMAL | 채널별 실제가 |
| fixed_price | DECIMAL | 고정가 |
| duty | DECIMAL | 관세 |
| channel_price | DECIMAL | 채널가 |
| cj_code | VARCHAR | CJ택배 코드 |

#### erp_product_details
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| title | VARCHAR | 제목 |
| content | TEXT | 내용 |

#### erp_product_image
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| title | VARCHAR | 이미지 제목 |
| url | VARCHAR | 이미지 URL |

#### erp_product_components
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| components | TEXT | 구성품 (BOM) |

#### erp_product_qc
| 컬럼 | 타입 | 설명 |
|------|------|------|
| product_code | VARCHAR (PK) | 상품코드 |
| contents | TEXT | QC 정보 |

---

### 2. 재고 관리 (Inventory & Stock) - 12 테이블

```
★ 창고 체계 (wms_warehouse)
  ┌─────────────────────────────────────────────────────┐
  │  GT (지티기어 창고)    → G-, P-, K-, Y- prefix       │
  │  GJ (CJ 풀필먼트 창고) → F- prefix                   │
  │                                                     │
  │  ⚠️ DB코드 'GJ' = 실제는 CJ 풀필먼트 (곤지암)        │
  │  ⚠️ 네이버 물량 대부분 CJ에서 3PL 직접 출고           │
  │  ⚠️ 동일 상품 = prefix만 다름:                       │
  │     G-0029-0108-0040 (GT) ↔ F-0029-0108-0040 (CJ)  │
  │  ⚠️ 재고 분석 시 G-코드 + F-코드 반드시 합산!         │
  └─────────────────────────────────────────────────────┘

erp_order (발주)
  └─ erp_order_item (발주 품목)
      └─ erp_stock (개별 물리 유닛 추적)
          ├─ import_date (입고일)
          ├─ warehouse_tag (GT=지티창고, GJ=CJ풀필먼트)
          ├─ export_date (출고일)
          └─ export_partner_code (출고처)
```

#### wms_warehouse (창고 마스터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| warehouse_tag | VARCHAR (PK) | 창고코드: GT=지티기어, GJ=CJ풀필먼트 |
| warehouse_name | VARCHAR | 창고명 |
| address | VARCHAR | 주소 |
| manager | VARCHAR | 담당자 |

**데이터:** GT(지티기어 창고, 142,040건), GJ(곤지암 풀필먼트 창고=CJ, 41,965건)

#### erp_stock (핵심 재고 테이블)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| stock_seq | INT (PK, AUTO) | 재고 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| item_seq | INT (FK) | 발주 품목 순번 |
| import_date | DATE | 입고일 |
| export_date | DATE | 출고일 |
| import_partner_code | VARCHAR (FK) | 입고 파트너 |
| export_partner_code | VARCHAR (FK) | 출고 파트너 |
| warehouse_tag | VARCHAR | ★ 창고: GT(지티) / GJ(CJ풀필) |
| location_code | VARCHAR | 로케이션 코드 |
| import_grade | VARCHAR | 입고 등급 |
| return_flag | CHAR | 반품 여부 (Y/N) |

**export_partner_code 주요 값:**
- `ptn0000` = 기타
- `ptn0001` = A/S
- `ptn0076` = 리퍼브
- `ptn0100` = 재고조정
- `ptn0103` = 마케팅
- `ptn0124` = 풀필먼트

#### erp_stock_adjustment
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK) | 순번 |
| product_code | VARCHAR | 상품코드 |
| adjusted_qty | INT | 조정 수량 |
| reason | VARCHAR | 사유 |

#### erp_stock_summary (일별 재고 스냅샷 — 261,207행)
> 매일 갱신되는 재고·판매·입고 집계 테이블. **발주 분석의 핵심 데이터 소스**
> product_code별 일별 스냅샷 (summary_date)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| product_code | VARCHAR (PK) | 상품코드 (= erp_product_info.product_code) |
| category_code | VARCHAR | 카테고리 (D/F/G/K/P/Y/Z) |
| brand_code | VARCHAR | 브랜드 코드 |
| brand_name | VARCHAR | 브랜드명 |
| goods_code | VARCHAR | 상품 코드 |
| goods_name | VARCHAR | 상품명 |
| option_code | VARCHAR | 옵션 코드 |
| option_name | VARCHAR | 옵션명 |
| cj_code | VARCHAR | CJ택배 코드 |
| **ea** | **INT** | **★ 현재 재고 수량** |
| **be_due_ea** | **INT** | **★ 입고 예정 수량** |
| schedule_ea | INT | 스케줄 수량 |
| location_schedule_ea | INT | 로케이션 스케줄 수량 |
| pre_order_ea | INT | 선주문 수량 |
| **one_week_sales_ea** | **INT** | **★ 최근 1주 판매 수량** |
| **one_month_sales_ea** | **INT** | **★ 최근 1개월 판매 수량** |
| **three_month_sales_ea** | **INT** | **★ 최근 3개월 판매 수량** |
| one_week_sales_price | DECIMAL | 1주 판매 금액 |
| one_month_sales_price | DECIMAL | 1개월 판매 금액 |
| three_month_sales_price | DECIMAL | 3개월 판매 금액 |
| stock_price | DECIMAL | 재고 금액 |
| **summary_date** | **DATE (PK)** | **★ 스냅샷 기준일** |
| updated_at | TIMESTAMP | 갱신일시 |

**사용 패턴:** `WHERE summary_date = (SELECT MAX(summary_date) FROM erp_stock_summary)` — 최신 날짜 조회

#### erp_stock_realtime (실시간 재고 — 9,451행)
> erp_stock_summary의 실시간 버전. summary_date 없이 현재 상태만 유지

| 컬럼 | 타입 | 설명 |
|------|------|------|
| productCode | VARCHAR (PK) | 상품코드 |
| ea | INT | ★ 현재 재고 |
| beDueEa | INT | ★ 입고 예정 |
| scheduleEa | INT | 스케줄 수량 |
| preOrderEa | INT | 선주문 수량 |
| stockPrice | DECIMAL | 재고 금액 |
| latestBuyingPrice | DECIMAL | ★ 최신 매입가 (KRW) |
| oneWeekSalesEa | INT | 1주 판매량 |
| oneMonthSalesEa | INT | 1개월 판매량 |
| threeMonthSalesEa | INT | 3개월 판매량 |
| oneWeekSalesPrice | DECIMAL | 1주 판매금액 |
| oneMonthSalesPrice | DECIMAL | 1개월 판매금액 |
| threeMonthSalesPrice | DECIMAL | 3개월 판매금액 |
| updatedAt | TIMESTAMP | 갱신일시 |

**⚠️ 주의:** camelCase 컬럼명 (summary와 다름). `productCode` = `erp_product_info.product_code`
**⚠️ 창고 구분:** G-코드(GT창고) + F-코드(CJ풀필먼트) 별도 행 → 합산 필요!
**합산 패턴:** `LEFT JOIN erp_stock_realtime f ON CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode`

#### erp_refurb_stock (리퍼브 재고)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK) | 순번 |
| product_code | VARCHAR | 상품코드 |
| import_date | DATE | 입고일 |
| sale_flag | CHAR | 판매 여부 |
| refurb_status | VARCHAR | 리퍼브 상태 |

---

### 3. 발주 & 구매 (Purchasing & Orders) - 16 테이블

#### ★ erp_purchase_order (키크론 발주서 — 535행)
> 키크론 본사 발주 기록. PO번호 형식: `K260206` (K + 날짜)
> erp_order와 별개 — erp_order는 입고/물류 단위, 이 테이블은 발주 단위

| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | BIGINT (PK, AUTO) | 순번 |
| register_date | DATETIME | ★ 발주 등록일 |
| po_title | VARCHAR(255) | 발주 제목 (= PO번호) |
| po_number | VARCHAR(255) | PO번호 (예: K260206) |
| partner_code | VARCHAR(20) | 파트너코드 (ptn0039 = 키크론 본사) |
| description | VARCHAR(255) | 설명 |
| member_id | VARCHAR(10) | 등록자 ID |
| brand_code | VARCHAR(20) | 브랜드코드 (0029 = Keychron) |
| **product_code** | **VARCHAR(255)** | **★ G-code (= erp_product_info.product_code)** |
| **quantity** | **DECIMAL(10,2)** | **★ 발주 수량** |
| due_date | DATETIME | 입고 예정일 (대부분 NULL) |
| order_seq | BIGINT | 연결 발주 순번 |
| created_at | TIMESTAMP | 생성일시 |
| modified_at | TIMESTAMP | 수정일시 |

**월별 발주 현황 (최근):**
- 2026-02: 52건, 17,669개
- 2026-01: 86건, 11,191개
- 2025-11: 150건, 29,407개

**⚠️ status 컬럼 없음** — 발주 상태 관리는 별도 시스템

#### erp_order (발주 마스터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| order_seq | INT (PK, AUTO) | 발주 순번 |
| order_title | VARCHAR | 발주 제목 |
| partner_code | VARCHAR (FK) | 파트너(공급처) 코드 |
| order_date | DATE | 발주일 |
| import_date | DATE | 입고 예정일 |
| export_date | DATE | 출고일 |
| dep_flag | CHAR | 출항 여부 |
| arr_flag | CHAR | 도착 여부 |
| deposit_price | DECIMAL | 선급금 |
| balance_price | DECIMAL | 잔금 |
| delivery_price | DECIMAL | 배송비 |
| transport | VARCHAR | 운송수단 |
| vat_flag | CHAR | 부가세 여부 |
| etc_price | DECIMAL | 기타 비용 |
| parent_order_seq | INT (FK) | 상위 발주 |

#### erp_order_item (발주 품목)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| item_seq | INT (PK, AUTO) | 품목 순번 |
| order_seq | INT (FK) | 발주 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| ea | INT | 수량 |
| buying_price | DECIMAL | 매입가 |
| currency | VARCHAR | 통화 |
| cur2krw | DECIMAL | 환율 |
| import_status | VARCHAR | 입고 상태 |
| po_seq | INT | PO 순번 |

#### erp_order_payment_history
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK) | 순번 |
| order_seq | INT (FK) | 발주 순번 |
| payment_date | DATE | 결제일 |
| amount | DECIMAL | 금액 |
| payment_type | VARCHAR | 결제 유형 |

#### erp_partner (파트너/거래처 마스터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| partner_code | VARCHAR (PK) | 파트너 코드 |
| partner_name | VARCHAR | 파트너명 |
| partner_type | VARCHAR | IMPORT/EXPORT/BOTH |
| partner_target | VARCHAR | 대상 |
| is_use_yn | CHAR | 사용 여부 |
| gtgear_url | VARCHAR | GTGear URL |
| sabangnet_mall_name | VARCHAR | 사방넷 몰명 |

---

### 4. B2B 수주 관리 (B2B Receiving Orders) - 10 테이블

#### erp_order_recv (B2B 수주 마스터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| recv_seq | INT (PK, AUTO) | 수주 순번 |
| biz_code | VARCHAR (FK) | 거래처 코드 |
| recv_muid | VARCHAR | 담당자 ID |
| tax_invoice_seq | INT (FK) | 세금계산서 순번 |
| export_type | VARCHAR | 출고 유형 |
| due_date | DATE | 납기일 |
| pay_date | DATE | 결제일 |
| pay_muid | VARCHAR | 결제 담당자 |
| export_schedule_date | DATE | 출고 예정일 |

#### erp_order_recv_biz (B2B 거래처)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| biz_code | VARCHAR (PK) | 거래처 코드 |
| biz_name | VARCHAR | 상호명 |
| biz_ceo | VARCHAR | 대표자 |
| biz_num | VARCHAR | 사업자등록번호 |
| biz_email | VARCHAR | 이메일 |
| biz_address | VARCHAR | 주소 |
| biz_type | VARCHAR | 업태 |
| biz_item | VARCHAR | 종목 |
| branch_flag | CHAR | 지점 여부 |

#### erp_order_recv_item (B2B 수주 품목)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| recv_item_seq | INT (PK, AUTO) | 품목 순번 |
| recv_seq | INT (FK) | 수주 순번 |
| product_code | VARCHAR | 상품코드 |
| ea | INT | 수량 |
| export_price | DECIMAL | 출고가 |
| order_num | VARCHAR | 주문번호 |
| transport_num | VARCHAR | 운송장번호 |
| courier | VARCHAR | 택배사 |
| invoice | VARCHAR | 송장번호 |

---

### 5. 출고 & 풀필먼트 (Export & Fulfillment) - 11 테이블

#### erp_export_request
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| title | VARCHAR | 제목 |
| description | TEXT | 내용 |
| request_date | DATE | 요청일 |
| request_member_id | VARCHAR | 요청자 |

#### erp_export_schedule (출고 스케줄)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| export_schedule_seq | INT (PK, AUTO) | 순번 |
| title | VARCHAR | 제목 |
| schedule_description | TEXT | 설명 |
| request_date | DATE | 요청일 |
| complete_date | DATE | 완료일 |

#### erp_export_schedule_item
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| export_schedule_seq | INT (FK) | 출고 스케줄 |
| product_code | VARCHAR (FK) | 상품코드 |
| ea | INT | 수량 |
| export_flag | CHAR | 출고 여부 |

---

### 6. 판매 & 마켓플레이스 연동 (Sales) - 8+ 테이블

#### tbnws_sabangnet_order (사방넷 주문 - 핵심 판매 데이터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| idx | VARCHAR (PK) | 주문 인덱스 |
| order_date | DATE | 주문일 |
| order_status | VARCHAR | 주문상태 (주문접수/배송중/배송완료/취소) |
| model_no | VARCHAR | G-code (상품코드) |
| p_ea | INT | 수량 |
| user_name | VARCHAR | 주문자명 |
| receive_address | VARCHAR | 배송지 |
| pay_cost | DECIMAL | 결제금액 |
| delv_cost | DECIMAL | 배송비 |
| mall_id | VARCHAR | 쇼핑몰 ID |

#### erp_coupang_order (쿠팡 주문)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| order_id | VARCHAR (PK) | 주문 ID |
| product_code | VARCHAR (FK) | 상품코드 |
| order_date | DATE | 주문일 |
| order_status | VARCHAR | 주문상태 |
| quantity | INT | 수량 |
| price | DECIMAL | 가격 |

#### erp_coupang (쿠팡 상품 매핑)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| coupang_seq | INT (PK, AUTO) | 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| coupang_sku_id | VARCHAR | 쿠팡 SKU |
| sku_name | VARCHAR | SKU명 |
| sku_delivery_price | DECIMAL | 배송비 |

---

### 7. 네이버 연동 (Naver Integration) - 11+ 테이블

#### ★ naver_smartstore_orders (실거래가 1순위 — 55,398행)
> 네이버 스마트스토어 주문 데이터. **실거래가 조회 최우선 소스**
> 기간: 2023-12-21 ~ 현재 (실시간 동기화)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| account | VARCHAR(50) (PK) | 스토어 구분 (keychron: 53,939건, gtgear: 1,459건) |
| product_order_id | VARCHAR(100) (PK) | 상품주문번호 |
| order_id | VARCHAR(100) | 주문번호 |
| product_no | VARCHAR(50) | ★ 네이버 상품번호 → md_keychron_sheet.naver_product_code |
| product_name | VARCHAR(500) | 상품명 |
| quantity | INT | 수량 |
| total_price | BIGINT | ★ 총 결제금액 (실거래가 = total_price/quantity) |
| product_order_status | VARCHAR(50) | 주문상태 |
| order_date | DATETIME | 주문일시 |
| pay_date | DATETIME | 결제일시 |
| claim_type | VARCHAR(50) | 클레임 유형 |
| created_at | TIMESTAMP | 생성일시 |

**product_order_status 분포:** PURCHASE_DECIDED(47,109) / CANCELED(4,405) / DELIVERED(1,877) / RETURNED(1,682) / EXCHANGED(256) / DELIVERING(38) / PAYED(31)

#### ★ naver_commerce_order (정산 상세 — 19,332행)
> 네이버 커머스 API 56개 필드. G-code 직접 매핑 가능
> 기간: 2025-01-13 ~ 2026-01-02 (⚠️ 동기화 중단)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | BIGINT (PK) | 순번 |
| orderId | VARCHAR(30) | 주문번호 |
| productOrderId | VARCHAR(50) | 상품주문번호 |
| orderDate | DATETIME | 주문일 |
| paymentDate | DATETIME | 결제일 |
| productOrderStatus | VARCHAR(50) | 주문상태 |
| sellerProductCode | VARCHAR(50) | ★ G-code 직접 매핑 (= erp_product_info.goods_code) |
| productId | VARCHAR(50) | 네이버 상품ID |
| productName | VARCHAR(255) | 상품명 |
| productOption | VARCHAR(255) | 옵션명 |
| quantity | INT | 수량 |
| unitPrice | INT | 개별 표시가 (할인 전) |
| totalPaymentAmount | INT | ★ 실결제 총액 |
| totalProductAmount | INT | 상품 총액 (할인 전) |
| expectedSettlementAmount | INT | ★ 정산 예정액 (수수료 차감 후) |
| paymentCommission | INT | 결제 수수료 |
| productDiscountAmount | INT | 상품 할인 |
| orderDiscountAmount | INT | 주문 할인 |
| sellerBurdenDiscountAmount | INT | 판매자 부담 할인 |
| generalPaymentAmount | INT | 일반 결제 |
| naverMileagePaymentAmount | INT | 마일리지 결제 |
| deliveryFeeAmount | INT | 배송비 |
| brandCode | VARCHAR(20) | 브랜드 코드 |
| ... | ... | 기타 30+ 컬럼 (주소, 배송, 클레임 등) |

**G-code 매핑률:** 전체 19,332건 중 G-code 형식(G-): 11,895건 (61.5%)

#### ★ md_keychron_sheet (네이버↔ERP 브릿지 매핑 — 925행)
> 네이버 상품코드 ↔ 내부 G-code 매핑 테이블. **3-way JOIN의 핵심**
> ⚠️ 1:N 관계: 하나의 naver_product_code에 최대 39개 옵션

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK) | 순번 |
| naver_product_code | VARCHAR(100) | ★ 네이버 상품번호 (= naver_smartstore_orders.product_no) |
| tobe_product_code | VARCHAR(100) | ★ G-code (= erp_product_info.product_code) |
| product_name | VARCHAR(500) | 상품명 |
| option_name | VARCHAR(255) | 옵션명 |
| internal_management_code | VARCHAR(100) | 내부 관리코드 |
| group_product_code | VARCHAR(100) | 그룹 상품코드 |
| naver_list_price | DECIMAL(10,0) | 네이버 정가 |
| naver_sale_price | DECIMAL(10,0) | 네이버 판매가 |
| tobe_list_price | DECIMAL(10,0) | 투비 정가 |
| tobe_sale_price | DECIMAL(10,0) | 투비 판매가 |
| brand_name | VARCHAR(100) | 브랜드 |
| product_line | VARCHAR(255) | 제품 라인 |
| product_option | VARCHAR(500) | 제품 옵션 |
| product_category | VARCHAR(50) | 카테고리 |

**통계:** 331개 유니크 네이버코드, 681개 유니크 G-code, 평균 중복도 2.8배

#### md_keychron_sheet_attach (복합 옵션 매핑 — 649행)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK) | 순번 |
| product_code | VARCHAR(50) | 상품코드 |
| option_code | VARCHAR(50) | 옵션코드 |
| tobe_code | VARCHAR(50) | G-code |
| product_name | VARCHAR(500) | 상품명 |
| option_name | VARCHAR(500) | 옵션명 |
| brand | VARCHAR(100) | 브랜드 |
| tobe_sale_price | INT | 투비 판매가 |

#### naver_ranking_tracker_list (랭킹 추적)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| 키워드별 순위 | - | 네이버 쇼핑 순위 추적 |

#### naver_store_review_crawler (리뷰 크롤링)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| 리뷰 데이터 | - | 네이버 스토어 리뷰 수집 |

#### naver_search_ad (검색광고)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| 광고 데이터 | - | 네이버 SA 검색광고 데이터 |

---

### 8. 재무 & 회계 (Finance & Accounting) - 7 테이블

#### finance_tax_invoice (세금계산서)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| biz_code | VARCHAR (FK) | 거래처 코드 |
| invoice_date | DATE | 발행일 |
| total_amount | DECIMAL | 총 금액 |
| approval_flag | CHAR | 승인 여부 |

#### finance_tax_invoice_detail
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| tax_invoice_seq | INT (FK) | 세금계산서 순번 |
| product_code | VARCHAR (FK) | 상품코드 |
| quantity | INT | 수량 |
| unit_price | DECIMAL | 단가 |

#### finance_estimate (견적서)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| biz_code | VARCHAR (FK) | 거래처 |
| estimate_date | DATE | 견적일 |
| total_amount | DECIMAL | 총 금액 |

#### finance_etc_remit (기타 송금/경비)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| finance_etc_seq | INT (PK, AUTO) | 순번 |
| muid | VARCHAR (FK) | 요청자 |
| type | VARCHAR | 비용/경조사 |
| approval_depth | INT | 승인 단계 |
| title | VARCHAR | 제목 |
| amount | DECIMAL | 금액 |
| bank | VARCHAR | 은행 |
| account_number | VARCHAR | 계좌번호 |
| approval_flag | CHAR | 승인 여부 (Y/N/빈값) |
| remit_flag | CHAR | 송금 여부 |
| cancel_flag | CHAR | 취소 여부 |

#### finance_fixtures_remit (구매 요청/고정자산)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| member_id | VARCHAR (FK) | 요청자 |
| fixture_type | VARCHAR | 자산 유형 |
| amount | DECIMAL | 금액 |
| approval_flag | CHAR | 승인 여부 |

#### currency_exchange_rate (환율)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| currency_code | VARCHAR | 통화코드 |
| exchange_rate | DECIMAL | 환율 |
| rate_date | DATE | 기준일 |

---

### 9. 인사 관리 (HR) - 12 테이블

#### member (직원 마스터)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO) | ID |
| uid | VARCHAR (UNIQUE) | 사용자 ID |
| name | VARCHAR | 이름 |
| email | VARCHAR | 이메일 |
| department_id | INT (FK) | 부서 |
| part_id | INT (FK) | 파트 |
| position_id | INT (FK) | 직급 |
| join_date | DATE | 입사일 |
| phone | VARCHAR | 연락처 |

#### department
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO) | ID |
| department_name | VARCHAR | 부서명 (지원, 마케팅, 운영, 물류) |

#### vacation (연차 현황)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO) | ID |
| member_id | INT (FK) | 직원 |
| year | INT | 연도 |
| vacation_days | DECIMAL | 총 연차 |
| use_days | DECIMAL | 사용 연차 |
| add_days | DECIMAL | 추가 연차 |
| overuse_days | DECIMAL | 초과 사용 |

#### vacation_detail (휴가 신청/승인)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO) | ID |
| request_m_id | INT (FK) | 신청자 |
| start_date | DATE | 시작일 |
| end_date | DATE | 종료일 |
| vacation_type | VARCHAR | 연차/오전반차/오후반차/경조사 |
| days | DECIMAL | 일수 |
| approval_flag | CHAR | 승인 여부 |
| approval_m_id | INT (FK) | 승인자 |

#### overtime (시간외 근무)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO) | ID |
| member_id | INT (FK) | 직원 |
| overtime_date | DATE | 근무일 |
| hours | DECIMAL | 시간 |
| approval_flag | CHAR | 승인 여부 |

#### welfare (복리후생)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| member_id | INT (FK) | 직원 |
| type | VARCHAR | 유형 |
| title | VARCHAR | 제목 |
| purpose | VARCHAR | 용도 |
| price | DECIMAL | 금액 |
| approval_flag | CHAR | 승인 여부 |
| remit_flag | CHAR | 송금 여부 |

---

### 10. CRM & CS 고객 관리 - 20+ 테이블

```
CS 데이터 흐름:
┌─────────────────────────────────────────────────────────────┐
│  고객 접점 채널                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │1:1 문의   │  │상품 QnA  │  │반품/교환  │  │AS 접수   │   │
│  │inquiries │  │qna       │  │claims    │  │crm_as    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │         │
│       ▼              ▼              ▼              ▼         │
│  naver_store_   naver_         naver_          crm_as       │
│  customer_      smartstore_    smartstore_     crm_as_      │
│  inquiries      qna            orders          product     │
│                                (claim_type)                 │
│                                                             │
│  ⚠️ 클레임 = naver_smartstore_orders.claim_type로 추적     │
│  ⚠️ AS = crm_as → crm_as_product (진단/해결 기록)          │
│  ⚠️ 문의 = naver_store_customer_inquiries (답변율 추적)     │
└─────────────────────────────────────────────────────────────┘
```

#### ★ naver_store_customer_inquiries (1:1 고객문의 — CS 핵심)
> 네이버 스마트스토어 1:1 문의. 답변율·카테고리별 분석의 핵심 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| inquiry_no | BIGINT (PK) | 문의번호 |
| brand_channel | VARCHAR(20) | 스토어 구분 (KEYCHRON, GTGEAR) |
| internal_product_code | VARCHAR(30) | 내부 상품코드 |
| category | VARCHAR(50) | ★ 문의 카테고리 (상품/교환/반품/배송/기타/환불) |
| title | VARCHAR(500) | 문의 제목 |
| inquiry_content | TEXT | 문의 내용 |
| inquiry_registration_date_time | DATETIME | ★ 문의 등록일시 |
| customer_id | VARCHAR(100) | 고객 ID |
| customer_name | VARCHAR(100) | 고객명 |
| order_id | VARCHAR(100) | 주문번호 |
| product_no | VARCHAR(100) | 네이버 상품번호 |
| product_order_id_list | VARCHAR(500) | 상품주문번호 목록 |
| product_name | VARCHAR(500) | 상품명 |
| product_order_option | VARCHAR(500) | 주문 옵션 |
| answer_content | TEXT | 답변 내용 |
| answer_template_no | INT | 답변 템플릿 번호 |
| answer_registration_date_time | DATETIME | 답변 등록일시 |
| answered | TINYINT(1) | ★ 답변 여부 (0=미답변, 1=답변완료) |
| ai_answer_generated | TINYINT(1) | AI 답변 생성 여부 |
| cs_reviewed | TINYINT(1) | CS 검토 여부 |
| processing_status | VARCHAR(50) | 처리 상태 (pending 등) |
| agent_answer | TEXT | AI 에이전트 답변 |

**카테고리 분포 (2월 기준):** 상품(37) > 교환(32) > 반품(29) > 배송(17) > 기타(15) > 환불(13)

#### ★ crm_as (A/S 접수 마스터)
> CRM AS 접수 건. 처리 상태는 start_date/complete_date 조합으로 판별

| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT UNSIGNED (PK, AUTO) | AS 순번 |
| customer_name | VARCHAR(30) | 고객명 |
| customer_phone | VARCHAR(20) | 연락처 |
| customer_email | VARCHAR(255) | 이메일 |
| customer_address | VARCHAR(255) | 주소 |
| customer_postal_code | VARCHAR(10) | 우편번호 |
| customer_environment | VARCHAR(100) | 사용 환경 |
| description | TEXT | 증상 설명 (⚠️ 최근 건은 대부분 미입력) |
| charge_member_id | INT UNSIGNED (FK) | 담당자 ID |
| sub_charge_member_id | INT UNSIGNED | 부담당자 ID |
| regist_date | DATETIME | ★ 접수일 |
| start_date | DATETIME | ★ 처리 시작일 (NULL=대기) |
| complete_date | DATETIME | ★ 완료일 (NULL=미완료) |
| feedback_flag | CHAR(1) | 피드백 여부 (Y/N) |

**처리 상태 판별:**
- `complete_date IS NOT NULL` → 완료
- `complete_date IS NULL AND start_date IS NOT NULL` → 진행중
- `complete_date IS NULL AND start_date IS NULL` → 대기(PENDING)

#### ★ crm_as_product (A/S 제품 상세 — 진단/해결 기록)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT UNSIGNED (PK, AUTO) | 순번 |
| as_seq | INT UNSIGNED (FK) | ★ crm_as.seq 연결 |
| product_code | VARCHAR(30) | ★ 상품코드 (erp_product_info와 JOIN 가능) |
| quantity | INT UNSIGNED | 수량 (기본값 1) |
| process_type | VARCHAR(50) | ★ 처리 유형 (종결/회수중/입고 완료/수리 완료/접수 완료/결제 대기) |
| resolution_type | VARCHAR(50) | ★ 해결 유형 (A/S/교환/반품/선조치) |
| retrieve_courier | CHAR(2) | 회수 택배사 |
| retrieve_invoice | VARCHAR(50) | 회수 송장번호 |
| retrieve_memo | TEXT | 회수 메모 |
| import_date | DATETIME | 입고일 |
| expected_diagnosis_days | TINYINT UNSIGNED | 예상 진단 일수 (기본 3) |
| customer_serial_num | VARCHAR(50) | 고객 시리얼 |
| serial_num | VARCHAR(50) | 시리얼 번호 |
| option_model | VARCHAR(100) | 옵션 모델 |
| stock_seq | INT UNSIGNED (FK) | 재고 순번 |
| repair_period | DATETIME | 수리 기간 |
| is_free_repair | CHAR(1) | 무상 수리 여부 (Y/N) |
| user_cost | INT UNSIGNED | 고객 부담 비용 |
| company_cost | INT UNSIGNED | 회사 부담 비용 |
| diagnose | TEXT | ★ 진단 내용 (고장 원인 기록) |
| reference_content | TEXT | 참조 내용 |
| return_flag | CHAR(1) | 반품 여부 |
| notify_phase | INT UNSIGNED | 알림 단계 |

**빈출 진단 패턴:** 메인보드 불량, 특정키 입력 씹힘, 펌웨어 이슈, 스위치/토글 이탈, PCB 파손

#### naver_smartstore_qna (상품 Q&A)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| account | VARCHAR(50) (PK) | 스토어 구분 (keychron/gtgear) |
| qna_id | VARCHAR(100) (PK) | Q&A ID |
| product_no | VARCHAR(50) | 네이버 상품번호 |
| product_name | VARCHAR(500) | 상품명 |
| question_content | TEXT | 질문 내용 |
| answer_content | TEXT | 답변 내용 |
| question_date | DATETIME | 질문 등록일 |
| answer_date | DATETIME | 답변 등록일 |
| answered | TINYINT(1) | ★ 답변 여부 (0=미답변, 1=답변완료) |

#### crm_return_management (반품 관리)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT UNSIGNED (PK, AUTO) | 순번 |
| as_seq | INT UNSIGNED (FK) | crm_as.seq 연결 |
| customer_name | VARCHAR(30) | 고객명 |
| customer_phone | VARCHAR(30) | 연락처 |
| customer_address | TEXT | 주소 |
| courier | CHAR(2) | 택배사 |
| original_invoice | VARCHAR(255) | 원 송장 |
| return_invoice | VARCHAR(255) | 반품 송장 |
| requested_flag | CHAR(1) | 요청 여부 (Y/N) |
| description | TEXT | 설명 |
| member_id | INT UNSIGNED (FK) | 담당자 |
| regist_date | DATETIME | 등록일 |

#### tbnws_sabangnet_claim (사방넷 클레임)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | BIGINT (PK, AUTO) | 순번 |
| idx | VARCHAR(100) | 클레임 인덱스 |
| order_id | VARCHAR(100) | 주문번호 |
| mall_id | VARCHAR(100) | 쇼핑몰 |
| order_status | VARCHAR(20) | 주문 상태 |
| clame_status_gubun | VARCHAR(20) | ★ 클레임 유형 |
| clame_content | TEXT | 클레임 내용 |
| clame_ins_date | DATE | 클레임 접수일 |
| product_name | VARCHAR(255) | 상품명 |
| sale_cnt | INT | 수량 |
| sale_cost | INT UNSIGNED | 판매가 |

#### naver_store_inquiry_issue (문의 이슈 분류)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT UNSIGNED (PK, AUTO) | 순번 |
| inquiry_no | BIGINT | 문의번호 (inquiries FK) |
| issue_type | CHAR(1) | 이슈 유형 |
| status | CHAR(1) | 상태 (P=대기) |

#### 기타 CRM 테이블
| 테이블 | 용도 |
|--------|------|
| customer | 고객 마스터 |
| crm_as_export | A/S 출고 |
| crm_as_history | A/S 이력 |
| crm_as_memo | A/S 메모 |
| crm_as_notify | A/S 알림 |
| crm_as_alimtalk_template | A/S 알림톡 템플릿 |
| customer_refund | 환불 |
| crm_refund | 환불 관리 |
| crm_return_product | 반품 제품 |
| crm_call_info | 전화 상담 |
| crm_call_memo | 상담 메모 |
| crm_report | CRM 리포트 |
| vip_history | VIP 고객 이력 |

#### naver_smartstore_orders 클레임 분석 (claim_type 활용)
> 주문 테이블에서 `claim_type` 필드로 취소/반품/교환 추적

**claim_type 값:** CANCEL(고객취소), RETURN(반품), EXCHANGE(교환), ADMIN_CANCEL(관리자취소)
**product_order_status 분포:** PURCHASE_DECIDED(47,119) / CANCELED(4,405) / DELIVERED(1,869) / RETURNED(1,682) / EXCHANGED(256)

#### 네이버 리뷰
| 테이블 | 용도 |
|--------|------|
| naver_store_review_item | 리뷰 |
| naver_store_review_photo | 리뷰 사진 |
| naver_store_review_crawler | 리뷰 크롤러 |

---

### 11. 채널/가격 관리 - 5 테이블

#### erp_channel_list
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| channel_code | VARCHAR | 채널 코드 |
| partner_code | VARCHAR (FK) | 파트너 |
| channel_name | VARCHAR | 채널명 |

#### erp_channel_price
| 컬럼 | 타입 | 설명 |
|------|------|------|
| product_code | VARCHAR (PK) | 상품코드 |
| partner_code | VARCHAR (PK) | 파트너 |
| channel_code | VARCHAR | 채널 |
| channel_price | DECIMAL | 채널가 |

#### erp_target_amount (매출 목표)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| seq | INT (PK, AUTO) | 순번 |
| target_amount | DECIMAL | 목표 금액 |
| period | VARCHAR | 기간 |
| member_id | VARCHAR | 담당자 |

---

### 12. 매출 분석 (Sales Analytics) - 5 테이블

| 테이블 | 용도 |
|--------|------|
| erp_annual_sales | 연간 매출 요약 |
| erp_annual_model_sales | 모델별 연간 매출 |
| erp_annual_acc_layout_sales | 레이아웃/액세서리별 매출 |
| erp_customer_month_targert_sales | 고객별 월간 매출 목표 |
| erp_discounted_products | 할인 상품 |

---

### 13. 프로모션 & 이벤트 - 17 테이블

#### 사은품 관리
| 테이블 | 용도 |
|--------|------|
| erp_free_gift_event | 사은품 이벤트 마스터 |
| erp_free_gift_target | 사은품 대상 |
| erp_free_gift | 사은품 상품 |
| coupon_list | 쿠폰 |
| erp_discounted_products | 할인 |

#### 이벤트 관리
| 테이블 | 용도 |
|--------|------|
| event_prelaunching | 프리런칭 이벤트 |
| event_prelaunching_applicant | 신청자 |
| event_giveaway | 기브어웨이 |
| event_giveaway_applicant | 기브어웨이 참가자 |
| event_subscription | 이벤트 구독 |

---

### 14. 메시징 & 알림 - 8 테이블

| 테이블 | 용도 |
|--------|------|
| alimtalk_template | 카카오 알림톡 템플릿 |
| alimtalk_template_button | 알림톡 버튼 |
| sms_template | SMS 템플릿 |
| message_template | 메시지 템플릿 |
| message_list | 메시지 발송 이력 |
| message_adcall | ACall 발송 |
| notification | 알림 |

---

### 15. 광고 & 마케팅 (Ad & Marketing) - 10+ 테이블

```
광고 데이터 체계:
┌──────────────────────────────────────────────────────────────────────┐
│  네이버 광고 시스템                                                    │
│                                                                      │
│  ┌─── SA (검색광고) ───┐    ┌─── GFA (디스플레이) ───┐               │
│  │ campaign_daily      │    │ gfa_campaign_daily     │               │
│  │ keyword_daily       │    │ gfa_adgroup_daily      │               │
│  │ adgroup_daily       │    │ gfa_creative_daily     │               │
│  └─────────┬───────────┘    │ gfa_budget             │               │
│            │                └─────────┬──────────────┘               │
│            │                          │                               │
│            └────────┬─────────────────┘                               │
│                     ▼                                                 │
│            campaign_brand_map                                        │
│            (campaign → brand: keychron/gtgear)                       │
│                     │                                                 │
│                     ▼                                                 │
│            naver_ad_bizmoney (잔액 추적)                              │
│                                                                      │
│  ┌─── 매출 연동 ────────────────────────────┐                       │
│  │ naver_smartstore_order_daily              │                       │
│  │ (stat_date, account, revenue, order_cnt)  │                       │
│  └───────────────────────────────────────────┘                       │
│                                                                      │
│  ⚠️ SA 계정: keychron, gtgear                                       │
│  ⚠️ GFA 캠페인명에 퍼널 단계 포함: TOF/MOF/BOF/ADVoost/리타게팅     │
│  ⚠️ 실시간 분석: tbnws-ad 프로젝트 (tbe.kr:8087)                    │
└──────────────────────────────────────────────────────────────────────┘
```

#### ★ naver_ad_campaign_daily (SA 검색광고 캠페인 일별)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK, AUTO) | 순번 |
| account | VARCHAR(20) | ★ 광고 계정 (keychron/gtgear) |
| customer_id | VARCHAR(20) | 네이버 광고 고객 ID |
| campaign_id | VARCHAR(100) | 캠페인 ID |
| campaign_name | VARCHAR(200) | 캠페인명 |
| stat_date | DATE | ★ 통계 날짜 |
| imp_cnt | INT | 노출수 |
| clk_cnt | INT | ★ 클릭수 |
| sales_amt | BIGINT | ★ 광고비 (소진액) |
| ccnt | INT | ★ 전환수 |
| conv_amt | BIGINT | ★ 전환매출 |
| ctr | DECIMAL(8,4) | 클릭률 |
| cpc | INT | 클릭당 비용 |
| ror | DECIMAL(12,2) | ROAS (conv_amt/sales_amt × 100) |

**ROAS 계산:** `ROUND(SUM(conv_amt)/NULLIF(SUM(sales_amt),0)*100, 0)`

#### ★ naver_ad_keyword_daily (SA 키워드 일별)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK, AUTO) | 순번 |
| account | VARCHAR(20) | 광고 계정 |
| keyword_id | VARCHAR(100) | 키워드 ID |
| keyword | VARCHAR(200) | ★ 키워드 텍스트 |
| adgroup_id | VARCHAR(100) | 광고그룹 ID |
| adgroup_name | VARCHAR(200) | 광고그룹명 |
| stat_date | DATE | 통계 날짜 |
| imp_cnt | INT | 노출수 |
| clk_cnt | INT | 클릭수 |
| sales_amt | BIGINT | ★ 광고비 |
| ccnt | INT | ★ 전환수 (⚠️ conv_cnt 아님!) |
| conv_amt | BIGINT | 전환매출 |
| ctr | DECIMAL(8,4) | 클릭률 |
| cpc | INT | 클릭당 비용 |
| ror | DECIMAL(12,2) | ROAS |

**⚠️ 전환수 컬럼명:** `ccnt` (NOT `conv_cnt`)

#### ★ naver_ad_gfa_campaign_daily (GFA 디스플레이 캠페인 일별)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK, AUTO) | 순번 |
| account | VARCHAR(20) | 광고 계정 |
| campaign_id | VARCHAR(100) | 캠페인 ID |
| campaign_name | VARCHAR(200) | ★ 캠페인명 (퍼널 식별: TOF/MOF/BOF/ADVoost/리타게팅) |
| stat_date | DATE | ★ 통계 날짜 |
| impressions | INT | 노출수 |
| clicks | INT | 클릭수 |
| total_cost | BIGINT | ★ 광고비 (소진액) |
| purchases | INT | ★ 구매 전환수 |
| purchase_conv_revenue | BIGINT | ★ 구매 전환매출 |

**GFA 퍼널 분류 (캠페인명 패턴):**
- `TOF%` → TOF (인지/트래픽)
- `MOF%` → MOF (고려)
- `BOF%` → BOF (전환)
- `%ADVoost%` → ADVoost (자동 최적화)
- `%리타게팅%` 또는 `%리타겟%` → Retargeting

#### naver_ad_bizmoney (비즈머니 잔액 — 시계열)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT (PK, AUTO) | 순번 |
| account | VARCHAR(20) | 광고 계정 (keychron/gtgear) |
| customer_id | VARCHAR(20) | 네이버 광고 고객 ID |
| bizmoney | BIGINT | ★ 잔액 (원) |
| collected_at | DATETIME | ★ 수집 시점 |

**⚠️ 컬럼명:** `bizmoney` (NOT `balance`)
**최신 잔액 조회:** `WHERE (account, collected_at) IN (SELECT account, MAX(collected_at) ...)`

#### campaign_brand_map (캠페인-브랜드 매핑)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| campaign_name | VARCHAR | 캠페인명 |
| brand | VARCHAR | 브랜드 (keychron/gtgear) |

#### ★ naver_smartstore_order_daily (스마트스토어 매출 일별)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| stat_date | DATE | ★ 통계 날짜 (⚠️ `date` 아님!) |
| account | VARCHAR | ★ 스토어 (keychron/gtgear) |
| revenue | BIGINT | ★ 매출 |
| order_cnt | INT | ★ 주문건수 |

**⚠️ 컬럼 주의:** `stat_date` (NOT `date`), `revenue` (NOT `total_revenue`)

#### 기타 광고 테이블
| 테이블 | 용도 |
|--------|------|
| naver_ad_adgroup_daily | SA 광고그룹 일별 성과 |
| naver_ad_gfa_adgroup_daily | GFA 광고그룹 일별 성과 |
| naver_ad_gfa_creative_daily | GFA 소재 일별 성과 |
| naver_ad_gfa_budget | GFA 예산 설정 |
| naver_ad_estimate_bid | 입찰가 추정 |
| naver_ad_search | 검색광고 설정 |
| naver_ad_strategy_config | 광고 전략 설정 |
| naver_ad_telegram_config | 텔레그램 알림 설정 |
| naver_ad_action_log | 광고 액션 로그 |
| yearly_ad_spend | 연간 광고비 |
| yearly_ad_summary | 연간 광고 요약 (⚠️ total_spend=0 다수, 불완전) |

#### 비즈데이터 테이블 (네이버 비즈 어드바이저)
| 테이블 | 용도 |
|--------|------|
| naver_bizdata_channel_daily | 채널별 일별 데이터 |
| naver_bizdata_custom_channel_daily | 커스텀 채널별 |
| naver_bizdata_customer | 고객 데이터 |
| naver_bizdata_hourly_sales | 시간대별 매출 |
| naver_bizdata_page_daily | 페이지별 일별 |
| naver_bizdata_product_performance | 상품 성과 |
| naver_bizdata_search_keyword | 검색 키워드 |
| naver_bizdata_website_daily | 웹사이트 일별 |

---

### 16. WMS 창고 관리 - 3 테이블

#### wms_relocation
| 컬럼 | 타입 | 설명 |
|------|------|------|
| relocation_seq | INT (PK, AUTO) | 순번 |
| relocation_date | DATE | 이동일 |
| warehouse_from | VARCHAR | 출발 창고 |
| warehouse_to | VARCHAR | 도착 창고 |
| member_id | VARCHAR | 담당자 |

---

### 17. 관리자 & 설정 - 15 테이블

| 테이블 | 용도 |
|--------|------|
| menu | 메뉴 마스터 |
| member_menu_rel | 사용자-메뉴 권한 |
| widget | 위젯 정의 |
| widget_layout | 위젯 레이아웃 |
| widget_permissions | 위젯 권한 |
| tbnws_admin_log | 감사 로그 |
| notice | 공지사항 |
| preset | 프리셋 설정 |

---

### 18. 시리얼 & 부품 - 12 테이블

| 테이블 | 용도 |
|--------|------|
| goods_serial | 상품 시리얼 번호 |
| erp_parts | 부품 카탈로그 |
| erp_parts_info | 부품 사양 |
| erp_parts_location | 부품 위치 |
| erp_parts_status | 부품 상태 |
| erp_parts_loaner | 부품 임대 |
| keychron_firmware | 펌웨어 정보 |
| keychron_product_manual | 제품 매뉴얼 |

### 19. AI 챗봇 제품 정보 (AI_CHATBOT DB) - 2 테이블

> DB: `AI_CHATBOT` (Bean: `sqlSessionTemplateAI`)
> 챗봇이 제품 추천·상세 스펙 안내에 사용하는 핵심 테이블

#### naver_product (네이버 스토어 상품 정보 — 211건)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| product_id | BIGINT (PK) | 네이버 상품번호 |
| product_name | VARCHAR(500) | 상품명 (옵션 포함, 예: "키크론 C1 PRO ... 레트로, 저소음 적축") |
| group_product_no | BIGINT (INDEX) | 그룹 상품번호 (같은 모델의 옵션들이 동일 값) |
| group_name | VARCHAR(500) | 그룹명 (모델명, 예: "키크론 C1 PRO 유선 사무용 기계식 키보드 텐키리스") |
| option_name | VARCHAR(500) | 옵션 (색상/축, 예: "레트로 / 저소음 적축") |
| sale_price | BIGINT | 판매가 (원) |
| discounted_price | BIGINT | 할인가 (원) |
| discount_rate | INT | 할인율 (%) |
| product_attributes | VARCHAR(2000) | 네이버 속성 (연결방식, 키스위치, 키캡 등 파이프 구분) |
| supplements | MEDIUMTEXT | 추가상품 JSON 배열 (팜레스트, 더스트커버 등) |
| seller_tags | VARCHAR(1000) | 셀러 태그 (쉼표 구분, 예: "가성비좋은,레트로,업무용") |
| event_phrase | VARCHAR(500) | 이벤트 문구 |
| is_sold_out | TINYINT(1) | 품절 여부 (0=판매중, 1=품절) |
| searchable_text | MEDIUMTEXT | 검색용 통합 텍스트 (상품명+속성+태그+추가상품 합본) |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

```
★ 인덱스: PK(product_id), UNIQUE(product_id), INDEX(group_product_no)
★ 용도: 네이버 스토어 기준 가격·할인·품절 정보 + 검색 최적화
★ group_product_no로 같은 모델의 옵션 묶기 가능
```

#### products (제품 상세 스펙 — 160건, 챗봇 핵심)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT (PK, AUTO_INCREMENT) | 제품 ID |
| product_name | VARCHAR(300) | 제품명 |
| product_name_synonyms | JSON | 동의어/별칭 배열 |
| price | VARCHAR(300) | 가격 |
| discontinued | TINYINT(1) | 단종 여부 (0=판매중, 1=단종) |
| release_date | VARCHAR(30) | 출시일 |
| key_binding | JSON | 키 바인딩 |
| tags | JSON | 태그 배열 |
| features | JSON | 기능 배열 |
| keyboard_layout | VARCHAR(300) | 키보드 배열 (75%, TKL, 풀사이즈 등) |
| keyboard_type | VARCHAR(300) | 키보드 타입 |
| switch_options | JSON | 스위치 옵션 배열 |
| multi_media_key_count | VARCHAR(300) | 멀티미디어 키 수 |
| main_frame_material | VARCHAR(300) | 프레임 소재 |
| key_cap_profile | VARCHAR(300) | 키캡 프로파일 |
| stabilizer | VARCHAR(300) | 스태빌라이저 |
| reinforcing_plate | VARCHAR(300) | 보강판 |
| n_key_rollover | VARCHAR(300) | N-키 롤오버 |
| plug_and_play | VARCHAR(300) | 플러그앤플레이 |
| polling_rate | VARCHAR(300) | 폴링레이트 |
| support_platforms | VARCHAR(300) | 지원 OS/플랫폼 |
| battery_capacity | VARCHAR(300) | 배터리 용량 |
| bluetooth_runtime | VARCHAR(300) | 블루투스 사용 시간 |
| backlight_pattern | VARCHAR(300) | 백라이트 패턴 |
| connection_method | VARCHAR(300) | 연결방식 (유선/무선/유무선) |
| supports_2_4ghz | VARCHAR(300) | 2.4GHz 지원 여부 |
| dynamic_keystroke | VARCHAR(300) | 다이나믹 키스트로크 |
| hot_swap_socket | VARCHAR(300) | 핫스왑 소켓 |
| rapid_trigger | VARCHAR(300) | 래피드 트리거 |
| size | VARCHAR(300) | 크기 |
| height_including_key_cap | VARCHAR(300) | 키캡 포함 높이 |
| height_not_including_key_cap | VARCHAR(300) | 키캡 미포함 높이 |
| package_contents | JSON | 패키지 구성품 |
| warranty_period | VARCHAR(300) | 보증 기간 |
| weight | VARCHAR(300) | 무게 |
| color | JSON | 색상 배열 |
| color_details | JSON | 색상 상세 |

```
★ 테이블 COMMENT: '제품 전체 스펙'
★ 용도: 챗봇이 제품 기술 스펙 질문에 답변 (스위치, 배열, 폴링레이트, 배터리 등)
★ product_name_synonyms: 제품 별칭으로 자연어 매칭 보조
```

#### naver_product ↔ products 관계

```
naver_product (네이버 판매 정보)          products (기술 스펙)
├─ product_id (네이버 상품번호)           ├─ id (자체 PK)
├─ product_name (판매명)                  ├─ product_name (제품명)
├─ sale_price / discounted_price         ├─ switch_options, polling_rate
├─ product_attributes (네이버 속성)       ├─ connection_method, battery_capacity
├─ supplements (추가상품)                 ├─ keyboard_layout, hot_swap_socket
└─ is_sold_out (품절)                     └─ features, tags

★ 직접 FK 없음 → product_name 기반 매칭
★ naver_product.group_name ≈ products.product_name (모델 단위)
★ naver_product: 가격/할인/품절/태그 (판매 관점)
★ products: 스위치/배열/배터리/폴링레이트 (기술 스펙 관점)
```

---

## 핵심 관계도 (Key Relationships)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        상품 체계                                      │
│  erp_category → erp_brand → erp_goods → erp_option                  │
│                                    ↓                                 │
│                           erp_product_info                           │
│                     (product_code = C-B-G-O)                         │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  재고 (Stock)  │   │  발주 (Order)  │   │  판매 (Sales)  │
│  erp_stock    │   │  erp_order    │   │  sabangnet   │
│  refurb_stock │   │  order_item   │   │  coupang     │
│  stock_adj    │   │  order_recv   │   │  naver       │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ↓
                  ┌──────────────┐
                  │  재무 (Finance) │
                  │  tax_invoice  │
                  │  estimate     │
                  │  etc_remit    │
                  └──────────────┘
```

---

## 주요 Enum 값 정리

| 필드 | 값 | 의미 |
|------|------|------|
| category_code | G | GTGear |
| category_code | F | Fulfillment |
| sell_status | Y | 판매중 |
| sell_status | N | 미판매 |
| sell_status | D | 삭제 |
| type (product) | G | 일반 |
| type (product) | F | 풀필먼트 |
| partner_type | IMPORT | 수입처 |
| partner_type | EXPORT | 출고처 |
| partner_type | BOTH | 양방향 |
| approval_flag | Y | 승인 |
| approval_flag | N | 반려 |
| approval_flag | (빈값) | 대기 |
| vacation_type | 연차 | 연차 |
| vacation_type | 오전반차 | 오전반차 |
| vacation_type | 오후반차 | 오후반차 |
| vacation_type | 경조사휴가 | 경조사 |
