# 재고 도메인 (Stock Domain)

---

## erp_stock (ERP 재고 - 개별 시리얼 단위)

summary: 개별 상품의 입출고를 시리얼(stock_seq) 단위로 추적하는 핵심 재고 테이블. 입고(import_date)부터 출고(export_date)까지의 전체 재고 라이프사이클을 관리한다.
domain: stock
related_tables: erp_stock_realtime, erp_stock_summary, erp_order_item, erp_product_info, erp_partner, pure_stock_view, stock_inventory_view

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| stock_seq | int unsigned | PK, AUTO_INCREMENT | 재고 고유번호 | |
| item_seq | int unsigned | NOT NULL | 입고 번호 (erp_order_item FK) | |
| import_date | date | NULL | 입고일시 (NULL이면 미입고) | |
| product_code | char(16) | NOT NULL | 상품 코드 | 'G-BR01-GD01-OP01' |
| export_seq | int | NULL | 출고 번호 | |
| export_partner_code | char(7) | NULL | 매출처 코드 (erp_partner FK) | 'ptn0091' |
| selling_price | int | DEFAULT 0 | 판매가 | |
| user_name | varchar(50) | DEFAULT '' | 수령인 이름 | |
| user_contact | varchar(50) | DEFAULT '' | 수령인 연락처 | |
| user_phone | varchar(50) | DEFAULT '' | 수령인 휴대폰 | |
| user_address | varchar(250) | DEFAULT '' | 수령인 주소 (배송지) | |
| user_memo | varchar(250) | DEFAULT '' | 배송 메세지 | |
| export_date | date | NULL | 출고일자 (NULL이면 미출고/가용재고) | |
| export_num | int | NULL | 출고차수 | |
| return_flag | char | DEFAULT 'N' | 반품 여부 | 'Y', 'N' |
| hc_send_date | datetime | NULL | 해피콜 발송일자 | |
| delivery_type | varchar(50) | DEFAULT '' | 배송속성 | |
| export_description | varchar(250) | DEFAULT '' | 출고 비고 | |
| courier | varchar(10) | DEFAULT '' | 택배사 | |
| invoice | varchar(255) | DEFAULT '' | 송장번호 | |
| invoice_date | datetime | NULL | 송장등록일자 | |
| warehouse_tag | varchar(10) | DEFAULT 'GT' | 창고 태그 | 'GT'(본사), 'GJ'(광주), 'AS'(AS센터) |
| location_code | char(16) | NULL | 로케이션 코드 (WMS 연동) | |
| import_grade | char | NULL | 발주 외 상품등급 | |
| product_type | char | DEFAULT 'G', NOT NULL | 상품 종류 | 'G'(판매상품), 'R'(리퍼브상품) |
| relocation_item_seq | int unsigned | NULL | 로케이션 이동 아이템 번호 | |
| buying_price | decimal(15,3) | DEFAULT 0.000, NOT NULL | 구매가 | |
| import_partner_code | char(7) | NULL | 입고 파트너 코드 | |
| import_seq | varchar(255) | NULL | 입고 시퀀스 | |
| cj_code | varchar(255) | NULL | CJ 풀필먼트 코드 | |
| export_order_date | datetime | NULL | 주문일자 | |
| export_selling_price | decimal(10,2) | DEFAULT 0.00 | 판매가 | |
| nff_order_num | varchar(255) | NULL | 네이버 풀필 주문번호 | |
| nff_product_order_num | varchar(255) | NULL | 네이버 풀필 상품주문번호 | |
| product_seq | int unsigned | NULL | 제품 시퀀스 | |

### 관계
- `item_seq` -> `erp_order_item.item_seq` (발주 아이템과 연결)
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- `export_partner_code` -> `erp_partner.partner_code` (매출처)
- `import_partner_code` -> `erp_partner.partner_code` (매입처)
- `stock_seq` <- `crm_report.compensation_stock_seq` (A/S 보상 재고)

### 비즈니스 맥락
- 가장 중요한 재고 테이블로, 모든 재고를 1건 1행(시리얼 단위)으로 관리
- G-코드(product_code 'G-'로 시작) = GT 본사 창고 재고, F-코드('F-'로 시작) = CJ 풀필먼트 센터 재고
- `import_date IS NULL` = 미입고(발주 중), `export_date IS NULL` = 미출고(가용 재고)
- `export_date IS NOT NULL AND user_address != ''` = 출고 완료
- `relocation_item_seq IS NOT NULL` = 로케이션 이동 중인 재고 (가용 재고에서 제외)
- `return_flag = 'Y'` = 반품된 재고 (판매량 집계 시 제외)
- 채널별 판매량 분석 시 자가이동/반품 파트너(ptn0000, ptn0001, ptn0100, ptn0103, ptn0124) 제외 필요
- 쿠팡로켓 파트너: ptn0091 (B2B 소속이지만 별도 분류)
- warehouse_tag: 'GT' = 본사, 'GJ' = 광주, 'AS' = AS센터

### 자주 쓰는 쿼리 패턴
- **가용 재고 조회**: `WHERE export_date IS NULL AND import_date IS NOT NULL AND relocation_item_seq IS NULL`
- **입고 예정 조회**: `WHERE import_date IS NULL`
- **출고 이력 조회**: `WHERE export_date IS NOT NULL AND return_flag = 'N'`
- **채널별 판매량 (erp_stock 기반)**: export_date 기간별 COUNT(*) + export_partner_code별 B2C/B2B/Rocket 분류
- **리드타임 실측**: erp_order + erp_order_item + erp_stock JOIN으로 AVG(DATEDIFF(import_date, order_date))
- **월별 출고 이력**: DATE_FORMAT(export_date, '%Y-%m') GROUP BY, return_flag='N' 및 자가이동 파트너 제외
- **잔여 재고 FIFO 선출**: `ORDER BY import_date ASC, stock_seq ASC LIMIT ?`

---

## product_available_stock (상품별 가용재고 - 실시간 캐시)

summary: 상품별 가용재고를 실시간으로 집계하는 캐시 테이블. 기존 여러 테이블을 JOIN하여 계산하던 가용재고를 미리 계산된 카운터로 대체하며, 일반적인 "재고 몇 개?" 질문에는 이 테이블을 최우선으로 사용한다. 특정일 기준 재고가 필요하면 erp_stock_summary를 사용한다.
domain: stock
related_tables: erp_stock, erp_export_schedule_item, erp_export_request, erp_order_recv_item, erp_order_recv, wms_relocation_item, erp_purchase_order, erp_product_info, erp_stock_summary

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | varchar(50) | PK | 상품코드 (G-code) | 'G-KC-Q1-HE-BNRDS' |
| stock_ea | int | DEFAULT 0 | 현재고 (입고완료, 미출고, 이동 아닌 것) | 150 |
| schedule_ea | int | DEFAULT 0 | 출고 예정 수량 | 10 |
| export_request_ea | int | DEFAULT 0 | 출고 요청 대기 수량 | 5 |
| pending_order_ea | int | DEFAULT 0 | 수주 미출고 수량 | 8 |
| relocation_outbound_ea | int | DEFAULT 0 | 재배치 출고 중 (이동 중) | 3 |
| relocation_inbound_ea | int | DEFAULT 0 | 재배치 입고 예정 (G↔F 코드 변환) | 2 |
| incoming_ea | int | DEFAULT 0 | 미입고 재고 (발주했지만 입고 확인 안 된 것) | 20 |
| pre_order_ea | int | DEFAULT 0 | 임의발주 미연결 수량 | 0 |
| available_ea | int | GENERATED | 가용재고 = stock_ea - schedule_ea - export_request_ea - pending_order_ea - relocation_outbound_ea + relocation_inbound_ea - incoming_ea - pre_order_ea | 106 |
| updated_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 마지막 갱신 시각 | |

### 관계
- `product_code` → `erp_product_info.product_code` (N:1): 상품 마스터 참조
- `stock_ea` 원본: `erp_stock` (import_date IS NOT NULL AND export_date IS NULL AND relocation_item_seq IS NULL)
- `schedule_ea` 원본: `erp_export_schedule_item` (export_flag = 'N')
- `export_request_ea` 원본: `erp_export_request` (request_flag IS NULL OR request_flag = 'W')
- `pending_order_ea` 원본: `erp_order_recv_item` + `erp_order_recv` (export_date IS NULL AND order_date IS NOT NULL)
- `relocation_outbound_ea` 원본: `wms_relocation_item` (import_date IS NULL)
- `relocation_inbound_ea` 원본: `wms_relocation_item` (import_date IS NULL, 반대 창고 코드로 G↔F 변환)
- `incoming_ea` 원본: `erp_stock` (import_date IS NULL AND export_date IS NULL)
- `pre_order_ea` 원본: `erp_purchase_order` (order_seq IS NULL)

### 비즈니스 맥락
- **재고 질문 라우팅 규칙**: "재고 몇 개?", "가용재고", "현재고" 등 일반적인 재고 수량 질문 → `product_available_stock` 사용. "특정일 기준 재고", "지난달 재고", "재고 추이" 등 과거/특정일 기준 → `erp_stock_summary` 사용.
- **UPSERT 패턴**: 비즈니스 로직이 원본 테이블 변경 시 `adjust{Counter}Ea(productCode, delta)` 호출로 실시간 갱신
- **delta 방식**: 절대값이 아닌 증감값으로 MySQL row-level lock 기반 동시성 안전
- **available_ea는 Generated Column**: DB에서 자동 계산 (위 공식)
- **G/F 코드 변환**: 재배치 시 GTGear(G)↔풀필먼트(F) 간 창고 코드가 바뀌므로 relocation_inbound_ea에서 코드 변환 처리
- **야간 보정(recalculate)**: 원본 테이블 8개를 전부 LEFT JOIN하여 실제 값으로 전체 덮어쓰기. drift(실시간 vs 실제 불일치) 자동 복구
- **뮤테이션 호출**: ExportService(8), ERPService(15), OrderService(22), FulfillService(1), B2BCRMService(9), B2CCRMService(10), WMSService(15), BizService(1) — 총 84개 호출 포인트

### 자주 쓰는 쿼리 패턴
- **단건 가용재고 조회**: `SELECT * FROM product_available_stock WHERE product_code = ?`
- **다건 가용재고 조회 (B2B 출고요청 분배)**: `SELECT * FROM product_available_stock WHERE product_code IN (?)`
- **G/F 창고 병합 조회**: `SELECT SUBSTRING(product_code, 2) AS base_code, SUM(available_ea) FROM product_available_stock WHERE product_code IN (CONCAT('G', ?), CONCAT('F', ?)) GROUP BY base_code`
- **drift 검출**: `SELECT * FROM product_available_stock pas LEFT JOIN (실제 집계 서브쿼리) actual WHERE pas.stock_ea != actual.stock_ea`
- **고아 재고 검출**: `SELECT * FROM erp_stock WHERE product_code NOT IN (SELECT product_code FROM erp_product_info)`

---

## erp_stock_realtime (ERP 실시간 재고 집계)

summary: 상품코드(productCode) 단위로 집계된 실시간 재고 현황 테이블. 현재 가용재고, 입출고 예정, 기간별 판매량/금액을 사전 계산하여 대시보드 성능을 최적화한다.
domain: stock
related_tables: erp_stock, erp_product_info, erp_stock_summary, product_available_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| productCode | varchar(50) | PK | 상품코드 (erp_product_info FK) | 'G-BR01-GD01-OP01' |
| ea | int | DEFAULT 0, NOT NULL | 현재 가용재고 | |
| beDueEa | int | DEFAULT 0, NOT NULL | 입고예정 수량 | |
| scheduleEa | int | DEFAULT 0, NOT NULL | 출고예정 수량 | |
| preOrderEa | int | DEFAULT 0, NOT NULL | 발주진행중 수량 | |
| stockPrice | decimal(15,2) | DEFAULT 0.00, NOT NULL | 재고 총 원가 | |
| latestBuyingPrice | decimal(15,2) | DEFAULT 0.00, NOT NULL | 최근 수입가 (가장 최근 입고 단가) | |
| oneWeekSalesEa | int | DEFAULT 0, NOT NULL | 1주 판매수량 | |
| oneMonthSalesEa | int | DEFAULT 0, NOT NULL | 1달 판매수량 | |
| threeMonthSalesEa | int | DEFAULT 0, NOT NULL | 3달 판매수량 | |
| oneWeekSalesPrice | decimal(15,2) | DEFAULT 0.00, NOT NULL | 1주 판매금액 | |
| oneMonthSalesPrice | decimal(15,2) | DEFAULT 0.00, NOT NULL | 1달 판매금액 | |
| threeMonthSalesPrice | decimal(15,2) | DEFAULT 0.00, NOT NULL | 3달 판매금액 | |
| updatedAt | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 마지막 갱신 시각 | |

### 관계
- `productCode` -> `erp_product_info.product_code` (상품 정보)
- G-코드 행과 F-코드 행을 합산해야 전체 재고 파악 가능: `LEFT JOIN erp_stock_realtime f ON CONCAT('F', SUBSTRING(g.productCode, 2)) = f.productCode`

### 비즈니스 맥락
- 실시간 Delta + 일일 배치 혼합 방식으로 갱신되는 집계 테이블
- G-코드(GT 창고)와 F-코드(CJ 풀필먼트)를 반드시 합산해야 정확한 재고 파악 가능
- F-코드 변환 공식: `CONCAT('F', SUBSTRING(G코드, 2))`
- days_of_supply(재고일수) 계산: `(ea + beDueEa) / (oneMonthSalesEa / 30.0)`
- sales_trend_ratio(판매 추세): `(oneWeekSalesEa / 7.0) / (threeMonthSalesEa / 90.0)` — 1.0 초과면 판매 증가 추세
- REORDER_ALERTS 분류 기준: STOCKOUT(품절+일평균>=0.3), BELOW_SAFETY(안전재고미달), LOW_2WEEKS(14일내소진), LOW_1MONTH(30일내소진)
- `p.sell_status != 'N'` 필터로 판매 중단 상품 제외

### 자주 쓰는 쿼리 패턴
- **현재 재고 전체 조회**: G-코드 기준 + F-코드 LEFT JOIN 합산, `WHERE g.productCode LIKE 'G-%'`
- **발주 필요 재고 알림**: days_of_supply 계산 후 STOCKOUT/BELOW_SAFETY/LOW_2WEEKS/LOW_1MONTH 분류
- **특정 상품 재고 상세**: 5-LEFT JOIN 구조 (출고예정 + 미입고발주 + 리드타임 + 최근입고)
- **옵션 그룹 재고 분석**: brand_code + goods_code 그룹핑으로 옵션별 재고 불균형 감지
- **출하촉진 필요 판단**: PENDING 발주 + beDueEa 교차검증 (14일+ & beDueEa=0 → CHECK, 30일+ → URGENT)
- **브랜드별 활성 상품**: `WHERE p.brand_name = ? AND monthly_sales > 0`

---

## erp_stock_summary (ERP 재고 요약 - 일별 스냅샷)

summary: erp_stock_realtime의 일별 스냅샷을 저장하는 대시보드 성능 최적화용 요약 테이블. 상품코드+날짜 복합 PK로 시계열 재고 추이를 추적한다. 특정일 기준 재고, 과거 재고 추이, 기간별 재고 비교 등 날짜 기반 재고 질문에 이 테이블을 사용한다. 일반적인 현재 재고 수량 질문에는 product_available_stock을 사용한다.
domain: stock
related_tables: erp_stock_realtime, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| product_code | varchar(50) | PK (복합) | 상품코드 | 'G-BR01-GD01-OP01' |
| category_code | varchar(10) | NULL | 카테고리코드 | |
| brand_code | varchar(10) | NULL | 브랜드코드 | |
| brand_name | varchar(100) | NULL | 브랜드명 | |
| goods_code | varchar(10) | NULL | 상품코드 | |
| goods_name | varchar(200) | NULL | 상품명 | |
| option_code | varchar(10) | NULL | 옵션코드 | |
| option_name | varchar(200) | NULL | 옵션명 | |
| cj_code | varchar(50) | NULL | CJ 코드 | |
| ea | int | DEFAULT 0 | 가용재고 | |
| be_due_ea | int | DEFAULT 0 | 입고예정 수량 | |
| schedule_ea | int | DEFAULT 0 | 출고예정 수량 | |
| location_schedule_ea | int | DEFAULT 0 | 로케이션 출고예정 수량 | |
| pre_order_ea | int | DEFAULT 0 | 발주진행중 수량 | |
| one_week_sales_ea | int | DEFAULT 0 | 1주 판매수량 | |
| one_month_sales_ea | int | DEFAULT 0 | 1달 판매수량 | |
| three_month_sales_ea | int | DEFAULT 0 | 3달 판매수량 | |
| one_week_sales_price | decimal(15,2) | DEFAULT 0.00 | 1주 판매금액 | |
| one_month_sales_price | decimal(15,2) | DEFAULT 0.00 | 1달 판매금액 | |
| three_month_sales_price | decimal(15,2) | DEFAULT 0.00 | 3달 판매금액 | |
| stock_price | decimal(15,2) | DEFAULT 0.00 | 재고 총 원가 | |
| summary_date | date | PK (복합), NOT NULL | 요약 기준 날짜 | |
| updated_at | timestamp | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 마지막 갱신 시각 | |

### 관계
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- erp_stock_realtime의 일별 스냅샷

### 비즈니스 맥락
- 복합 PK: (product_code, summary_date) — 상품별 일별 재고 현황 기록
- 대시보드에서 재고 추이 차트, 기간별 비교 분석에 사용
- erp_stock_realtime과 동일한 컬럼 구조이나 상품 메타데이터(brand_name, goods_name 등)를 비정규화하여 포함
- 일일 배치로 생성되며, 과거 재고 상태를 소급 조회할 때 사용

### 자주 쓰는 쿼리 패턴
- **기간별 재고 추이 조회**: `WHERE product_code = ? AND summary_date BETWEEN ? AND ?`
- **브랜드별 일별 집계**: `GROUP BY brand_code, summary_date`
- **카테고리별 일별 집계**: `GROUP BY category_code, summary_date`

---

## erp_stock_adjustment (재고 조정 이력)

summary: 수동 조정, 입고, 출고, 반품 등 모든 재고 변동을 기록하는 감사 로그 테이블. 조정 전후 수량과 사유를 추적한다.
domain: stock
related_tables: erp_stock, erp_stock_realtime, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | bigint | PK, AUTO_INCREMENT | 조정 고유번호 | |
| batch_id | varchar(36) | NULL | UUID, 일괄 처리 시 동일 값 | |
| product_code | varchar(50) | NOT NULL | 상품코드 | |
| refurbish_grade | char | NULL | 리퍼브 등급 | NULL, 'S', 'A', 'R' |
| adjustment_type | varchar(20) | NOT NULL | 조정 유형 | 'INCREASE'(증가), 'DECREASE'(감소) |
| quantity | int | NOT NULL | 조정 수량 (양수) | |
| before_quantity | int | NOT NULL | 조정 전 재고 | |
| after_quantity | int | NOT NULL | 조정 후 재고 | |
| reason | varchar(500) | NOT NULL | 조정 사유 | |
| created_by | varchar(50) | NOT NULL | 작업자 ID (member.uid) | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 생성 일시 | |
| reference_type | varchar(30) | NULL | 참조 테이블 유형 | 'order', 'as', 'recv' 등 |
| reference_seq | bigint | NULL | 참조 테이블의 PK | |

### 관계
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- `created_by` -> `member.uid` (작업자)

### 비즈니스 맥락
- 재고 수량 변동의 감사 추적(audit trail) 테이블
- batch_id로 일괄 처리 건을 묶어서 추적 가능
- adjustment_type: INCREASE(재고 증가), DECREASE(재고 감소)
- refurbish_grade: NULL(일반 상품), S/A/R(리퍼브 등급별)
- reference_type + reference_seq로 변동 원인이 된 원본 데이터 추적

### 자주 쓰는 쿼리 패턴
- **상품별 조정 이력**: `WHERE product_code = ? ORDER BY created_at DESC`
- **일괄 처리 조회**: `WHERE batch_id = ?`
- **작업자별 이력**: `WHERE created_by = ?`

---

## erp_refurb_stock (리퍼브 재고)

summary: 리퍼브(중고/재생) 상품의 입출고를 개별 건 단위로 관리하는 테이블. 등급(S/A/R)별 분류, 매입/매출 파트너, 수령인 정보를 포함한다.
domain: stock
related_tables: erp_refurb_export_request, erp_product_info, erp_partner, pure_refurb_inventory_view, refurb_inventory_pivot_view

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int unsigned | PK, AUTO_INCREMENT | 번호 | |
| import_partner_code | char(7) | NOT NULL | 매입처 코드 | |
| product_code | char(16) | NOT NULL | 상품 코드 | |
| buying_price | int | NOT NULL | 구매가 | |
| import_date | datetime | NULL | 입고일시 | |
| import_grade | varchar(1) | NOT NULL | 리퍼브 등급 | 'S', 'A', 'R', '-' |
| description | text | NULL | 입고 비고 | |
| sale_flag | varchar(10) | DEFAULT 'N' | 판매 등록 여부 | 'Y'(등록), 'N'(비등록) |
| export_partner_code | char(7) | NULL | 매출처 코드 | |
| export_num | int | NULL | 출고차수 | |
| selling_price | int | NULL | 판매가 | |
| user_name | varchar(50) | DEFAULT '' | 수령인 이름 | |
| user_contact | varchar(50) | DEFAULT '' | 수령인 연락처 | |
| user_phone | varchar(50) | DEFAULT '' | 수령인 휴대폰 | |
| user_address | varchar(250) | DEFAULT '' | 수령인 주소 | |
| user_memo | varchar(250) | DEFAULT '' | 배송 메세지 | |
| export_date | datetime | NULL | 출고일자 | |
| export_description | text | NULL | 출고 비고 | |
| order_date | datetime | NULL | 주문일자 | |
| import_seq | bigint | NULL | 입고 시퀀스 | |
| return_flag | char | DEFAULT 'N', NOT NULL | 반품 여부 | 'Y', 'N' |
| courier | varchar(10) | DEFAULT '' | 택배사 | |
| invoice | varchar(255) | DEFAULT '' | 송장번호 | |
| invoice_date | datetime | NULL | 송장등록일자 | |

### 관계
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- `import_partner_code` -> `erp_partner.partner_code` (매입 파트너)
- `export_partner_code` -> `erp_partner.partner_code` (매출 파트너)

### 비즈니스 맥락
- erp_stock과 유사한 구조이지만 리퍼브(중고/재생) 상품 전용
- import_grade 등급 우선순위: S(최상) > A(양호) > R(수리 필요) > -(미분류)
- `export_date IS NULL` = 가용 재고, `export_date IS NOT NULL` = 출고 완료
- `sale_flag = 'Y'` = 판매 등록 상태, 'N' = 비등록 상태
- 출고 시 FIFO 적용: `ORDER BY import_date ASC, seq ASC`
- 리퍼브 출고 확인 시 export_partner_code, selling_price, user_* 정보 일괄 업데이트
- 송장 관리: courier(택배사) + invoice(송장번호)로 배송 추적

### 자주 쓰는 쿼리 패턴
- **가용 재고 조회**: `WHERE export_date IS NULL AND import_date IS NOT NULL ORDER BY product_code, import_grade`
- **등급별 재고 집계**: `GROUP BY product_code, import_grade`
- **출고 이력 조회**: `WHERE export_date IS NOT NULL` + erp_partner, erp_product_info JOIN
- **리퍼브 송장 조회**: `WHERE export_date IS NOT NULL` + courier, invoice 필터
- **FIFO 출고 대상**: `WHERE product_code = ? AND import_grade = ? AND export_date IS NULL ORDER BY import_date ASC, seq ASC LIMIT ?`

---

## erp_refurb_export_request (리퍼브 출고 요청)

summary: 리퍼브 상품의 출고 요청을 관리하는 테이블. 요청 등록 후 승인(request_flag='Y') 처리하며, 미승인 요청은 가용재고에서 차감(출고예정)으로 반영된다.
domain: stock
related_tables: erp_refurb_stock, erp_product_info, member, pure_refurb_inventory_view

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 고유번호 | |
| title | varchar(100) | NULL | 제목 | |
| type | varchar(20) | NULL | 유형 | |
| export_schedule_date | date | NULL | 출고 예정일 | |
| export_method | varchar(10) | NULL | 출고 방법 | |
| stock_flag | char | DEFAULT 'O' | 재고 상태 플래그 | 'O' |
| product_code | varchar(100) | NULL | 상품코드 | |
| grade | char | NULL | 리퍼브 등급 | 'S', 'A', 'R' |
| user_name | varchar(30) | DEFAULT '' | 수령인 이름 | |
| user_contact | varchar(15) | DEFAULT '' | 수령인 연락처 | |
| user_address | varchar(250) | DEFAULT '' | 수령인 주소 | |
| user_postal_code | varchar(10) | DEFAULT '' | 수령인 우편번호 | |
| user_memo | varchar(250) | DEFAULT '' | 배송 메세지 | |
| description | varchar(250) | DEFAULT '' | 비고 | |
| request_flag | char | NULL | 요청 승인 여부 | 'Y'(승인), NULL(미승인) |
| regist_muid | varchar(30) | NULL | 등록자 UID | |
| regist_member_id | int | NULL | 등록자 멤버 ID | |
| regist_date | datetime | DEFAULT NOW() | 등록일시 | |

### 관계
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- `regist_member_id` -> `member.id` (등록자)

### 비즈니스 맥락
- 리퍼브 출고 요청 워크플로우: 요청 등록 → 승인(request_flag='Y') → 실제 출고(erp_refurb_stock 업데이트)
- `request_flag != 'Y'`인 미승인 요청은 pure_refurb_inventory_view에서 export_requested_count로 집계
- 이를 통해 가용 재고에서 출고 예정 수량을 차감한 실제 가용 수량 계산

### 자주 쓰는 쿼리 패턴
- **출고 요청 목록**: `SELECT * FROM erp_refurb_export_request` + erp_product_info, member JOIN
- **미승인 요청 집계**: `WHERE request_flag != 'Y' GROUP BY product_code, grade`
- **승인 처리**: `UPDATE SET request_flag = 'Y' WHERE seq IN (?)`

---

## branch_list (오프라인 지점 목록)

summary: 오프라인 매장/지점 정보를 관리하는 마스터 테이블. 지점코드, 주소, 담당자 정보를 포함한다.
domain: stock
related_tables: branch_product_list, branch_stock, branch_stock_pending, branch_supplier_codes, branch_gift_rule

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 지점 고유번호 | |
| biz_code | varchar(100) | NOT NULL | 사업자 코드 | |
| branch_code | varchar(100) | NOT NULL | 지점 코드 | |
| branch_name | varchar(100) | NOT NULL | 지점명 | |
| zip_code | varchar(100) | NOT NULL | 우편번호 | |
| branch_address | varchar(500) | NOT NULL | 지점 주소 | |
| branch_manager_name | varchar(100) | NOT NULL | 담당자 이름 | |
| branch_manager_position | varchar(100) | NOT NULL | 담당자 직책 | |
| branch_manager_phone | varchar(100) | NOT NULL | 담당자 전화번호 | |
| branch_manager_email | varchar(200) | NOT NULL | 담당자 이메일 | |
| description | text | NULL | 비고 | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 등록일시 | |

### 관계
- `seq` <- `branch_product_list.branch_seq` (지점별 취급 상품)
- `seq` <- `branch_stock.branch_seq` (지점별 재고)
- `seq` <- `branch_stock_pending.branch_seq` (지점별 입고 대기)
- `biz_code` <- `branch_supplier_codes.biz_code` (사업자별 매입처)
- `biz_code` <- `branch_gift_rule.biz_code` (사업자별 사은품 규칙)

### 비즈니스 맥락
- B2B CRM 도메인에서 관리되는 오프라인 지점(매장) 마스터 테이블
- biz_code로 사업자 단위 그룹핑, branch_code로 지점 식별

### 자주 쓰는 쿼리 패턴
- **사업자별 지점 목록**: `WHERE biz_code = ?`
- **지점 상세 조회**: `WHERE seq = ?`

---

## branch_product_list (지점 취급 상품 목록)

summary: 각 오프라인 지점에서 취급하는 상품 목록과 납품가/행사가를 관리하는 테이블. 바코드를 통한 POS 연동을 지원한다.
domain: stock
related_tables: branch_list, branch_stock, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 고유번호 | |
| branch_seq | int | NOT NULL | 지점 번호 (branch_list FK) | |
| barcode | varchar(100) | NOT NULL | 바코드 | |
| product_code | varchar(100) | NOT NULL | 상품 코드 | |
| delivery_price | decimal(10,2) | DEFAULT 0.00 | 납품가 | |
| promotion_price | decimal(10,2) | DEFAULT 0.00 | 행사가 | |
| is_delivered | char | DEFAULT 'Y', NOT NULL | 납품 여부 | 'Y', 'N' |
| is_soldout | varchar(1) | DEFAULT 'N', NOT NULL | 품절 여부 | 'Y', 'N' |

### 관계
- `branch_seq` -> `branch_list.seq` (소속 지점)
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- UNIQUE 제약: (branch_seq, product_code) — 지점별 상품 중복 등록 방지

### 비즈니스 맥락
- 지점별 취급 가능 상품과 가격 정책(납품가/행사가) 관리
- is_delivered: 납품 여부 (지점에 실제 입고된 상태)
- is_soldout: 품절 상태 플래그
- barcode를 통해 POS 시스템과 연동

### 자주 쓰는 쿼리 패턴
- **지점별 상품 목록**: `WHERE branch_seq = ?` + erp_product_info JOIN
- **바코드로 상품 찾기**: `WHERE branch_seq = ? AND barcode = ?`
- **사업자별 상품 가격 일괄 조회**: branch_list JOIN + `GROUP BY barcode, product_code`

---

## branch_stock (지점 재고)

summary: 오프라인 지점의 개별 입출고 이력을 관리하는 테이블. 입고일/판매일, 매입처, 매출 정보(VAT 포함/별도)를 기록한다.
domain: stock
related_tables: branch_list, branch_product_list, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 번호 | |
| branch_seq | varchar(100) | NOT NULL | 지점번호 (branch_list FK) | |
| product_code | varchar(100) | NOT NULL | 상품코드 | |
| import_date | date | NOT NULL | 입고일자 | |
| import_description | varchar(500) | NULL | 입고 전용 비고 | |
| export_date | date | NULL | 판매일자 | |
| supplier_code | varchar(100) | NULL | 매입처코드 (branch_supplier_codes FK) | |
| avg_sale_price_vat | int | NULL | 평균판매단가 (VAT 포함) | |
| net_sales_ex_vat | int | NULL | 순매출 (VAT 별도) | |
| vat_amount | int | NULL | 부가세 | |
| export_description | varchar(500) | NULL | 출고 전용 비고 | |
| return_flag | varchar(10) | DEFAULT 'N', NOT NULL | 반품 여부 | 'Y', 'N' |

### 관계
- `branch_seq` -> `branch_list.seq` (소속 지점)
- `product_code` -> `erp_product_info.product_code` (상품 정보)
- `supplier_code` -> `branch_supplier_codes.supplier_code` (매입처)

### 비즈니스 맥락
- erp_stock과 유사한 1건 1행 구조이지만 오프라인 지점용
- `export_date IS NULL` = 가용 재고 (미판매)
- `export_date IS NOT NULL` = 판매 완료
- `return_flag = 'Y'` = 반품 처리
- 매출 정보: avg_sale_price_vat(VAT 포함 판매가), net_sales_ex_vat(VAT 제외 순매출), vat_amount(부가세)

### 자주 쓰는 쿼리 패턴
- **지점별 현재 재고**: `WHERE branch_seq = ? AND export_date IS NULL GROUP BY product_code`
- **지점별 판매 이력**: `WHERE branch_seq = ? AND export_date IS NOT NULL`
- **지점 입고 이력**: `GROUP BY product_code, import_date`
- **매출 집계**: `WHERE export_date BETWEEN ? AND ? GROUP BY product_code` + SUM(avg_sale_price_vat, net_sales_ex_vat, vat_amount)

---

## branch_stock_pending (지점 입고 대기)

summary: 오프라인 지점에 입고 예정인 상품의 대기 현황을 관리하는 테이블. 발주와 연결(recv_item_seq)하여 입고 추적이 가능하다.
domain: stock
related_tables: branch_list, branch_stock, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 번호 | |
| branch_seq | int | NOT NULL | 지점번호 (branch_list FK) | |
| product_code | varchar(100) | NOT NULL | 상품코드 | |
| ea | int | NOT NULL | 수량 | |
| description | varchar(500) | NULL | 비고 | |
| recv_item_seq | int | NULL | 수령 아이템 시퀀스 (발주 연결) | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 등록일시 | |

### 관계
- `branch_seq` -> `branch_list.seq` (소속 지점)
- `product_code` -> `erp_product_info.product_code` (상품 정보)

### 비즈니스 맥락
- 지점으로의 입고 대기(예정) 수량을 관리
- `recv_item_seq IS NULL` = 아직 발주와 미연결 상태
- recv_item_seq가 설정되면 실제 발주와 연결된 입고 대기
- branch_product_list 재고 현황에서 pending 수량으로 표시

### 자주 쓰는 쿼리 패턴
- **지점별 입고 대기**: `WHERE branch_seq = ?` + erp_product_info JOIN
- **미연결 대기 목록**: `WHERE recv_item_seq IS NULL`
- **대기 수량 집계**: `GROUP BY branch_seq, product_code`

---

## branch_supplier_codes (지점 매입처 코드)

summary: 오프라인 지점(사업자)별로 사용하는 매입처 코드와 수수료율을 관리하는 마스터 테이블.
domain: stock
related_tables: branch_list, branch_stock

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 고유번호 | |
| biz_code | varchar(100) | NOT NULL | 사업자 코드 | |
| supplier_code | varchar(100) | NOT NULL | 매입처 코드 | |
| supplier_name | varchar(100) | NOT NULL | 매입처명 | |
| description | text | NOT NULL | 설명 | |
| fee_rate | int | NOT NULL | 수수료율 (%) | |

### 관계
- `biz_code` -> `branch_list.biz_code` (소속 사업자)
- `supplier_code` <- `branch_stock.supplier_code` (재고 매입처 참조)

### 비즈니스 맥락
- 사업자(biz_code) 단위로 매입처를 관리
- fee_rate: 수수료율(%), 매출 정산 시 활용

### 자주 쓰는 쿼리 패턴
- **사업자별 매입처 목록**: `WHERE biz_code = ? ORDER BY seq`

---

## branch_gift_rule (지점 사은품 규칙)

summary: 오프라인 지점(사업자)별 사은품 증정 규칙을 관리하는 테이블. 특정 상품 구매 시 자동으로 사은품을 증정하는 규칙을 정의한다.
domain: stock
related_tables: branch_list, branch_product_list, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| seq | int | PK, AUTO_INCREMENT | 고유번호 | |
| biz_code | varchar(100) | NOT NULL | 사업자 코드 | |
| trigger_product_code | varchar(50) | NULL | 구매 트리거 상품코드 | |
| gift_product_code | varchar(50) | NULL | 사은품 상품코드 | |
| gift_qty | int | DEFAULT 1 | 사은품 수량 | |
| is_active | char | DEFAULT 'Y' | 활성 여부 | 'Y', 'N' |
| start_date | date | DEFAULT CURDATE(), NOT NULL | 시작일 | |
| end_date | date | DEFAULT CURDATE(), NOT NULL | 종료일 | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| updated_at | datetime | DEFAULT CURRENT_TIMESTAMP | 수정일시 | |

### 관계
- `biz_code` -> `branch_list.biz_code` (소속 사업자)
- `trigger_product_code` -> `erp_product_info.product_code` (트리거 상품)
- `gift_product_code` -> `erp_product_info.product_code` (사은품 상품)

### 비즈니스 맥락
- trigger_product_code 상품 구매 시 gift_product_code를 gift_qty만큼 증정
- is_active = 'N'으로 soft delete 처리
- start_date ~ end_date 기간 동안만 규칙 적용
- 인덱스: (biz_code, trigger_product_code)

### 자주 쓰는 쿼리 패턴
- **사업자별 활성 규칙**: `WHERE biz_code = ? AND is_active = 'Y'` + erp_product_info JOIN (트리거/사은품 모두)
- **비활성화**: `UPDATE SET is_active = 'N' WHERE seq = ?`

---

## stock_deduction_log (재고 차감 이력)

summary: 풀필먼트 출고 시 재고 차감 처리 이력을 기록하는 테이블. 성공/실패(부족) 상태를 추적하고, 실패 건에 대한 후속 처리(RESOLVED)를 관리한다.
domain: stock
related_tables: erp_stock, erp_product_info, erp_partner

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| idx | bigint | PK, AUTO_INCREMENT | 고유번호 | |
| orderIdx | varchar(255) | NOT NULL, UNIQUE(복합) | 주문번호 | |
| productOrderId | varchar(255) | NOT NULL, UNIQUE(복합) | 상품주문번호 | |
| partnerCode | varchar(100) | NOT NULL | 파트너 코드 | |
| productCode | varchar(255) | NOT NULL | 상품코드 | |
| requestEa | bigint | NOT NULL | 차감 재고 수량 | |
| shortEa | bigint | DEFAULT 0 | 부족 수량 (실패 시) | |
| status | varchar(50) | DEFAULT 'COMPLETED' | 처리 상태 | 'COMPLETED', 'PENDING', 'RESOLVED' |
| resolvedAt | datetime | NULL | 실패 > 성공 처리 일시 | |
| createdAt | datetime | DEFAULT CURRENT_TIMESTAMP | 생성일시 | |
| resolved_member_id | bigint | NULL | 해결 처리자 멤버 ID | |
| errorDescription | varchar(255) | NULL | 에러 로그 | |

### 관계
- `productCode` -> `erp_product_info.product_code` (상품 정보)
- `partnerCode` -> `erp_partner.partner_code` (파트너)
- UNIQUE 제약: (orderIdx, productOrderId) — 동일 주문의 중복 차감 방지

### 비즈니스 맥락
- 풀필먼트 자동 재고 차감의 감사 로그
- status 흐름: COMPLETED(정상 차감) / PENDING(재고 부족으로 차감 실패) → RESOLVED(수동 해결)
- shortEa > 0: 재고 부족으로 요청 수량만큼 차감하지 못한 경우
- PENDING 상태의 건은 관리자가 수동으로 재고 확보 후 RESOLVED 처리

### 자주 쓰는 쿼리 패턴
- **미해결 건 조회**: `WHERE status = 'PENDING'` + erp_product_info, erp_partner JOIN
- **상태 업데이트**: `UPDATE SET status = ?, resolvedAt = NOW(), resolved_member_id = ? WHERE idx = ?`
- **주문별 차감 이력**: `WHERE orderIdx = ?`

---

## coupang_inventory_daily_log (쿠팡 일별 재고 스냅샷 로그)

summary: 쿠팡 로켓그로스/로켓배송의 일별 재고 현황을 스냅샷으로 저장하는 로그 테이블. SKU별 현재고, 입출고 수량, 매입원가를 기록한다.
domain: stock
related_tables: erp_coupang

### 컬럼
| 컬럼명 | 타입 | 제약조건 | 설명 | 샘플 값/ENUM |
|--------|------|----------|------|-------------|
| id | bigint unsigned | PK, AUTO_INCREMENT | 고유번호 | |
| sale_date | date | NOT NULL | 기준일 | |
| brand | varchar(100) | NOT NULL | 브랜드 | |
| sku_id | bigint | NOT NULL | 쿠팡 SKU ID | |
| sku_name | varchar(500) | NOT NULL | SKU 이름 | |
| barcode | varchar(100) | NULL | 바코드 | |
| current_stock_qty | int | DEFAULT 0, NOT NULL | 현재고 수량 | |
| purchase_cost | decimal(15,2) | DEFAULT 0.00, NOT NULL | 매입원가 | |
| inbound_qty | int | DEFAULT 0, NOT NULL | 입고 수량 | |
| outbound_qty | int | DEFAULT 0, NOT NULL | 출고 수량 | |
| created_at | datetime | DEFAULT CURRENT_TIMESTAMP, NOT NULL | 생성일시 | |

### 관계
- `sku_id` -> `erp_coupang.sku_id` (쿠팡 상품 매핑)
- 인덱스: brand, (sale_date, sku_id), sale_date, (sku_id ASC, sale_date DESC), sku_id

### 비즈니스 맥락
- 쿠팡 로켓 물류센터의 일별 재고/판매 데이터를 수집하여 저장
- outbound_qty = 쿠팡 출고 수량 (판매량)
- inbound_qty = 쿠팡 입고 수량
- 브랜드별(GTGear, Keychron, Aiper 등) 판매 분석에 사용
- erp_coupang 테이블과 sku_id로 연결하여 사내 상품코드와 매핑
- sku_id 미매핑 건 감지: `LEFT JOIN erp_coupang e ON t.sku_id = e.sku_id WHERE e.sku_id IS NULL`

### 자주 쓰는 쿼리 패턴
- **SKU별 판매량 분석**: `WHERE sale_date BETWEEN ? AND ? GROUP BY sku_id` + SUM(outbound_qty), SUM(inbound_qty)
- **브랜드별 일별 판매 추이**: `GROUP BY brand, sale_date`
- **브랜드별 기간 합계**: `GROUP BY brand` + SUM(outbound_qty), daily_avg 계산
- **최신 재고 조회**: 서브쿼리로 `MAX(sale_date)` 기준 current_stock_qty 조회
- **미매핑 SKU 감지**: erp_coupang LEFT JOIN + `WHERE e.sku_id IS NULL`

---

## pure_stock_view (순수 재고 뷰)

summary: erp_stock 테이블에서 미출고(export_date IS NULL) 재고를 상품코드+창고별로 집계하여 입고예정/출고예정/이동중/가용 재고를 분류하는 뷰.
domain: stock
related_tables: erp_stock, stock_inventory_view

### 컬럼
| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| product_code | char(16) | 상품 코드 |
| warehouse_tag | varchar(10) | 창고 태그 ('GT', 'GJ', 'AS') |
| incoming_count | bigint | 입고예정 수량 (import_date IS NULL) |
| outgoing_count | bigint | 출고예정 수량 (import_date IS NOT NULL AND user_address != '') |
| relocation_count | bigint | 로케이션 이동 중 수량 (import_date IS NOT NULL AND user_address IS NULL/'' AND relocation_item_seq IS NOT NULL) |
| available_count | bigint | 가용 재고 수량 (import_date IS NOT NULL AND user_address IS NULL/'' AND relocation_item_seq IS NULL) |

### 조인 테이블
- `erp_stock` (WHERE export_date IS NULL, GROUP BY product_code, warehouse_tag)

### 비즈니스 맥락
- erp_stock에서 export_date IS NULL인 행만 대상으로 4가지 상태로 분류:
  - incoming_count: import_date가 NULL인 행 = 아직 입고되지 않은 발주 상품
  - outgoing_count: import_date가 있고 user_address가 있는 행 = 출고 준비 중
  - relocation_count: 입고되었지만 로케이션 이동 중인 행
  - available_count: 입고 완료, 출고/이동 없는 순수 가용 재고
- stock_inventory_view의 소스 뷰

### 자주 쓰는 쿼리 패턴
- **상품별 가용 재고**: `WHERE product_code = ?`
- **창고별 재고 현황**: `WHERE warehouse_tag = ?`

---

## pure_refurb_inventory_view (순수 리퍼브 재고 뷰)

summary: erp_refurb_stock의 가용 재고와 erp_refurb_export_request의 출고 요청을 UNION ALL로 합산하여 상품코드+등급별 재고 및 출고요청 수량을 집계하는 뷰.
domain: stock
related_tables: erp_refurb_stock, erp_refurb_export_request, refurb_inventory_pivot_view

### 컬럼
| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| product_code | char(16) | 상품 코드 |
| import_grade | varchar(1) | 리퍼브 등급 ('S', 'A', 'R', '-') |
| stock_count | bigint | 가용 재고 수량 (입고완료+미출고) |
| export_requested_count | bigint | 출고 요청 수량 (미승인 요청) |

### 조인 테이블
- UNION ALL:
  - `erp_refurb_stock` (WHERE import_date IS NOT NULL AND export_date IS NULL) → stock_count
  - `erp_refurb_export_request` (WHERE request_flag != 'Y') → export_requested_count

### 비즈니스 맥락
- stock_count: 입고되었지만 아직 출고되지 않은 리퍼브 가용 재고
- export_requested_count: 출고 요청이 등록되었지만 아직 승인되지 않은 수량
- 실제 가용 재고 = stock_count - export_requested_count
- refurb_inventory_pivot_view의 소스 뷰

### 자주 쓰는 쿼리 패턴
- **상품+등급별 재고**: `WHERE product_code = ?`
- **가용 재고 계산**: `stock_count - export_requested_count`

---

## refurb_inventory_pivot_view (리퍼브 재고 피벗 뷰)

summary: pure_refurb_inventory_view를 등급(S/A/R/N)별로 피벗하여 상품코드 단위로 각 등급의 재고 수량과 출고요청 수량을 컬럼으로 전개하는 뷰.
domain: stock
related_tables: pure_refurb_inventory_view, product_view, erp_product_info

### 컬럼
| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| product_code | varchar | 상품 코드 |
| category_code | varchar | 카테고리 코드 |
| category_name | varchar | 카테고리명 |
| brand_code | varchar | 브랜드 코드 |
| brand_name | varchar | 브랜드명 |
| goods_code | varchar | 상품 코드 |
| goods_name | varchar | 상품명 |
| option_code | varchar | 옵션 코드 |
| option_name | varchar | 옵션 이름 |
| stock_count_S | bigint | S등급 가용 재고 |
| export_requested_count_S | bigint | S등급 출고 요청 수량 |
| stock_count_A | bigint | A등급 가용 재고 |
| export_requested_count_A | bigint | A등급 출고 요청 수량 |
| stock_count_R | bigint | R등급 가용 재고 |
| export_requested_count_R | bigint | R등급 출고 요청 수량 |
| stock_count_N | bigint | 기타등급(S/A/R 외) 가용 재고 |
| export_requested_count_N | bigint | 기타등급 출고 요청 수량 |

### 조인 테이블
- `product_view` LEFT JOIN `pure_refurb_inventory_view` ON product_code
- GROUP BY product_code, CASE WHEN import_grade = 'S'/'A'/'R' THEN ... ELSE 'N'

### 비즈니스 맥락
- 등급별로 컬럼을 피벗하여 상품 단위로 전체 리퍼브 재고 현황을 한 행에 표시
- 프론트엔드 리퍼브 재고 화면에서 직접 사용
- available_count 계산: `(stock_count_S - export_requested_count_S)` 등 등급별 계산

### 자주 쓰는 쿼리 패턴
- **리퍼브 재고 현황 전체**: `SELECT * FROM refurb_inventory_pivot_view`
- **특정 상품 리퍼브 재고**: `WHERE product_code IN (?)`
- **등급별 가용 재고 계산**: `(stock_count_S - export_requested_count_S) AS available_count_S`

---

## stock_inventory_view (재고 인벤토리 뷰)

summary: product_view + pure_stock_view + export_scheduled_ea_view를 조인하여 상품 마스터 정보와 함께 창고별 입고예정/출고예정/이동중/가용 재고를 통합 제공하는 뷰.
domain: stock
related_tables: pure_stock_view, product_view, erp_product_info, export_scheduled_ea_view

### 컬럼
| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| product_code | varchar | 상품 코드 |
| category_code | varchar | 카테고리 코드 |
| category_name | varchar | 카테고리명 |
| brand_code | varchar | 브랜드 코드 |
| brand_name | varchar | 브랜드명 |
| goods_code | varchar | 상품 코드 |
| goods_name | varchar | 상품명 |
| option_code | varchar | 옵션 코드 |
| option_name | varchar | 옵션 이름 |
| warehouse_tag | varchar(10) | 창고 태그 |
| incoming_count | int | 입고예정 수량 |
| outgoing_count | int | 출고예정 수량 |
| relocation_count | int | 로케이션 이동 중 수량 |
| available_count | int | 가용 재고 수량 |
| export_scheduled_ea | int | 출고 스케줄 수량 |

### 조인 테이블
- `product_view` LEFT JOIN `pure_stock_view` ON product_code
- LEFT JOIN `export_scheduled_ea_view` ON product_code

### 비즈니스 맥락
- 재고 관리 화면의 메인 데이터소스로 사용
- 상품 마스터(product_view) + 순수 재고(pure_stock_view) + 출고 스케줄(export_scheduled_ea_view) 통합
- 성능 이슈로 현재 MyBatis에서 직접 계산 방식(selectStockInventory)으로 대체하여 사용하는 경우도 있음
- NULL 값은 IFNULL(..., 0)으로 처리

### 자주 쓰는 쿼리 패턴
- **재고 인벤토리 조회**: `WHERE product_code IN (?)` + warehouse_tag 필터
- **창고별 재고 현황**: `WHERE warehouse_tag = ?`
- **전체 재고 대시보드**: 상품 계층(category > brand > goods > option) 기준 집계
