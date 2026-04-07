# 세션 로그: 2026-02-23 — 스마트스토어 통합

## 주요 작업

### 1. 네이버 스마트스토어 6탭 구축

Commerce API 연동으로 스마트스토어 매출 분석 기능 추가:

| 탭 | 데이터 소스 | 주요 기능 |
|----|-----------|----------|
| 대시보드 | DB + SA/GFA JOIN | KPI 4개, 매출 트렌드, 상품 비중, 광고-매출 연계 |
| 주문분석 | DB (naver_smartstore_orders) | 상태분포, 시간대/요일분석, 가격대분포, 클레임 유형 |
| 상품분석 | DB + Commerce API | TOP15, 파레토, 상품 카탈로그 (1566상품) |
| 문의분석 | DB (주문 클레임 데이터) | 클레임유형 도넛, 일별추이, 상품별 클레임률 |
| 판매자정보 | Commerce API (/v1/seller) | 셀러 정보, 채널 정보, 30일 평균 통계 |
| 정산분석 | DB (수수료 5.5% 추정) | 일별/월별 정산, 매출-수수료-정산 차트 |

**파일:**
- Backend: `app/routes/smartstore.py` (6개 신규 엔드포인트)
- Frontend: `ss-orders.js`, `ss-products.js`, `ss-inquiries.js`, `ss-seller.js`, `ss-settlement.js`
- 등록: `app.js`, `index.html` (사이드바 + 탭 컨테이너)

### 2. Commerce API 연동 이슈

**사용 가능 엔드포인트:**
- `GET /v1/seller/account` — 판매자 정보
- `GET /v1/seller/channels` — 채널 정보
- `POST /v1/products/search` — 상품 검색
- `GET /v2/products/channel-products/{id}` — 상품 상세
- 주문 API (last-changed-statuses, product-orders/query)

**사용 불가 (404):**
- 문의(inquiry) 관련 모든 엔드포인트
- 정산(settlement) 관련 모든 엔드포인트
- 판매자 주소록(address-books)
- 클레임(claims) 전용 엔드포인트

→ 문의/정산은 DB 내 주문 데이터 기반 추정으로 대체

### 3. 크론 스크립트 페이징 버그 수정

**문제:** `page` 파라미터로 페이징 → 무한루프 (같은 300건 반복)
**원인:** Commerce API는 커서 기반 페이징(`moreFrom`/`moreSequence`) 사용
**수정:** `scripts/smartstore_cron.py` — 커서 기반으로 전환
**추가:** 파일 잠금(fcntl), 지수 백오프(5→80초), HTTP 타임아웃(30초), 배치 크기 축소(100→50)

### 4. 검색트렌드 Y축 자동스케일링

**문제:** 전체 해제 후 한 키워드만 활성화하면 Y축 0-100 고정 → 가독성 나쁨
**수정:** `trends.js` — `afterDataLimits` 콜백으로 보이는 데이터셋 기준 Y축 자동 조정

### 5. EBS 스냅샷 자동화

- EC2-SSM-Role에 EBS 권한 인라인 정책 추가 (CloudShell 사용)
- `scripts/ebs_snapshot.sh` 작성 — 매일 04:00 실행, 7일 보관
- 스냅샷 태그: `AutoSnapshot=true`, `Instance=i-03115fe75a57de373`
- 첫 스냅샷 수동 실행 확인: `snap-050dee0e74e631dc6`

### 6. 스마트스토어 데이터 현황

| 계정 | 주문수 | 기간 | 상태 |
|------|-------|------|------|
| 키크론 | 764건 | 2023-12-14~2026-01-24 | 1/24 이후 신규주문 없음 |
| 지티기어 | 67건 | 2023-12-21~2026-02-13 | 정상 |

## 수정 파일 목록

| 파일 | 변경 유형 |
|------|----------|
| `app/smartstore_client.py` | 수정 (timeout 추가) |
| `app/routes/smartstore.py` | 수정 (6개 엔드포인트 추가) |
| `app/static/index.html` | 수정 (사이드바 5탭 + 컨테이너) |
| `app/static/js/app.js` | 수정 (탭 등록 + window 바인딩) |
| `app/static/js/tabs/trends.js` | 수정 (Y축 자동스케일) |
| `app/static/js/tabs/ss-orders.js` | 신규 |
| `app/static/js/tabs/ss-products.js` | 신규 |
| `app/static/js/tabs/ss-inquiries.js` | 신규 |
| `app/static/js/tabs/ss-seller.js` | 신규 |
| `app/static/js/tabs/ss-settlement.js` | 신규 |
| `scripts/smartstore_cron.py` | 수정 (커서 페이징 + 잠금 + 백오프) |
| `scripts/ebs_snapshot.sh` | 신규 |
