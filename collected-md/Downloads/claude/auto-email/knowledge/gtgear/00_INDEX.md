# Weaviate Text-to-SQL Enriched Schema Index

Generated: 2026-03-11 14:18

## File Structure

| File | Domain | Tables | Views | Lines |
|------|--------|--------|-------|-------|
| 01_product.md | ERP Product Master & Pricing | 19 | 4 | 748 |
| 02_order.md | ERP Orders & Purchasing | 19 | 0 | 746 |
| 03_stock.md | ERP Stock & Inventory | 15 | 4 | 820 |
| 04_export_sales.md | Export, Shipment & Sales | 15 | 1 | 713 |
| 05_crm.md | CRM, A/S & Customer | 20 | 0 | 950 |
| 06_naver.md | Naver SmartStore Channel | 24 | 0 | 1068 |
| 07_finance_event.md | Finance & Events/Marketing | 26 | 0 | 1170 |
| 08_wms_parts_misc.md | WMS, Parts, Serial & Misc | 35 | 0 | 1493 |
| **Total** | | **173** | **9** | **7708** |

## Weaviate Property Mapping

Each `## table_name` section maps to a Weaviate `TableSchema` object:

| Markdown Element | Weaviate Property | Vectorized | Purpose |
|-----------------|-------------------|------------|---------|
| First paragraph after summary: | summary | Yes | Semantic search |
| Full section content | description / markdown_full | Yes / No | Semantic search / LLM context |
| `## table_name` header | table_name | No (BM25) | Keyword matching |
| Column names from table | column_names[] | No (BM25) | Column filtering |
| DDL from ddl.md | ddl | No | LLM context |
| `related_tables:` line | related_tables[] | No | FK 1-hop expansion |
| `domain:` line | domain | No | Domain filtering |

## Parsing Instructions

1. Split each file by `## ` headers to get individual table chunks
2. Extract `summary:` line -> summary property
3. Extract `domain:` line -> domain property
4. Extract `related_tables:` line -> related_tables[] (split by comma)
5. Extract column names from `### columns` table -> column_names[]
6. Full section text -> markdown_full property
7. Look up DDL from ddl.md by table_name -> ddl property

## Table List

### 00_INDEX.md
- File Structure
- Weaviate Property Mapping
- Parsing Instructions
- Table List

### 01_product.md
- erp_category (ERP 카테고리 목록)
- erp_brand (ERP 브랜드 목록)
- erp_goods (ERP 상품 목록)
- erp_option (ERP 옵션 목록)
- erp_item_code (ERP 품목코드 매핑)
- erp_product_info (ERP 제품 마스터)
- erp_product_details (ERP 제품 상세 스펙)
- erp_product_image (ERP 제품 이미지)
- erp_product_components (ERP 제품 구성품)
- erp_product_qc (ERP 제품 QC 정보)
- erp_channel_list (ERP 판매 채널 목록)
- erp_channel_price (ERP 채널별 상품 가격)
- erp_discounted_products (ERP 할인 상품 가격)
- eflexs_product (eflexs 외부 ERP 상품 매핑)
- eflux_recv_stock (eflexs 입고 재고)
- TBNWS_ADMIN_eflux_recv_stock (eflexs 입고 재고 백업)
- erp_goods_keyword (ERP 상품 키워드 매핑)
- currency_exchange_rate (환율 정보)
- md_keychron_sheet (키크론 상품 관리 시트)
- brand_view (브랜드 뷰)
- goods_view (상품 뷰)
- option_view (옵션 뷰)
- product_view (제품 통합 뷰)

### 02_order.md
- erp_order (발주 테이블)
- erp_order_item (발주 품목)
- erp_order_payment_history (발주 결제 이력)
- erp_order_recv (B2C 수주)
- erp_order_recv_item (B2C 수주 품목)
- erp_order_recv_memo (수주 메모)
- erp_order_recv_biz (B2B 수주 거래처)
- erp_order_recv_biz_item (B2B 거래처 취급 상품)
- erp_order_recv_biz_recipient (B2B 거래처 배송지)
- erp_customer_order_list (고객 주문 목록)
- erp_customer_order_partner_list (고객 주문 거래처 마스터)
- erp_customer_order_product_list (고객 주문 상품 마스터)
- erp_purchase_order (발주 PO 관리)
- erp_coupang (쿠팡 SKU 마스터)
- erp_coupang_order (쿠팡 발주 이력)
- non_sabangnet_b2c_orders (사방넷 외 B2C 주문)
- erp_target_order (브랜드별 매출 목표)
- erp_target_amount (파트너별 발주 목표 금액)
- erp_customer_month_targert_sales (월별 브랜드 매출 목표)

### 03_stock.md
- erp_stock (ERP 재고 - 개별 시리얼 단위)
- product_available_stock (상품별 가용재고 - 실시간 캐시)
- erp_stock_realtime (ERP 실시간 재고 집계)
- erp_stock_summary (ERP 재고 요약 - 일별 스냅샷)
- erp_stock_adjustment (재고 조정 이력)
- erp_refurb_stock (리퍼브 재고)
- erp_refurb_export_request (리퍼브 출고 요청)
- branch_list (오프라인 지점 목록)
- branch_product_list (지점 취급 상품 목록)
- branch_stock (지점 재고)
- branch_stock_pending (지점 입고 대기)
- branch_supplier_codes (지점 매입처 코드)
- branch_gift_rule (지점 사은품 규칙)
- stock_deduction_log (재고 차감 이력)
- coupang_inventory_daily_log (쿠팡 일별 재고 스냅샷 로그)
- pure_stock_view (순수 재고 뷰)
- pure_refurb_inventory_view (순수 리퍼브 재고 뷰)
- refurb_inventory_pivot_view (리퍼브 재고 피벗 뷰)
- stock_inventory_view (재고 인벤토리 뷰)

### 04_export_sales.md
- erp_export_log (출고 이력 로그)
- erp_export_request (출고 요청)
- erp_export_schedule (출고 스케줄)
- erp_export_schedule_item (출고 스케줄 아이템)
- erp_item_shipment (입고/선적 관리)
- erp_advance_shipment (사전 선적 알림)
- tbnws_combined_packaging (합포 주문 정보)
- tbnws_packing_list (패킹리스트)
- erp_settlement (정산)
- erp_annual_sales (연간 브랜드별 매출)
- erp_annual_acc_sales (연간 악세서리별 판매량)
- erp_annual_model_sales (연간 모델별 판매량)
- erp_annual_alice_layout_sales (연간 레이아웃별 판매량)
- erp_annual_sales_memos (연간 매출 메모)
- erp_import_log (입고 이력 로그)
- export_scheduled_ea_view (출고 예정 수량 뷰)

### 05_crm.md
- crm_as (A/S 테이블)
- crm_as_alimtalk_template (A/S 알림톡 템플릿)
- crm_as_export (A/S 출고 관리)
- crm_as_export_product (A/S 출고 제품)
- crm_as_history (A/S 히스토리 테이블)
- crm_as_memo (A/S 메모 테이블)
- crm_as_notify (A/S 알림 발송 내역)
- crm_as_product (A/S 상품 테이블)
- crm_call_info (전화 수신 정보)
- crm_call_memo (전화 상담 메모)
- crm_refund (교환/반품)
- crm_report (A/S 리포트 테이블)
- crm_return_management (회수 관리)
- crm_return_product (회수 관리 대상 제품)
- customer (고객 DB)
- customer_refund (고객 환불 기록)
- dw_grade (제품 등급/가용월 데이터)
- erp_as (ERP A/S 테이블)
- erp_blacklist (블랙리스트)
- erp_smartStore (스마트스토어 주문 상태)

### 06_naver.md
- naver_smartstore_orders (스마트스토어 주문)
- naver_smartstore_order_daily (스마트스토어 일별 주문 집계)
- naver_commerce_order (네이버 커머스 API 주문)
- naver_commerce_product_info (네이버 커머스 상품 정보)
- naver_smartstore_product_daily (스마트스토어 상품별 일별 실적)
- naver_smartstore_qna (스마트스토어 Q&A)
- naver_smartstore_settlement_daily (스마트스토어 일별 정산)
- naver_store_customer_inquiries (고객 문의)
- naver_store_inquiry_greetings_template (문의 답변 인사말 템플릿)
- naver_store_inquiry_issue (문의 이슈 관리)
- naver_store_product_qna (상품 문의)
- naver_reviews (네이버 리뷰)
- naver_ranking_tracker_list (네이버 랭킹 추적 목록)
- naver_shopping_ranking (네이버 쇼핑 랭킹)
- naver_store_review_crawler (리뷰 크롤러 설정)
- naver_store_review_item (크롤링 리뷰 아이템)
- naver_store_review_photo (크롤링 리뷰 사진)
- naver_tracking_item (추적 상품)
- naver_trend_daily (네이버 데이터랩 트렌드)
- smartstore_channel_detail (채널별 유입 상세)
- smartstore_keyword_daily (키워드별 일별 성과)
- smartstore_realtime_hourly (실시간 시간별 매출)
- smartstore_realtime_products (실시간 상품별 매출)
- brandstore_realtime_daily (브랜드스토어 실시간 현황)

### 07_finance_event.md
- finance_accounts (재무 게시판/장부)
- finance_estimate (견적서)
- finance_estimate_item (견적서 항목)
- finance_etc_remit (기타 송금 신청)
- finance_fixtures_remit (비품 구매 신청)
- finance_tax_invoice (세금계산서)
- finance_tax_invoice_biz (세금계산서 거래처)
- finance_tax_invoice_detail (세금계산서 품목 상세)
- event_best_review (베스트리뷰 이벤트)
- event_best_review_prize (베스트리뷰 사은품 목록)
- event_best_review_winner (베스트리뷰 당첨자)
- event_giveaway (기브어웨이 이벤트)
- event_giveaway_applicant (기브어웨이 신청자)
- event_giveaway_image (기브어웨이 이미지)
- event_giveaway_mission (기브어웨이 미션)
- event_message (이벤트 메세지 템플릿 릴레이션)
- event_prelaunching (프리런칭 이벤트)
- event_prelaunching_applicant (프리런칭 참여자)
- event_prelaunching_image (프리런칭 이미지)
- event_prelaunching_message (프리런칭 메시지 설정)
- event_prelaunching_message_log (프리런칭 메시지 발송 로그)
- event_prelaunching_option (프리런칭 옵션)
- coupon_list (쿠폰 관리)
- erp_free_gift (ERP 사은품 - 레거시)
- erp_free_gift_event (사은품 이벤트)
- erp_free_gift_target (사은품 이벤트 대상 상품)

### 08_wms_parts_misc.md
- wms_warehouse (창고 마스터)
- wms_relocation (재고 이동 작업)
- wms_relocation_item (재고 이동 품목)
- wms_relocation_history (재고 이동 히스토리)
- erp_parts (AS 부품)
- erp_parts_info (부품 정보 마스터)
- erp_parts_loaner (부품 대여 관리)
- erp_parts_location (부품 보관 위치 마스터)
- erp_parts_status (부품 상태 마스터)
- erp_serial (시리얼 번호 목록)
- erp_serial_registration (정품등록 정보)
- erp_serial_registration_manual (수동 정품등록 신청)
- erp_partner (거래처/채널 파트너)
- task_list (업무 목록)
- task_assignment (업무 할당)
- tbnws_sabangnet_order (사방넷 주문 정보)
- tbnws_sabangnet_preOrder (사방넷 사전 주문)
- tbnws_sabangnet_fault_order (사방넷 오류 주문)
- tbnws_sabangnet_gtgear_order (사방넷 GT기어 주문)
- tbnws_prelaunching_list (사전 출시 목록)
- tbnws_prelaunching_list_item (사전 출시 아이템)
- tbnws_prelaunching_object_item (사전 출시 옵션 집계)
- tbnws_prelaunching_user (사전 출시 참여자)
- tbnws_certificate (인증서 관리)
- alimtalk_template (카카오 알림톡 템플릿)
- alimtalk_template_button (알림톡 템플릿 버튼)
- message_adcall (애드콜 목록)
- adcall_alimtalk_rel (애드콜-알림톡 릴레이션)
- message_adcall_item (애드콜 대상 항목)
- as_board_rel (AS-게시판 릴레이션)
- as_material_parts_rel (AS 부품 사용 릴레이션)
- as_material_stock_rel (AS 제품 사용 릴레이션)
- crawler_coupang_price (쿠팡 가격 크롤링 대상)
- crawler_coupang_price_history (쿠팡 가격 변동 히스토리)
- gtgear_partner_list (GT기어 거래처 목록)

