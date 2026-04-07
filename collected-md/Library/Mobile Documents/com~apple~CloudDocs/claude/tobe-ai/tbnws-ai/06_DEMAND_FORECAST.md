# 추세 가중 수요예측 (Trend-Weighted Demand Forecast)

> 신제품·성장기 상품은 단순 평균 판매량이 현재 속도를 과소평가함.
> 주간 판매 추이를 분석하여 추세 구간을 감지하고, 가중치를 적용한 예측 공식.
> 최종 업데이트: 2026-03-03

---

## 1. 문제 정의

### 단순 평균의 함정
```
EX75 사례 (2026년):
- ERP 월판매 334개 → 일평균 11.1개 → 재고소진 49일
- 실제 최근 2주: 일평균 14.0개 → 재고소진 39일 (20% 차이!)

원인: 출시 초기(일 4~5개) 저판매 구간이 평균을 끌어내림
```

### 적용 대상
- 신제품 출시 후 성장기 (launch → growth)
- 시즌 진입 상품 (수요 급증기)
- 프로모션/광고 효과로 판매 상승 중인 상품
- 반대로, 하락 추세 상품도 동일 로직으로 과대재고 방지 가능

---

## 2. 추세 감지 공식

### 2-1. 주간 판매량 수집
```sql
-- 최근 8주 주간 판매량 (일요일~토요일 기준)
SELECT
    DATE_FORMAT(order_date, '%Y-%U') AS week_label,
    MIN(order_date) AS week_start,
    SUM(quantity) AS weekly_qty,
    COUNT(DISTINCT DATE(order_date)) AS active_days,
    ROUND(SUM(quantity) / COUNT(DISTINCT DATE(order_date)), 1) AS daily_avg
FROM naver_smartstore_orders
WHERE account = %s
  AND product_name LIKE %s
  AND order_date >= DATE_SUB(CURDATE(), INTERVAL 8 WEEK)
  AND product_order_status NOT IN ('CANCELED', 'RETURNED', 'EXCHANGED')
GROUP BY DATE_FORMAT(order_date, '%Y-%U')
ORDER BY week_start ASC
```

### 2-2. 추세 판별 (3구간 비교)
```
구간 정의:
  - 초기(early):  W1 ~ W3 (첫 3주)
  - 중기(mid):    W4 ~ W6
  - 최근(recent): W7 ~ W8 (최근 2주)

추세 판별 기준:
  early_avg = 초기 3주 일평균
  recent_avg = 최근 2주 일평균
  growth_ratio = recent_avg / early_avg

  growth_ratio >= 2.0  → STRONG_GROWTH (급성장)
  growth_ratio >= 1.3  → GROWTH (성장)
  growth_ratio >= 0.8  → STABLE (안정)
  growth_ratio >= 0.5  → DECLINING (하락)
  growth_ratio <  0.5  → SHARP_DECLINE (급락)
```

### 2-3. EX75 실제 적용 예시
```
early_avg  = (4.0 + 5.4 + 5.1) / 3 = 4.83
recent_avg = (12.3 + 14.0) / 2 = 13.15
growth_ratio = 13.15 / 4.83 = 2.72 → STRONG_GROWTH

→ ERP 평균(11.1)이 아닌 recent_avg(13.15) 기준으로 재고 계산해야 함
```

---

## 3. 가중 수요예측 공식

### 3-1. 예측 일판매량 (forecast_daily)
```
추세별 가중치:
  STRONG_GROWTH:  forecast = recent_avg × 1.1  (추세 지속 + 10% 성장 반영)
  GROWTH:         forecast = recent_avg × 1.05
  STABLE:         forecast = overall_avg        (전체 평균 사용)
  DECLINING:      forecast = recent_avg × 0.95  (하락 반영)
  SHARP_DECLINE:  forecast = recent_avg × 0.85

EX75 적용:
  forecast = 13.15 × 1.1 = 14.5개/일
```

### 3-2. 재고 소진일 재계산
```
adjusted_days_of_supply =
    (current_stock + incoming_stock) / forecast_daily

EX75:
  (542 + 500) / 14.5 = 71.9일 (vs ERP기준 94일)
```

### 3-3. 발주 필요 판단
```
urgency_level:
  adjusted_days <= 14  → URGENT (즉시 발주)
  adjusted_days <= 30  → WARNING (발주 준비)
  adjusted_days <= 45  → WATCH (주시, 리드타임 고려)
  adjusted_days > 45   → OK

⚠️ 리드타임 보정:
  effective_days = adjusted_days - lead_time_days
  (키크론 해외 발주 리드타임: 약 30~45일)

EX75:
  effective_days = 71.9 - 35(리드타임) = 36.9일 → WARNING
  BUT: 1/23 발주건 37일째 미입고 → 입고 안되면 URGENT로 격상
```

---

## 4. AI 분석 규칙

### 재고 분석 시 필수 적용
1. **항상 주간 추이를 먼저 확인** — 단순 평균으로 바로 결론 내지 말 것
2. **growth_ratio 계산** — 1.3 이상이면 recent_avg 기준으로 전환
3. **리드타임 감안** — 해외 발주(키크론) 30~45일, 국내 5~10일
4. **발주 상태 확인** — DRAFT(임시)/ORDERED(확정·미입고)/PARTIAL(일부입고) 모두 "진행 중" 취급
5. **SKU별 분리 분석** — 동일 모델이라도 기존 판매 SKU와 신규 SKU는 별도 분석 (Section 7 참고)

### 리포트 출력 양식
```
📦 [상품명] 재고 분석
├─ 현재고: XXX개
├─ ERP 일평균: XX.X개 (⚠️ 추세 미반영)
├─ 주간 추이: [W1] → [W2] → ... → [최근]
├─ 추세: STRONG_GROWTH (growth_ratio: X.XX)
├─ 예측 일판매: XX.X개 (가중 적용)
├─ 예측 소진일: XX일 (입고예정 포함: XX일)
├─ 리드타임 감안: XX일 (해외 35일)
└─ 판단: ⚠️ WARNING — 3월 중순까지 추가 발주 결정 필요
```

### 주의사항
- 출시 4주 미만 상품은 데이터 부족 → growth_ratio 대신 "모니터링" 표시
- 프로모션/이벤트 기간 판매량은 별도 표시 (일시적 스파이크)
- 하락 추세에서도 동일 공식 적용 → 과대재고 방지
- 여러 옵션이 합산된 경우 옵션별 분리 분석 권장

---

## 5. 데이터소스 매핑

| 데이터 | 테이블 | 비고 |
|--------|--------|------|
| 주간 판매량 | `naver_smartstore_orders` | account + product_name LIKE |
| 현재 재고 | `erp_stock_realtime` | GT(G-code) + CJ(F-code) 합산 |
| 입고 예정 | `erp_stock_realtime.beDueEa` | GT + CJ 합산 |
| 발주 현황 | `erp_purchase_order` | 4단계: DRAFT/ORDERED/PARTIAL/RECEIVED (LEFT JOIN erp_order_item+erp_stock으로 입고 상태 판별) |
| 수입원가 | `erp_stock` + `erp_order_item` | cur2krw > 100 필터 |
| 리드타임 | 수동 설정 | 해외 35일, 국내 7일 (추후 DB화) |

---

## 6. API 엔드포인트

```
GET /api/v1/inventory/trend-analysis?product_code=G-0029-0509-0001
GET /api/v1/inventory/trend-analysis?keyword=EX75

응답:
{
  "product_code": "G-0029-0509-0001",
  "current_stock": 542,
  "weekly_trend": [
    {"week": "W2", "daily_avg": 4.0},
    {"week": "W3", "daily_avg": 5.4},
    ...
  ],
  "trend": "STRONG_GROWTH",
  "growth_ratio": 2.72,
  "erp_daily_avg": 11.1,
  "forecast_daily": 14.5,
  "erp_days_of_supply": 49,
  "adjusted_days_of_supply": 37,
  "pending_orders": [...],
  "urgency": "WARNING",
  "recommendation": "3월 중순까지 추가 발주 결정 필요"
}
```

---

## 7. 발주 상태 4단계 시스템

### 7-1. PO 상태 판별 (erp_purchase_order)
```
기존 문제: order_seq IS NULL = "임시" 로만 판별
→ order_seq IS NOT NULL인데 실제 입고 안 된 건을 놓침 (EX75 2000개 누락 사건)

수정된 4단계:
  DRAFT    — order_seq IS NULL (임시 저장, 발주 미확정)
  ORDERED  — order_seq IS NOT NULL + 입고수량 0 (확정됐으나 미입고)
  PARTIAL  — order_seq IS NOT NULL + 0 < 입고수량 < 발주수량 (일부 입고)
  RECEIVED — order_seq IS NOT NULL + 입고수량 >= 발주수량 (전량 입고 완료)

입고 확인 방법:
  erp_purchase_order po
  LEFT JOIN erp_order_item oi ON po.order_seq = oi.order_seq AND po.product_code = oi.product_code
  LEFT JOIN erp_stock s ON oi.item_seq = s.item_seq AND s.import_date IS NOT NULL
  → imported_qty = COUNT(s.stock_seq)
```

### 7-2. "발주 진행 중" 기준
```
발주 진행 중 = DRAFT + ORDERED + PARTIAL
→ RECEIVED만 제외하고 모두 "아직 물건 안 들어온 발주"로 취급
→ pending_order_qty에 합산
→ 재고 소진일 계산 시 가용재고에 포함
```

---

## 8. SKU별 발주 분석 규칙

### 8-1. 문제 정의
```
EX75 발주 2000개 ≠ 현재 판매 SKU 2000개

실제 구성:
  EX75-A6 (기존 판매 SKU) — 500개
  EX75-A1 (신규 SKU)      — 500개
  EX75-J6 (신규 SKU)      — 500개
  EX75-J1 (신규 SKU)      — 500개

→ 기존 판매 SKU(A6)는 500개만 발주된 상태!
→ 전체 2000개를 퉁쳐서 "여유 있음" 판단하면 위험
```

### 8-2. 기존 판매 SKU vs 신규 SKU 분리
```
기존 판매 SKU:
  - 이미 판매 이력이 있고 추세 데이터 존재
  - growth_ratio 적용하여 미래 수요 예측 가능
  - 성장 추세라면 판매가 더 올라갈 가능성 반영해야 함
  - 해당 SKU의 PO 수량만으로 소진일 계산

신규 SKU (미출시/출시 직후):
  - 판매 이력 없거나 극소량
  - 추세 데이터 불충분 → growth_ratio 대신 "모니터링"
  - 램프업(ramp-up) 기간 필요 — 출시 후 4~8주 소요
  - 해당 SKU PO는 별도 재고로 취급 (기존 SKU 소진일에 합산 금지)
```

### 8-3. 분석 워크플로우
```
1. get_trend_analysis(keyword="EX75")
   → 전체 키워드 추세 + 전체 PO 목록 확인

2. get_inventory_current(keyword="EX75")
   → SKU별 현재고, 월판매량(monthly_sales) 확인
   → 기존 판매 SKU 식별 (monthly_sales > 0)

3. SKU별 분리 계산:
   기존 SKU (예: A6):
     - 해당 SKU 현재고 + 해당 SKU PO수량
     - 해당 SKU 일판매 = SKU별 monthly_sales / 30
     - 성장 추세 반영: forecast = daily × growth_weight
     - 소진일 = (현재고 + PO) / forecast

   신규 SKU (예: A1, J6, J1):
     - 해당 SKU 현재고 + 해당 SKU PO수량
     - 판매 예측 불가 → "입고 후 모니터링" 권고
     - 기존 SKU 소진일 계산에서 제외

4. 최종 판단:
   - 기존 SKU 소진일 기준으로 추가 발주 필요성 판단
   - 신규 SKU는 별도 안내 (램프업 기간 고려)
```

### 8-4. EX75 실제 적용 예시
```
📦 EX75 SKU별 발주 분석

[기존 판매 SKU]
├─ EX75-A6: 현재고 542 + PO 500 = 1,042개
├─ 추세: STRONG_GROWTH (ratio 2.72)
├─ 예측 일판매: 14.5개 → 더 성장 가능성 있음
├─ 소진일: 1,042 / 14.5 = 71.9일
├─ 리드타임 감안: 71.9 - 35 = 36.9일
└─ 판단: ⚠️ WARNING — A6 추가 발주 검토 필요

[신규 SKU]
├─ EX75-A1: PO 500개 (미출시, 판매이력 없음)
├─ EX75-J6: PO 500개 (미출시)
├─ EX75-J1: PO 500개 (미출시)
└─ 판단: 📊 MONITOR — 출시 후 4주간 판매추이 관찰 필요
```
