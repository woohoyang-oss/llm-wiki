# TBNWS Admin Frontend - 메뉴별 도메인 & API Prefix 매핑

## 인프라 개요

| 서버 | Base Path | 설명 |
|------|-----------|------|
| Main API | `${VITE_SERVER_URL}/api` | 메인 백엔드 (`/src/constant/url/BASE_URL.ts`) |
| Python Server | `${VITE_PYTHON_SERVER_URL}` | AI/분석용 백엔드 |
| N8N Webhooks | `https://n8n.tbnws.co.kr/webhook/*` | 워크플로우 자동화 |

---

## 도메인별 API Prefix 요약표

| # | 도메인 | Route | Primary API Prefix | URL 상수 파일 |
|---|--------|-------|-------------------|---------------|
| 0 | Home | `/` | `/api/widget/*`, `/api/calendar/*` | `widget.ts`, `api.ts`, `user.ts` |
| 1 | 통합 ERP | `/erp` | `/api/erp/*` | `erp.ts`, `OrderUrl.ts`, `erp/serial.ts` |
| 2 | WMS | `/wms` | `/api/wms/*` | `wms/wms.ts`, `wms/warehouse.ts`, `wms/location.ts`, `wms/tracking.ts` |
| 3 | 이벤트 관리 | `/event` | `/api/event/*` | `event/event.ts`, `prelaunching.ts` |
| 4 | 메세지 관리 | `/message` | `/api/message/*` | `message/message.ts` |
| 5 | B2B CRM | `/b2bcrm` | `/api/v1/b2bcrm/*` | `B2BUrl.ts` |
| 6 | B2C CRM | `/b2ccrm` | `/api/b2ccrm/*` | `b2ccrm/b2ccrm.ts`, `b2ccrm/blog.ts` |
| 7 | 인사 정보 | `/hr` | `/api/hr/*`, `/api/vacation/*`, `/api/overtime/*`, `/api/businessTrip/*`, `/api/v1/hr/welfare/*` | `hr/*.ts`, `employeeState.ts` |
| 8 | 재무 관리 | `/finance` | `/api/v1/finance/*`, `/api/finance/*` | `finance.ts`, `OrderUrl.ts` |
| 9 | 챗봇 | `/chatbot` | `/api/send/*`, `/api/tpt/*` | `send/ai.ts`, `send/alimtalk.ts`, `send/send.ts` |
| 10 | 설정 | `/settings` | `/api/settings/*` | `settings/settings.ts`, `employeeState.ts` |
| 11 | 알림 | `/alert` | `/api/alert/*`, `/api/widget/taskList/*` | `hr/NewsUrl.ts`, `user.ts`, `widget.ts` |
| 12 | Test | `/test` | (테스트용) | - |
| 13 | 업체 발주 | `/biz` | `/api/biz/*` | `biz.ts` |
| 14 | 모니터링 | `/naver` | `/api/naver/*`, `/api/crawler/*` | `naver.ts`, `api.ts` |
| 15 | TPT 모바일 | `/tpt_` | `/api/open/TPTChat/v2` | `user.ts` |
| 16 | 브랜드스토어 | `/brandstore` | `/api/naver/*` (모니터링과 공유) | `naver.ts` |
| 17 | 풀필먼트 | `/fulfillment` | `/api/v1/fulfillment/*` | `fulfillment.ts` |
| 18 | 개발 | `/dev` | `/api/dev/*`, N8N, `/api/ai/*` | `dev.ts`, `ai.ts` |
| 19 | 쇼피파이 | `/shopify` | `/api/shopify/admin/*` | `shopify.ts` |

---

## 상세 페이지별 API 매핑

### 0. Home (`/`)

| 기능 | API Endpoints |
|------|---------------|
| 위젯 바로가기 | `/api/widget/shortcutList`, `insertShortcut`, `updateShortcut`, `deleteShortcut` |
| 할일 목록 위젯 | `/api/widget/taskList/personal`, `create`, `update`, `delete`, `toggleComplete`, `assignment/*`, `setRecurringFlag` |
| TPT 데일리 질문 | `/api/widget/tpt/dailyQuestion`, `create`, `update`, `unused` |
| 캘린더 | `/api/calendar`, `/api/calendar/notion`, `/api/calendar/google` |
| 알림 | `/api/getNotificationList`, `changeReadStatusOfNotification`, `archiveNotification`, `readAllNotifications` |
| 유저 정보 | `/api/memberInfo`, `pendingTasksCount`, `getEmployeeBirth` |

---

### 1. 통합 ERP (`/erp`) — API prefix: `/api/erp/*`

#### 상품 관리

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 상품 리스트 | `/erp/goodsList` | `getGoodsList`, `getProducts`, `createGoodsCode`, `createOptionCode`, `updateProductInfo`, `updateProductSize`, `updateOptionName`, `getProductImageList`, `getProductDetailsList`, `getProductComponentsList`, `getProductQC` |
| 리퍼브 관리 | `/erp/refurbList` | `addRefurbList`, `getRefurbList`, `exportRefurb`, `deleteRefurbInfo`, `updateRefurbInfo`, `getRefurbInvoiceList`, `getRefurbExportRequestList`, `addRefurbExportRequestList` |
| 부품 관리 | `/erp/parts` | `getParts`, `getPartsBrands`, `getPartsGoods`, `getPartsOptions`, `getPartsList`, `insertPartsInfo`, `deletePartsInfo` |
| 임대품 관리 | `/erp/loaner` | `loaner`, `loaner/seq`, `loanableParts` |
| 사은품 관리 | `/erp/freeGiftEventList` | `selectFreeGiftEventList_DT`, `selectFreeGiftEventInfo`, `addFreeGiftEventInfo`, `selectFreeGiftEventTargetList_DT` |
| 시리얼 관리 | `/erp/serial` | **prefix: `/api/serial/*`** — `search`, `batch`, `export`, `toggle-registrable`, `manual/*`, `warranty/open/search` |
| 품절상품관리 | `/erp/soldout` | `soldOut` |
| 쿠팡 상품 리스트 | `/erp/coupangSkuList` | `coupangList`, `coupangList/status`, `coupangList/excel`, `coupangList/coupangCheckForm` |

#### 발주/입출고

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 상품 발주 현황 | `/erp/goodsOrder` | `goodsOrder`, `goodsOrder/excel`, `goodsOrder/info`, `goodsOrder/delivery`, `goodsOrder/arrive`, `goodsOrder/clearanceConfirm`, `goodsOrder/importConfirm`, `goodsOrder/header` |
| 발주 준비 (PO) | `/erp/goodsOrder/po` | `purchaseOrder`, `purchaseOrder/excel`, `purchaseInvoice`, `goodsOrder/po` |
| 발주 외 상품 입고 | `/erp/external/order` | `externalOrder`, `externalOrder/excel`, `selectExternalPartner` |
| 입고 관리 | `/erp/import` | `selectImportList_DT_V2`, `confirmImport`, `completeImport` |
| 상품 출고 요청 | `/erp/export/exportRequest` | `exportRequestDT`, `createExportRequest`, `createMultiExportRequest`, `exportMultiProduct`, `deleteExportRequest` |
| 출고 관리 | `/erp/export` | `selectExportSchedule_DT`, `confirmExportSchedule`, `selectExportScheduleDetail_DT`, `getExportPostProcess_DT` |
| 출고 자동화 | `/erp/orderAutomatic` | `orderAutomatic`, `orderAutomatic/excel`, `orderAutomatic/product/excel`, `orderAutomatic/picking/excel`, `gtgear/warehouse/*` |
| 송장 등록&조회 | `/erp/invoice` | `getInvoiceList`, `createInvoiceList`, `updateInvoiceList` |
| 발주 회의 시트 | `/erp/orderMeetingSheet` | `goodsOrder/list`, `selectCoupangOrderCheckList_DT` |

#### 재고

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 재고 상황 대시보드 | `/erp/goodsStockDashboard` | `getGoodsStockDashboard` |
| 재고 현황 | `/erp/stock` | `getStockList`, `v2/getStockList`, `stock`, `getRemainStocksImportDates`, `getStocksByInboundInfo` |
| 상품 입출고 & 재고 현황 | `/erp/goodsStatus` | `getStockList`, `searchExportList` |
| 시리즈 재고 | `/erp/seriesStock` | `getSeriesStockList` |
| 풀필먼트 재고 | `/erp/eflexStockList` | `eflexStockList` |
| 재고 조정 | `/erp/stockAdjustment` | `stockAdjustment` |
| 재고 흐름 관리 | `/erp/productHistory` | `productHistory` |
| 재고 대시보드 | `/erp/supplyChainDashboardPage` | ERP 재고 관련 집계 API |

#### 가격/판매/분석

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 상품 가격 리스트 | `/erp/goodsPriceList` | `getGoodsPriceList`, `updateGoodsPriceListRowValue`, `insertMultipleDiscountPrice` |
| 채널 별 가격 리스트 | `/erp/channelPriceList` | `getChannelList`, `getChannelPriceList`, `updateChannelPrice`, `getChannelPriceInfo` |
| 판매량 분석 | `/erp/salesAnalysis` | `getSalesAnalysisList`, `getPartnerSalesList` |
| 매출 집계 | `/erp/salesSummary` | `getSalesSummary`, `addSalesData`, `getSalesSummaryDaily`, `getAnnualSalesList`, `getBrandSaleData`, `getMonthlySalesTargetData` |
| 판매독려상품 | `/erp/salesEncouragement` | `getSalesEncouragementListV2`, `getBrandList` |
| 신규/재입고 현황 | `/erp/newRestockStatus` | `getImportList_V2` |
| 미납금 현황 | `/erp/unPayStatus` | `unPayStatus_DT` |
| 키크론 출고 수량 | `/erp/exportEaList` | `getExportEaList`, `getGoodsSeriesList` |
| 재고 분석 | `/erp/inventory-analysis` | **N8N**: `n8n.tbnws.co.kr/webhook/erp-analysis` |
| 판매 분석 | `/erp/inventory-analysis-v2` | **N8N** + `/api/ai/prompt` |
| 발주 예측 대시보드 | `/erp/orderForecast` | ERP 발주 관련 집계 API |
| 판매 대시보드 V2 | `/erp/v2/saleDashboard` | ERP 매출 관련 집계 API |

---

### 2. WMS (`/wms`) — API prefix: `/api/wms/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 창고 관리 | `/wms/warehouse` | `getWarehouseList`, `addWarehouse`, `updateWarehouseInfo`, `deleteWarehouse`, `getCurrentLocStock` |
| 상품 입고 | `/wms/inbound` | `inbound`, `inbound/detail` |
| 상품 출고 | `/wms/export` | `operateLogistics` |
| 로케이션 이동 | `/wms/relocation` | `operateRelocation`, `relocationStocks`, `batchRelocationStocks`, `getRelocationOperationList`, `completeRelocation`, `cancelRelocation` |
| 히스토리 | `/wms/history` | `getLocationHistory`, `searchLocationHistory` |
| (로케이션 관리) | (공통 기능) | `getLocationList`, `locTree`, `addLoc`, `modLoc`, `delLoc`, `gatherProductsByEA` |

---

### 3. 이벤트 관리 (`/event`) — API prefix: `/api/event/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 기브어웨이 | `/event/giveaway` | `giveaway`, `giveaway/{seq}/applicants` |
| 프리런칭 | `/event/prelaunching` | `prelaunching`, `prelaunching/{seq}`, `prelaunching/{seq}/basic`, `prelaunching/{seq}/schedule`, `prelaunching/{seq}/page`, `prelaunching/{seq}/applicants`, `prelaunching/check-id`, `selectPrelaunchingStats` |
| 베스트리뷰 | `/event/best-review` | `best-review`, `best-review/{seq}`, `best-review/{seq}/winners`, `best-review/prizes` |
| 단축 URL 현황 | `/event/shortUrl` | `shortUrl` 관련 |
| 쿠폰 관리 | `/event/coupon` | `coupon`, `coupon/excel` |
| 메세지 중복 제거 | `/event/segmentedMessage` | `message`, `message/seq` |
| 체험단 관리 | `/event/experience` | `getExperienceList`, `getExperienceInfo`, `saveExperienceInfo`, `deleteExperienceInfo`, `getExperienceResultList`, `applyExperience` |

---

### 4. 메세지 관리 (`/message`) — API prefix: `/api/message/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 문자메세지 | `/message/sms` | `sendSMS`, `getSMSHistory`, `getSMSSenderList`, `getSMSTemplateList`, `cancelReservationSMS/seq`, `sendImmediately/seq` |
| 이메일 | `/message/email` | `sendEmail`, `sendGmail`, `sendGmailUsingSendAs`, `getEmailHistory`, `getEmailSenderList` |
| 템플릿 관리 | `/message/template` | `messageTemplate`, `messageTemplate/seq` |
| 변동 메세지 | `/message/variable` | `variables`, `eventMessage`, `eventMessage/seq`, `getEventMessageVariables` |
| 알림톡 | `/message/alimtalk` | `alimtalk` |
| 애드콜 | `/message/adCall` | `adcall`, `adcall/sendStatistics` |

---

### 5. B2B CRM (`/b2bcrm`) — API prefix: `/api/v1/b2bcrm/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 외부 발주 현황 | `/b2bcrm/recvOrder` | `recvOrder`, `recvOrder/info`, `recvOrder/detail`, `recvOrder/excel`, `recvOrder/pdf`, `recvOrder/delivery`, `recvOrder/payment`, `recvOrder/recipient`, `recvOrder/code`, `recvOrder/brand`, `recvOrder/goods`, `recvOrder/option`, `recvOrder/exportList`, `recvOrder/taxStatus`, `recvOrder/batchPayment` |
| 외부 발주처 리스트 | `/b2bcrm/recvOrderBizList` | `recvOrderBiz`, `recvOrderBizItem`, `recvOrderBiz/memo`, `recvOrderBizList/info`, `allBizInfo`, `selectOrderBiz`, `uploadRecvBusinessOverview` |
| 미수금 현황 | `/b2bcrm/payStatus` | `payStatusList` |
| 견적서 발행내역 | `/b2bcrm/estimateIssue` | `estimateIssue`, `estimateIssue/requestExport`, `estimateIssue/taxInvoice`, `estimateIssue/pdf`, `estimateIssue/excel` |
| 지점 관리 | `/b2bcrm/branches` | `branch/bizBranchList`, `branch/branchList`, `branch/branchInfo`, `branch/supplierCodeList`, `branch/branchProductList`, `branch/branchProduct`, `branch/branchStock/*`, `branch/branchSettlement`, `branch/branchProductPrice`, `branch/warehouseStock`, `branch/externalOrder`, `branch/giftRule/*` |

---

### 6. B2C CRM (`/b2ccrm`) — API prefix: `/api/b2ccrm/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 전체 상담 내역 | `/b2ccrm/allMemo` | `selectMemoCrmList_DT`, `getMemoStatus`, `insertCrmMemo`, `updateFinishFlag` |
| 전체 젠데스크 문의 | `/b2ccrm/zendeskList` | `getZendeskList`, `getZendeskStats` |
| 고객 환불 요청 | `/b2ccrm/customerRefund` | `customerRefund`, `customerRefund/isCompleted` |
| 리포트 현황 | `/b2ccrm/report` | `report` |
| 상품 A/S | `/b2ccrm/as` | `as`, `createAS`, `getASProductInfo`, `getASHistory`, `searchASProduct`, `sendASAlimtalk`, `getEngineerList`, `as/{asSeq}/feedback` |
| A/S 출고 관리 | `/b2ccrm/asExport` | `asExport`, `asExport/{seq}`, `asExport/seq/product`, `asExport/excel` |
| 상담 내역 | `/b2ccrm/selectmemo` | `selectMemoInfo`, `selectMemoExportList_DT`, `selectMemoSmsList_DT`, `selectCallList_DT`, `selectMemoAsList_DT`, `selectMemoZendeskList_DT`, `selectMemoForumList_DT`, `selectSerialList_DT`, `selectSubscribeList_DT`, `selectGiveawayList_DT`, `selectChatbotLogList` |
| 정품등록 관리 | `/b2ccrm/warranty` | `selectSerialList_DT` + **cross**: `/api/serial/*` |
| 회수 관리 | `/b2ccrm/returnManagement` | `returnManagement`, `returnManagement/excel`, `returnManagement/seq/returnRequest` |
| A/S 비용 현황 | `/b2ccrm/asCostStatus` | `crmCost` |
| 교환&반품 | `/b2ccrm/refund` | `refund`, `refund/seq` |
| 사방넷 제외 채널 | `/b2ccrm/nonSabang` | **cross**: `/api/erp/*` |
| 블로그 관리 | `/b2ccrm/blog` | **별도 prefix**: `/api/blog/brandsite/*` |

---

### 7. 인사 정보 (`/hr`) — API prefix: `/api/hr/*`, `/api/vacation/*`, `/api/overtime/*`, `/api/businessTrip/*`, `/api/v1/hr/welfare/*`

#### HR 관리 (`/hr/manage`) 탭별

| Sub-tab | Route | API Prefix |
|---------|-------|------------|
| 근태 관리 | `/hr/manage/attendance` | `/api/hr/selectAttendanceManage_DT`, `selectCommuteStatusPersonal_DT`, `selectTodayWorkStartTime`, `updateCommuteStatusInfo` |
| 휴가 관리 | `/hr/manage/vacation` | `/api/vacation/*` — `list/all`, `monthlyDetail`, `dailyDetail`, `requestVacation`, `decision`, `cancel`, `addDays`, `DynamicList`, `deductionHistory` 등 |
| 시간외 근무 | `/hr/manage/overtime` | `/api/overtime/*` — `statusFinderOvertime`, `apply`, `status_DT`, `selectOvertimeApply_DT`, `decision`, `dynamicList` 등 |
| 출장 관리 | `/hr/manage/businessTrip` | `/api/businessTrip/*` — `selectBusinessTripStatus_DT`, `requestBusinessTrip`, `decisionBusinessTrip`, `dynamicList` 등 |
| 복리후생 | `/hr/manage/welfare` | `/api/v1/hr/welfare/*` — `statistics`, `create`, `list`, `detail`, `decision`, `remit`, `group/*`, `manage/*` 등 |
| 일정 관리 | `/hr/manage/schedule` | `/api/calendar/*` — `holidayList`, `tobeSchedule`, `setHoliday`, `unsetHoliday` |

#### 기타 HR 페이지

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 개인 근무 기록 | `/hr/myWorkOverview` | `/api/hr/*`, `/api/vacation/personalStats`, `/api/overtime/status_DT/personal`, `/api/businessTrip/personalStats` |
| 재직증명서 | `/hr/employmentCertificate` | `/api/settings/certificate_DT`, `certificate`, `selectCertificateInfo` |
| 경력증명서 | `/hr/manage/workExperience` | `/api/hr/certificate/experience`, `experience/createDoc` |
| 좌석배치도 | `/hr/seatingChart` | `/api/settings/getSeatingChart`, `updateSeatingChart` |
| 직원 현황 | `/hr/employeeState` | `/api/settings/employeeState`, `employee/statistic`, `selectEmployeeDetail`, `leaveEmployee` |
| 비상연락망 | `/hr/emergencyContact` | `/api/settings/getEmergencyContactList` |
| 급여 정산 통계 | `/hr/payrollSettlements` | `/api/overtime/payrollSettlementData`, `payrollSettlementUpset` |
| 조직 관리 | `/hr/organization/management` | `/api/hr/org`, `org/department`, `org/part`, `org/position`, `org/position/level`, `org/moveTo/*` |
| 조직도 | `/hr/organization/chart` | `/api/hr/org` |
| 휴직 관리 | (HR 내 기능) | `/api/hr/leave/*` — `history`, `create`, `extend`, `complete`, `cancel` |

---

### 8. 재무 관리 (`/finance`) — API prefix: `/api/v1/finance/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 기타 송금 요청 | `/finance/requestRemit` | `etc`, `etc/approve`, `etc/cancel`, `etc/reject`, `etc/accountList` |
| 구매 요청 | `/finance/fixtures` | `fixtures`, `fixtures/approval` |
| 세금계산서 발행 | `/finance/taxInvoice` | `taxInvoice`, `taxInvoice/info`, `taxInvoice/registIssue`, `taxInvoice/cancel`, `taxInvoice/combine`, `selectInvoiceeList_DT` |
| 기타 장부 자료 | `/finance/accountsData` | `accountData` |
| 상품 발주 정산 | `/finance/calculate` | **별도 prefix**: `/api/finance/calculateOrder`, `calculateOrder/paymentHistory` |

---

### 9. 챗봇 (`/chatbot`) — API prefix: `/api/send/*`, `/api/tpt/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| AI 설정 | `/chatbot/ai` | `/api/send/getChatBotResponseList`, `getChatBotLogList`, `getChatbotStatistic`, `updateChatBotResponse`, `getPrompt`, `changeChatbotPrompt`, `allFaqDocs`, `allProducts`, `allEvents`, `chatBotHistoryByPhone`, `chatbotSimulator`, `getAllMouse`, `selectAllAccessories_DT`, `getUserDic`, `getForecastInfo` |
| 알리고 템플릿 | `/chatbot/addcall` | `/api/send/getAlimtalkTemplateList`, `updateAlimtalkTemplate`, `createNewAlimtalkTemplate`, `deleteAlimtalkTemplate`, `alimtalkTemplate/{templateCode}` |
| TPT | `/chatbot/tpt` | `/api/tpt/history` |

---

### 10. 설정 (`/settings`) — API prefix: `/api/settings/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 내 정보 관리 | `/settings/mypage` | `myPage`, `/api/memberInfo` |
| 권한 관리 | `/settings/permission` | `membersAuthorizedMenuList`, `menusAuthorizedMemberList` |
| (위젯 관리) | (공통) | `widgetList`, `updateWidgetIsActive`, `settingWidgetLayout`, `getWidgetLayout` |
| (프리셋) | (공통) | `presetList`, `createPreset`, `deletePreset` |
| (테이블 설정) | (공통) | `tablePreference` |
| (네이버 트래킹) | (공통) | `naver/tracking` |
| (컴포넌트 인증) | (공통) | `checkAuthForComp` |

---

### 11. 알림 (`/alert`) — API prefix: `/api/alert/*`, `/api/widget/taskList/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 공지사항 | `/alert/notice` | `/api/alert/notice/list`, `unReadList`, `notVisibleList`, `selectNoticeInfo`, `createNotice`, `updateNotice`, `deleteNotice`, `uploadImage` |
| 전체 알림 | `/alert/notifications/list` | `/api/getNotificationList`, `changeReadStatusOfNotification`, `archiveNotification`, `readAllNotifications`, `sendNotifications`, `deleteNotification` |
| 할일 목록 | `/alert/taskList` | `/api/widget/taskList/personal`, `create`, `update`, `delete`, `toggleComplete`, `recurring`, `assignment/*` |

---

### 13. 업체 발주 시스템 (`/biz`) — API prefix: `/api/biz/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 업체 발주 시스템 | `/biz/stock` | `getBizStockList`, `getBizOrderList`, `getBizOrderInfo`, `getBizExportList`, `getRecvOrderInfo`, `getRecvOrderItemList`, `insertNewBizOrder` |

---

### 14. 모니터링 관리 (`/naver`) — API prefix: `/api/naver/*`, `/api/crawler/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 네이버 순위 모니터링 | `/naver/monitoring` | `/api/naver/getMonitoringList`, `naverRankingPreset`, `getMonitoringInfo`, `saveMonitoringInfo`, `getMonitoringChartData`, `getMonitoringTargetKeywordRanking` |
| 네이버 크롤러 | `/naver/naverCrawler` | `/api/naver/naverStoreReviewCrawler`, `naverStoreReviewCrawler/seq` |
| 쿠팡 가격 트래커 | `/naver/coupangPrice` | `/api/crawler/coupang/price` |

---

### 16. 브랜드스토어 정보 (`/brandstore`) — API prefix: `/api/naver/*` (모니터링과 공유)

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 브랜드 스토어 실시간 | `/brandstore/realtime` | `realtime/status`, `realtime/compare`, `realtime/predict` |
| 고객 통계 | `/brandstore/stats/customer` | `customer/stat`, `repurchase/customer/stat` |
| 광고 통계 | `/brandstore/stats/ad` | `naver/ad`, `ad/monthly/result`, `ad/action/execute`, `ad/action/logs` |
| 브랜드스토어 모니터링 | `/brandstore/monitoring` | `getMonitoringList` |
| 네이버 순위 모니터링 | `/brandstore/naverMonitoring` | `naverRankingPreset` |
| 네이버 고객 문의 | `/brandstore/inquiry` | `{storeName}/customer-inquiries`, `inquiryDetailInfo`, `{storeName}/product-qna`, `productQna/generateAnswer` |
| 네이버 고객 주문 | `/brandstore/order` | `{storeName}/order/state`, `{storeName}/order/{orderId}/state`, `order`, `setShippingAddress` |
| 네이버 리뷰 | `/brandstore/reviews` | `reviews`, `reviews/confirm` |

---

### 17. 풀필먼트 (`/fulfillment`) — API prefix: `/api/v1/fulfillment/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 풀필먼트 상품 | `/fulfillment/product` | `product`, `product/refetch`, `product/excel` |
| 풀필먼트 주문 | `/fulfillment/order` | `order`, `order/exclude` |
| 풀필먼트 히스토리 | `/fulfillment/history` | `history` |

---

### 18. 개발 (`/dev`) — API prefix: `/api/dev/*`, N8N, `/api/ai/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 디자인 가이드 | `/dev/designGuide` | N/A (UI only) |
| 재고 분석 대시보드 | `/dev/inventoryAnalysis` | N8N: `n8n.tbnws.co.kr/webhook/erp-analysis`, `/api/ai/prompt` |
| 테스트 | `/dev/test` | 테스트 엔드포인트 |
| 개발뷰 | `/dev/view` | `view/list`, `view/cache/evict` |

---

### 19. 쇼피파이 관리 (`/shopify`) — API prefix: `/api/shopify/admin/*`

| Page | Route | Key API Endpoints |
|------|-------|-------------------|
| 1:1 문의 관리 | `/shopify/inquiry` | `inquiry`, `inquiry/{seq}/answer` |
| 재입고 신청 관리 | `/shopify/restock` | `restock`, `restock/{seq}/notify` |
| FAQ 관리 | `/shopify/faq` | `faq`, `faq/{seq}` |

---

## Cross-Domain API 사용 현황

일부 페이지는 자기 도메인 외의 API를 호출합니다:

| Page | 소속 도메인 | Cross-Domain API |
|------|------------|-----------------|
| B2C 정품등록 (`/b2ccrm/warranty`) | `/api/b2ccrm/*` | `/api/serial/*` (ERP) |
| B2C 사방넷 제외 (`/b2ccrm/nonSabang`) | `/api/b2ccrm/*` | `/api/erp/*` (ERP) |
| B2C 블로그 (`/b2ccrm/blog`) | `/api/b2ccrm/*` | `/api/blog/brandsite/*` (독립) |
| HR 재직증명서/직원현황 | `/api/hr/*` | `/api/settings/*` |
| HR 급여정산 | `/api/hr/*` | `/api/overtime/*` |
| 재무 발주정산 | `/api/v1/finance/*` | `/api/finance/calculateOrder/*` (비버전) |
| ERP 재고/판매분석 | `/api/erp/*` | N8N webhooks, `/api/ai/prompt` |
| 브랜드스토어 전체 | `/brandstore/*` | `/api/naver/*` (모니터링과 공유) |

---

## 공통/글로벌 API Endpoints

| Endpoint | 사용처 | 파일 |
|----------|--------|------|
| `/api/memberInfo` | Home, Settings, Auth | `user.ts` |
| `/api/memberList` | 다수 관리 기능 | `user.ts` |
| `/api/menuList` | 네비게이션/인증 | `menu.ts` |
| `/api/upload/file` | 파일 업로드 전체 | `common.ts` |
| `/api/productName` | 상품 검색 모달 | `common.ts` |
| `/api/v1/productList` | 상품 선택 전역 | `common.ts` |
| `/api/v1/exchangeRate` | ERP 발주, 재무 | `common.ts` |
| `/api/getEmployeeOfSelectBoxList` | HR, B2C, Finance | `user.ts` |
| `/api/selectMember` | 멤버 조회 전역 | `employeeState.ts` |

---

## URL 상수 파일 구조

```
src/constant/url/
├── BASE_URL.ts              # Base API URL (VITE_SERVER_URL + '/api')
├── api.ts                   # Calendar, Crawler endpoints
├── common.ts                # Upload, Product, ExchangeRate
├── menu.ts                  # Menu/Auth
├── user.ts                  # Member/User
├── widget.ts                # Widget endpoints
├── ai.ts                    # AI/N8N webhook endpoints
├── dev.ts                   # Development endpoints
├── naver.ts                 # Naver/BrandStore endpoints
├── shopify.ts               # Shopify admin endpoints
├── finance.ts               # Finance/Tax endpoints
├── fulfillment.ts           # Fulfillment endpoints
├── biz.ts                   # Business order endpoints
├── employeeState.ts         # Employee state endpoints
├── prelaunching.ts          # Pre-launch event endpoints
├── typing.ts                # Typing game endpoints
├── EventUrl.ts              # Event management
├── B2BUrl.ts                # B2B CRM endpoints
├── OrderUrl.ts              # Purchase/Order endpoints
├── erp/
│   ├── erp.ts               # Main ERP endpoints (322 lines)
│   └── serial.ts            # Serial number management
├── b2ccrm/
│   ├── b2ccrm.ts            # B2C CRM endpoints
│   └── blog.ts              # Blog endpoints
├── hr/
│   ├── hr.ts                # Core HR endpoints
│   ├── vacationUrl.ts       # Vacation
│   ├── overtime.ts          # Overtime
│   ├── BusinessTrip.ts      # Business trip
│   ├── welfare.ts           # Welfare benefits
│   ├── leaveUrl.ts          # Leave management
│   ├── Calendar.ts          # Calendar/Schedule
│   ├── certificate.ts       # Certificates
│   ├── NewsUrl.ts           # Internal news
│   └── sone.ts              # SONE system
├── wms/
│   ├── wms.ts               # Core WMS endpoints
│   ├── warehouse.ts         # Warehouse management
│   ├── location.ts          # Location management
│   └── tracking.ts          # Tracking endpoints
├── send/
│   ├── send.ts              # SMS/messaging
│   ├── alimtalk.ts          # KakaoTalk AlimTalk
│   └── ai.ts                # AI chatbot
├── message/
│   └── message.ts           # Email/SMS/Template
├── event/
│   └── event.ts             # Event management
├── settings/
│   └── settings.ts          # Settings endpoints
└── setting/
    ├── naver.ts              # Naver settings
    └── restrictComp.ts       # Restriction settings
```
