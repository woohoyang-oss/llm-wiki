# ToBe Networks 웹 페이지 & 데이터 분석 현황

> 최종 업데이트: 2026-02-28

---

## 1. 발주 예측 대시보드 (admin.tbnws.co.kr/erp/orderForecast)

### 개요
안전재고 14일 기준, 가중평균 판매량 기반으로 발주 필요 상품을 자동 예측하는 대시보드

### 핵심 KPI 카드
| 지표 | 현재 값 | 설명 |
|------|---------|------|
| 전체 상품 | 1,651개 | 관리 중인 전체 상품 |
| 긴급 발주 필요 | 246개 | 전체 대비 15% |
| 오늘 발주 필요 | 246건 | 발주추천일 ≤ 오늘 |
| 총 추천 발주량 | 24,820개 | - |
| 과잉재고 | 973개 | 리드타임 비례 기준 초과 |

### 필터/파라미터
- **브랜드**: 전체/개별 선택
- **리드타임**: 일 단위 설정
- **가중치**: 1주(50), 1개월(30), 3개월(20) = 100%
- **커버리지**: 2.3x (안전재고 배수)
- **표시 모드**: 보수적 / 표준 / 공격적 / 자동

### 상태 분류
| 상태 | 건수 | 의미 |
|------|------|------|
| 과잉 | - | 과다 재고 보유 |
| 긴급 | 246 | 즉시 발주 필요 |
| 발주중 | 45 | 이미 발주 진행 |
| 안전 | 300 | 적정 재고 |
| 주의 | 87 | 재고 감소 추세 |

### 차트
- **상태 분포**: 도넛 차트 (과잉/긴급/발주중/안전/주의)
- **브랜드별 현황**: 스택 바 차트 (Keychron, 로이체, 피너텍 등)
- **일평균 판매 TOP 10**: 가로 바 차트
- **발주 스케줄**: 주차별 상품 수 & 추천 발주량

### 데이터 테이블 컬럼
| 컬럼 | 설명 |
|------|------|
| 상태 | 긴급/주의/안전/과잉/발주중 |
| 우선순위 | 발주 긴급도 점수 |
| 상품코드 | G-XXXX-XXXX-XXXX 형식 |
| 브랜드 | Keychron, GTGear 등 |
| 상품명 | 상품명 |
| 옵션 | 상세 옵션 |
| MSRP | 권장소비자가 |
| 현재재고 | 현재 보유 수량 |
| 발주진행 | 현재 발주 진행 수량 |
| 입고예정 | 입고 예정 수량 |
| 출고예정 | 출고 예정 수량 |
| 가용재고 | 실제 가용 수량 |
| B2C 1주 | B2C 1주 판매량 |

### 연결 DB 테이블
- `erp_product_info` → 상품 정보
- `erp_stock` → 재고 현황
- `erp_order` + `erp_order_item` → 발주 진행 현황
- `erp_export_schedule` → 출고 예정
- `tbnws_sabangnet_order` + `erp_coupang_order` + `naver_commerce` → 판매 데이터 (판매량 계산)

---

## 2. ToBe Marketing Analysis (tbe.kr)

### 개요
네이버 SA(검색광고) + GFA(디스플레이광고) 통합 마케팅 분석 플랫폼

### 브랜드 선택
- 키크론 (Keychron)
- 지티기어 (GTGear)
- 에이퍼 (Aiper)

### 좌측 메뉴 구조

#### 통합 대시보드
| 메뉴 | 설명 |
|------|------|
| 오늘의 마케팅 | 통합 인사이트 + KPI |
| SA vs GFA 비교 | 검색광고 vs 디스플레이 비교 |
| 연도별 광고비 | 연간 광고비 추이 |
| 2026 키크론 보고서 | 연간 보고서 |

#### SA 검색광고
| 메뉴 | 설명 |
|------|------|
| 대시보드 | SA 전체 현황 |
| 일별비교 | 일별 성과 비교 |
| 캠페인 | 캠페인별 성과 |
| 광고그룹 | 광고그룹별 성과 |
| 키워드 | 키워드별 성과 |
| 검색트렌드 | 검색어 트렌드 |
| 통합분석 | SA 통합 분석 |
| 스코어링 | 키워드 효율 스코어링 |

#### GFA 디스플레이
| 메뉴 | 설명 |
|------|------|
| 대시보드 | GFA 전체 현황 |
| 캠페인 관리 | 캠페인 관리 |
| 퍼널 분석 | 전환 퍼널 |
| 수동 데이터 처리 | 수동 데이터 입력 |

#### 스마트스토어
| 메뉴 | 탭 파라미터 | 설명 |
|------|------------|------|
| 대시보드 | `?tab=ss-dashboard` | 스마트스토어 전체 현황 |
| 마케팅분석 | `?tab=ss-marketing` | 마케팅 성과 분석 |
| 실시간분석 | `?tab=ss-realtime` | 실시간 주문/매출 데이터 |
| 크로스채널 | `?tab=ss-crosschannel` | 채널간 비교 분석 |
| 주문분석 | `?tab=ss-orders` | 주문 상세 분석 |
| **상품분석** | **`?tab=ss-products`** | **상품별 매출/성과/카탈로그** |
| 문의분석 | `?tab=ss-inquiry` | 고객 문의 분석 |
| 정산분석 | `?tab=ss-settlement` | 정산 데이터 분석 |
| 판매자정보 | `?tab=ss-seller` | 판매자 기본정보 |

#### ★ 상품분석 탭 상세 (`?tab=ss-products`)

> **URL 패턴**: `https://tbe.kr/?tab=ss-products&account={account}&start={YYYY-MM-DD}&end={YYYY-MM-DD}`
> **프론트엔드 모듈**: `js/tabs/ss-products.js`
> **데이터소스**: `naver_smartstore_orders` (실거래가 1순위)

##### API 호출 구조

| API 엔드포인트 | 용도 | 응답 주요 필드 |
|---|---|---|
| `/api/smartstore/products/analysis` | 상품 성과 + 일별 데이터 | `product_perf`, `product_daily`, `weekly_summary`, `concentration`, `total_revenue` |
| `/api/smartstore/products/catalog?size=100` | 상품 카탈로그 | `products[]`, `total` |
| `/api/smartstore/data-freshness` | 데이터 최신성 | freshness 상태 |
| `fetchEvents()` (calendar-events.js) | 공휴일/이벤트 | 캘린더 이벤트 배열 |

##### API 응답 데이터 구조

**`/api/smartstore/products/analysis` 응답:**

```
{
  product_perf: [{                    ← 상품별 성과 (Top N 랭킹)
    product_no: string,               ← ★ 네이버 상품번호 (md_keychron_sheet.naver_product_code)
    product_name: string,             ← 네이버 등록 상품명
    order_cnt: number,                ← 주문건수
    quantity: number,                 ← 판매수량
    revenue: number,                  ← 매출액 (= SUM(total_price))
    cancel_cnt: number,               ← 취소건수
    cancel_rate: number               ← 취소율 (%)
  }],
  product_daily: [{                   ← ★ 상품별 일별 데이터 (판매추이 차트 원본)
    stat_date: "YYYY-MM-DD",          ← 날짜
    product_no: string,               ← 네이버 상품번호
    product_name: string,             ← 상품명
    revenue: number,                  ← 일 매출
    order_cnt: number                 ← 일 주문건수
  }],
  weekly_summary: [{                  ← 주간 요약
    week_start: "YYYY-MM-DD",
    revenue: number,
    order_cnt: number
  }],
  concentration: [{                   ← 매출 집중도 (파레토)
    revenue: number,
    cum_pct: number
  }],
  total_revenue: number               ← 기간 총매출
}
```

**`/api/smartstore/products/catalog` 응답:**
```
{
  products: [{
    name: string,                     ← 상품명
    status: "판매중" | "품절",
    sale_price: number,               ← 판매가
    discounted_price: number | null,  ← 할인가
    stock_quantity: number            ← 재고수량
  }],
  total: number                       ← 전체 상품수
}
```

##### 페이지 섹션 구성

| 섹션 | 차트 ID / 유형 | 데이터소스 필드 |
|------|---------------|----------------|
| KPI 카드 (등록 상품수, 매출 상위 상품, 기간 총매출, 매출 집중도) | 텍스트 | `product_perf`, `total_revenue`, `concentration` |
| 상품별 매출 TOP 15 | `sspTopChart` (수평 바) | `product_perf[].revenue` |
| 매출 집중도 (파레토) | `sspParetoChart` (바+라인) | `concentration[].revenue, cum_pct` |
| **★ 상품별 판매 추이 (Top 20)** | **`sspDailyTrendChart` (바+라인)** | **`product_daily[]` — 아래 상세** |
| 주간 매출 추이 | `sspWeeklyChart` (바+라인) | `weekly_summary[]` |
| 상품 상태 분포 | 파이 차트 | `catalog.products[].status` |
| 상품별 매출 비중 | 파이 차트 | `product_perf[].revenue` |
| 상품별 성과 (테이블) | 페이지네이션 테이블 | `product_perf[]` |
| 상품 카탈로그 (테이블) | 페이지네이션 테이블 | `catalog.products[]` |

##### ★ 상품별 판매 추이 차트 (`sspDailyTrendChart`) 상세 로직

**차트 유형**: Chart.js Mixed (bar + line)
**상품 선택**: `<select id="sspProductSelect">` — `value`는 `product_no` 또는 `'__ALL__'`

**렌더링 함수**: `_renderDailyTrend(productDaily, top20, events)`

```
[렌더링 흐름]
1. 전체 날짜 수집: productDaily.map(r => r.stat_date) → Set → sort
2. 라벨 변환: "YYYY-MM-DD" → "M/D" 형식
3. 어노테이션 생성: buildAnnotations(events, allDates) → 공휴일 표시
4. 모드 분기:
   ├─ '__ALL__': 상품별 스택 막대 + 전체 주문건수 + 전체 7일 평균
   └─ product_no: 개별 상품 매출 막대 + 주문건수 + 7일 평균
5. 차트 생성: new Chart(canvas, config)
6. select 이벤트: select.addEventListener('change', () => drawChart(select.value))
```

**7일 이동평균 계산:**
```javascript
// revenues = 일별 매출 배열
const ma7 = revenues.map((_, i) => {
    const start = Math.max(0, i - 6);
    const slice = revenues.slice(start, i + 1);
    return Math.round(slice.reduce((s, v) => s + v, 0) / slice.length);
});
```

**데이터셋 구성 (3개 레이어):**

| 레이어 | label | Chart 유형 | Y축 | 색상 | 스타일 | drawOrder |
|--------|-------|-----------|-----|------|--------|-----------|
| 매출 | `"매출"` | bar | `y` (좌) | `rgba(1,118,211,0.5)` | 단색 막대 | 2 |
| 주문건수 | `"주문건수"` | line | `y1` (우) | `#2E8449` | 실선, tension=0.3, pointRadius=2 | 1 |
| 매출 7일 평균 | `"매출 7일 평균"` | line | `y` (좌) | `#FE9339` | 점선 [5,3], tension=0.4, pointRadius=0 | 0 |

**축(Scales) 구성:**

| 축 ID | 타입 | 위치 | 타이틀 | 특이사항 |
|-------|------|------|--------|---------|
| `x` | category | bottom | - | 날짜 라벨 "M/D" |
| `y` | linear | left | 매출 | beginAtZero=true, '__ALL__' 시 stacked=true |
| `y1` | linear | right | 주문건수 | beginAtZero=true, grid 비표시 |

**어노테이션 (공휴일/이벤트 표시):**
- `buildAnnotations(events, allDates)` → `calendar-events.js` 모듈에서 import
- 공휴일: 빨간 점선 (`rgba(186,5,23,0.22)`) + 라벨 (예: 신정, 설날, 삼일절)
- 이벤트: 파란 점선 (`rgba(1,118,211,0.22)`) + 라벨 (예: 밸런타인데이)
- 라벨 태그: `type: 'label'`, 테두리 `#666`

**product_no → 상품명 매핑 (셀렉트 옵션 생성):**
```javascript
// top20 = product_perf 상위 20개 (revenue DESC)
<option value="__ALL__">전체 Top 20 합계</option>
${top20.map(p =>
    `<option value="${p.product_no}">
        ${(p.product_name || '').slice(0, 30)} (${fmtWon(p.revenue)})
    </option>`
).join('')}
// 예: <option value="10310833443">키크론 K10 PRO SE 레트로 무선 기계식 키보드 (350,777,930)</option>
```

**개별 상품 데이터 필터링:**
```javascript
const prodRows = productDaily.filter(r => String(r.product_no) === String(selectedNo));
const byDate = {};
prodRows.forEach(r => {
    byDate[r.stat_date] = { revenue: r.revenue || 0, order_cnt: r.order_cnt || 0 };
});
```

##### 상품별 성과 테이블

| 컬럼 | 필드 | 설명 |
|------|------|------|
| 상품명 | `product_name` | 네이버 등록 상품명 |
| 주문수 | `order_cnt` | 총 주문건수 |
| 판매수량 | `quantity` | 총 판매수량 |
| 매출 | `revenue` | 총 매출액 |
| 취소수 | `cancel_cnt` | 취소 건수 |
| 취소율 | `cancel_rate` | 취소율 (%) |
| 검색: `상품명 검색...` | `product_name` LIKE | 클라이언트 사이드 필터 |
| 페이지네이션 | 20건/페이지 | 총 N건 중 1-20건 |

##### 상품 카탈로그 테이블

| 컬럼 | 필드 | 설명 |
|------|------|------|
| 상품명 | `name` | 정렬 가능 (↕) |
| 상태 | `status` | 판매중 / 품절 뱃지 |
| 판매가 | `sale_price` | 정렬 가능 (↕) |
| 할인가 | `discounted_price` | 할인가 (없으면 `-`) |
| 재고 | `stock_quantity` | 정렬 가능 (↕) |

### 통합 마케팅 현황 KPI
| 지표 | 값 | 설명 |
|------|-----|------|
| 총 광고비 | 5,230,589원 | SA 448,481 + GFA 4,782,108 |
| 구매 전환매출 | 118,295,890원 | SA전환 49,791,900 + GFA전환 68,503,990 |
| 장바구니 매출 | 460,733,240원 | GFA 전체전환 529,237,230 중 87% |
| 구매 ROAS | 2,262% | 효율 양호 |
| 실제 매출(스토어) | 190,578,230원 | 주문 2,740건 |
| 비즈머니 잔액 | 3,423,833원 | 예상 잔여 4일 |
| 월 예상 광고비 | 26,023,838원 | SA 2,113,298 + GFA 23,910,540 |
| 비효율 절감 가능 | 380,459원/월 | SA 140,404 + GFA 240,055 |

### 기간 성과 분석 (YoY)
| 지표 | 증감 |
|------|------|
| 총 비용 YoY | ▲ 121% |
| 전환매출 YoY | ▲ 6,809% |
| ROAS YoY | ▲ 3,030% |
| 전환건수 YoY | ▲ 1,662% |

### AI 인사이트 (자동 생성)
1. 구매 ROAS 2,262%로 우수한 효율 → 예산 확대 권장
2. 전환매출 전년 대비 6,809% 증가
3. 비즈머니 잔액 경고 (7일 이내 소진)
4. 전환 0건 키워드 10개 정리 시 월 140,404원 절감
5. 최고 효율 키워드 추천
6. 최대 낭비 키워드 경고
7. 스토어 실제매출이 전환매출의 161% (자연검색 유입)
8. 광고비 SA 9% : GFA 91% 비율 분석

### 핵심 인사이트 상세
- **SA 효율 키워드 TOP 10**: ROAS 기준 순위 + 입찰가 + 변경 버튼
- **SA 비효율 키워드 TOP 10**: 낭비 금액 기준 순위 + 입찰가 + 변경 버튼
- **GFA 효율 캠페인 TOP 10**: ROAS 기준 순위
- **GFA 비효율 소재 TOP 10**: 성과 부진 소재

---

## 3. 판매 대시보드 (sales-dashboard-prototype.html)

### 개요
전체 브랜드 월간 매출 종합 대시보드

### 필터
- 연도: 2022~2026
- 월: 1~12
- ON/OFF 채널 체크박스

### 월 목표 달성 섹션
| 브랜드 | 목표 | 실적 | 달성률 |
|--------|------|------|--------|
| 키크론 | 11.3억 | 11.9억 | 104.5% |
| 지티기어 | - | - | 0.0% |
| AIPER | 2.4천만 | 3.2천만 | 131.4% |
| 캐논 | 3.0억 | 2.5억 | 81.8% |

### 매출 KPI 요약
| 지표 | 값 |
|------|-----|
| 총 매출 | 22.5억 |
| 일 평균 매출 | 8.0천만 |
| ON 채널 매출 | 17.6억 (78.3%) |
| OFF 채널 매출 | 4.9억 (21.7%) |
| 주문 건수 | 9,073건 (▼12.6%) |
| 수량 | 25,539개 |
| 건당 평균 단가 (AOV) | 25만 (▲5.9%) |
| 활성 거래처 | 52곳 |

### 이상 감지 & 액션
- 브랜드 매출 급감 알림 (그외: -58.8%, 지티기어 레이싱: -82.6%)
- 가격 정책, 재고 상황, 프로모션 점검 권장

### 차트
- **매출 트렌드 차트**: 일별/월별/년도별 (7일 이동평균 포함)
- **채널 분석 (ON/OFF)**: 도넛 차트 + 거래처별 매출 테이블
- **브랜드 포트폴리오 분석**: 브랜드별 매출/비중/수량/MoM
- **모델/상품 심층 분석**: 주요 모델별 매출 추이

### 채널별 매출 현황 (상위 10)
| # | 거래처 | 채널 | 매출 | 비중 |
|---|--------|------|------|------|
| 1 | 브랜드스토어 | ON | 7.0억 | 30.9% |
| 2 | Cafe24(신) 유튜브쇼핑 | ON | 4.6억 | 20.6% |
| 3 | 스마트스토어 | ON | 2.5억 | 11.1% |
| 4 | 쿠팡 로켓 (지티) | OFF | 2.5억 | 10.9% |
| 5 | (주)밸류포인트 | OFF | 1.5억 | 6.8% |

### 연결 DB 테이블
- `erp_annual_sales` → 연간 매출 데이터
- `erp_annual_model_sales` → 모델별 매출
- `erp_target_amount` → 매출 목표
- `erp_channel_list` + `erp_channel_price` → 채널 정보
- `tbnws_sabangnet_order` → 온라인 주문
- `erp_order_recv` → B2B 수주 (OFF 채널)
- `erp_partner` → 거래처 정보

---

## 4. 판매 분석 (sales-analysis-prototype.html)

### 탭 구조 (9개 분석 뷰)
| 탭 | 설명 |
|-----|------|
| 월별 목표 매출 | 브랜드별 목표 대비 실적 달성률 |
| 월별 전체 매출 | 월간 총매출 / ON / OFF / 건수 / 수량 / AOV |
| 월별 상세 매출 | 파트너별 상세 매출 테이블 |
| 일별 상세 매출 | 일자별 매출 상세 |
| 년별 비교 매출 | 전년도 대비 매출 비교 |
| 브랜드별 판매 추이 | 브랜드 시계열 추이 |
| 모델별 비교 매출 | 상품 모델간 매출 비교 |
| 악세서리 판매 추이 | 액세서리 카테고리 추이 |
| 앨리스배열 판매 추이 | 특정 키보드 배열 판매 추이 |

### 필터
- 연도 (2022~2026)
- 월 (전체/1~12)
- 브랜드 (전체/키크론/AIPER/캐논/파나텍/MOZA/플레이시트/트랙레이서/STRIX/ZOYO/지티기어 레이싱/트러스트마스터/로지텍/Gamesir)
- ON/OFF 채널 토글
- 테마 (Salesforce/Monochrome/Earth Tone/Midnight/Pastel)

### 연결 DB 테이블
- `erp_annual_sales` / `erp_annual_model_sales` / `erp_annual_acc_layout_sales`
- `erp_customer_month_targert_sales`
- `erp_channel_list` + `erp_partner`
- `tbnws_sabangnet_order`

---

## 5. 재고 분석 (inventory-analysis-prototype.html)

### 개요
상품 재고 흐름 파이프라인 및 위험 감지 대시보드

### 필터
- 카테고리 / 브랜드 / 상품 / 옵션
- 판매상태 (판매중)
- 기준일 (2026-02-28)

### 핵심 KPI
| 지표 | 값 | 설명 |
|------|-----|------|
| 품절 판매중 | 62 | 매출 손실 추정 ~2,325만/주 |
| 소진 임박 | 64 | 1주 내 재고 소진 예상 |
| 추가 발주 필요 | 3 | - |
| 입고 예정 | 302 SKU / 45,567개 | 이번주 입고 54,700개 |

### 재고 흐름 파이프라인
```
발주완료 707건 → 입고예정 302건 → 재고보유 1,481건 → 판매활성 669건 → 소진임박 64건
(116,706개)    (45,567개)      (42.5억)       (주간~3.0억)    (977개)
```

### 위험 감지 영역
| 영역 | 건수 | 내용 |
|------|------|------|
| 놓쳐진 발주 감지 | 103건 | 재고 4주 미만 + 발주 없음, 주간 위험액 ~3,494만 |
| 소진 속도 초과 | 72건 | 3달 평균 대비 주간판매 1.5배 이상, 초과소진 ~6,778만/주 |
| 출하 촉진 필요 | 12건 | PO 입고 전 재고 소진 예상, 위험액 ~884만/주 |
| 체류 재고 | 24.4억 | 사장 재고 6.6억 (90일간 판매 0건, 471 SKU), 저회전 17.7억 (12개월+ 소진, 310 SKU) |

### 연결 DB 테이블
- `erp_stock` + `erp_stock_summary` → 현재 재고
- `erp_order` + `erp_order_item` → 발주/입고 예정
- `erp_product_info` → 상품 마스터
- `erp_export_schedule` → 출고 스케줄
- `tbnws_sabangnet_order` → 판매 속도 계산

---

## 페이지간 데이터 연결 관계

```
┌──────────────────────────────────────────────────────────────────────┐
│                     tbe.kr (마케팅 분석)                               │
│  광고비 → 전환매출 → ROAS → 키워드/캠페인 최적화                          │
│  ↕ 실제 매출 비교                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                 매출 대시보드 (sales-dashboard)                         │
│  총매출 → ON/OFF 채널 → 브랜드 포트폴리오 → 이상감지                      │
│  ↕ 목표 대비 실적                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                 판매 분석 (sales-analysis)                             │
│  월별/일별/년별 → 브랜드별 → 모델별 → 채널별 심층 분석                     │
│  ↕ 판매량 데이터                                                       │
├──────────────────────────────────────────────────────────────────────┤
│                 재고 분석 (inventory-analysis)                         │
│  현재 재고 → 파이프라인 → 위험 감지 → 체류 재고                          │
│  ↕ 발주 필요 판단                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                 발주 예측 (orderForecast)                              │
│  가중평균 판매 → 안전재고 → 긴급/주의/안전 분류 → 자동 발주 추천            │
└──────────────────────────────────────────────────────────────────────┘
```

### 공통 데이터 소스
1. **판매 데이터**: `tbnws_sabangnet_order` + `erp_coupang_order` + `naver_commerce`
2. **상품 마스터**: `erp_product_info` (모든 페이지 공통)
3. **재고 데이터**: `erp_stock` + `erp_stock_summary`
4. **발주 데이터**: `erp_order` + `erp_order_item`
5. **채널/파트너**: `erp_partner` + `erp_channel_list`
6. **매출 목표**: `erp_target_amount` + `erp_customer_month_targert_sales`
