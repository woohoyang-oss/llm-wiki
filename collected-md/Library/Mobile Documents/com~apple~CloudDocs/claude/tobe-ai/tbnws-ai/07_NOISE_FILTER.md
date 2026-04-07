# 07. 발주 노이즈 필터링 (Smart Reorder)

## 1. 배경 및 목적

### 문제
기존 `reorder-alerts` 엔드포인트는 ERP 재고 데이터 기반으로 단순히 days_of_supply가 낮은 항목을 경고했다.
그러나 실제 분석 결과 **92개 "긴급" 발주 항목 중 11개만 실제 발주 필요**한 것으로 판별되었다.

나머지 81개는 다음과 같은 "노이즈"였다:
- 이미 후속 모델로 전환된 구형 제품 (B1→B1 Pro, M6→M6S)
- 판매 이력은 있으나 사실상 단종된 제품
- 스마트스토어가 아닌 사방넷 채널 전용 제품
- 시즌 종료된 캐릭터 상품 (로이체)
- 월 1~3개 소량 판매 품목

### 해결
`/api/v1/inventory/smart-reorder` 엔드포인트를 추가하여 5가지 노이즈를 자동 감지하고,
각 항목에 `recommended_action` (REORDER/SKIP/VERIFY/MONITOR)을 부여한다.

---

## 2. 노이즈 유형 (noise_type)

### 2.1 MODEL_TRANSITION (모델 전환)

**감지 조건**: Keychron 브랜드 상품에서 후속 모델이 활성 판매 중인 경우

**후속 모델 패턴**:
| 구형 | 신형 | 패턴 |
|------|------|------|
| 키크론 B1 | 키크론 B1 Pro | goods_name + " Pro" |
| 키크론 B6 | 키크론 B6 Pro | goods_name + " Pro" |
| M6 (M6-A28 등) | M6S (M6S-A1 등) | model_code + "S" |
| M3 (M3-xxx) | M3S (M3S-xxx) | model_code + "S" |
| K8 Pro | K8 Pro SE | goods_name + " SE" |

**모델 코드 추출 규칙**:
```
option_name: "(B1-K5) Lavender..." → model_code = "B1"
option_name: "(M6S-A1) M6 Black..." → model_code = "M6S"
```
regex: `\(([A-Za-z0-9]+)` — 괄호 안 첫 번째 alphanumeric 그룹

**판정 로직**:
1. goods_name에 "Pro"/"SE" 없음 → 같은 이름 + " Pro" 활성 상품 검색
2. model_code가 "S"/"P"로 끝나지 않음 → model_code + "S" 활성 상품 검색
3. goods_name에 "Pro" 있고 "SE" 없음 → "Pro SE" 활성 상품 검색

**confidence**: HIGH (Rule 1, 2) / MEDIUM (Rule 3)
**recommended_action**: SKIP (발주 불필요)

### 2.2 DISCONTINUED (단종 의심)

**감지 조건**: 재고 0 + PO 미진행(DRAFT/ORDERED/PARTIAL 모두 없음) + 판매 감소 패턴

| 패턴 | confidence | action |
|------|------------|--------|
| 주간=0, 월간>0, 분기>월간×2 (급감) | HIGH | SKIP |
| 주간=0, 월간>0 (최근 판매 없음) | MEDIUM | VERIFY |
| 월간=0, 분기>0 (1개월 이상 판매 없음) | HIGH | SKIP |

**데이터 소스**: `erp_stock_realtime`의 `oneWeekSalesEa`, `oneMonthSalesEa`, `threeMonthSalesEa`

### 2.3 CHANNEL_MISMATCH (채널 불일치)

**감지 조건**:
1. **프린터 등 사방넷 전용**: product_code가 `G-9995*` 또는 brand_name = "캐논"
2. **비-키크론 브랜드**: Keychron/로이체/캐논 외 브랜드 (월판매 10개 이상)

이들은 네이버 스마트스토어 대신 쿠팡, 지마켓 등 사방넷 채널에서 주로 판매되어
`trend-analysis` (naver_smartstore_orders 기반) 검증이 불가능.

**confidence**: MEDIUM (프린터/캐논) / LOW (기타 브랜드)
**recommended_action**: VERIFY (월판매 30+ & 14일 이내 소진이면 REORDER)

### 2.4 SEASONAL (시즌/캐릭터 상품)

**감지 조건**: brand_name ∈ {"로이체"} (산리오, 잔망루피, 미피, 스누피 등 캐릭터 라인)

로이체 브랜드는 캐릭터 라이선스 제품으로 시즌별 라인업 교체가 빈번하다.
PO 유무(DRAFT/ORDERED/PARTIAL)에 따라 시즌 종료/신규 대기 판별.

| PO 유무 | confidence | action |
|---------|------------|--------|
| PO 없음 (미입고 발주 0건) | HIGH | VERIFY (시즌 종료 확인) |
| PO 진행 중 (DRAFT/ORDERED/PARTIAL) | LOW | MONITOR |

### 2.5 LOW_VOLUME (소량 품목)

**감지 조건**: 월판매 ≤ 3 AND 분기판매 ≤ 10

소량 판매 품목은 일시 품절이 비즈니스 임팩트가 낮고,
ERP 재고 소진 계산이 부정확해지는 구간이다.

**confidence**: HIGH
**recommended_action**: SKIP

---

## 3. 분류 우선순위

노이즈 판정은 아래 순서로 적용되며, 먼저 매칭된 유형이 최종 판정:

```
1. LOW_VOLUME     (월 ≤3, 분기 ≤10)
2. SEASONAL       (로이체 브랜드)
3. CHANNEL_MISMATCH (G-9995*, 캐논, 비-키크론 고판매)
4. MODEL_TRANSITION (Keychron 후속 모델 존재)
5. DISCONTINUED   (재고 0 + PO 없음 + 판매 감소)
6. 비-키크론 브랜드 CHANNEL_MISMATCH (월판매 10+)
7. NONE           (정상 → REORDER 또는 MONITOR)
```

### NONE (노이즈 없음)인 경우:
- PO 미진행 (DRAFT/ORDERED/PARTIAL 없음) → `recommended_action = REORDER`
- PO 진행 중 (DRAFT/ORDERED/PARTIAL 중 하나 이상) → `recommended_action = MONITOR`

### 발주 상태 4단계 (06_DEMAND_FORECAST.md §7 참조):
```
DRAFT    — order_seq IS NULL (임시 저장)
ORDERED  — 확정 + 미입고
PARTIAL  — 일부 입고
RECEIVED — 전량 입고 완료 (진행 중에서 제외)
```

---

## 4. API 엔드포인트

### `GET /api/v1/inventory/smart-reorder`

**파라미터**:
| 이름 | 타입 | 설명 |
|------|------|------|
| action | string (enum) | 필터: REORDER / SKIP / VERIFY / MONITOR (미지정=전체) |
| min_monthly | integer | 최소 월판매량 필터 (기본 0) |

**응답 필드** (SmartReorderAlert):
| 필드 | 설명 |
|------|------|
| product_code | G-code |
| brand_name, goods_name, option_name | 상품 정보 |
| total_stock | GT+CJ 합산 재고 |
| total_incoming | 입고 예정 합산 |
| safe_ea | 안전재고 기준 |
| weekly_sales | 주간 판매량 |
| monthly_sales | 월 판매량 |
| quarterly_sales | 3개월 판매량 |
| daily_avg | 일평균 판매량 |
| days_of_supply | 재고 소진 예상일 |
| alert_level | STOCKOUT/BELOW_SAFETY/LOW_2WEEKS/LOW_1MONTH |
| **noise_type** | NONE/MODEL_TRANSITION/DISCONTINUED/CHANNEL_MISMATCH/SEASONAL/LOW_VOLUME |
| **noise_confidence** | HIGH/MEDIUM/LOW/NONE |
| **noise_reason** | 노이즈 판정 사유 (한글) |
| **successor_model** | 후속 모델 option_name (MODEL_TRANSITION인 경우) |
| **has_pending_po** | 미입고 발주 존재 여부 (DRAFT+ORDERED+PARTIAL 기준) |
| **pending_po_qty** | 미입고 발주 수량 합계 (DRAFT+ORDERED+PARTIAL) |
| **recommended_action** | REORDER/SKIP/VERIFY/MONITOR |

### 사용 예시

```bash
# 실제 발주 필요 품목만 조회
curl "http://localhost:8100/api/v1/inventory/smart-reorder?action=REORDER"

# 확인 필요 품목 (담당자 검증 필요)
curl "http://localhost:8100/api/v1/inventory/smart-reorder?action=VERIFY"

# 월판매 5개 이상만
curl "http://localhost:8100/api/v1/inventory/smart-reorder?min_monthly=5"

# 전체 (노이즈 분류 포함)
curl "http://localhost:8100/api/v1/inventory/smart-reorder"
```

---

## 5. 검증 결과 (2026-02 기준 실데이터)

### 기존 reorder-alerts vs smart-reorder 비교

| 구분 | reorder-alerts | smart-reorder (REORDER) |
|------|---------------|------------------------|
| STOCKOUT 경고 | 92건 | **~11건** |
| 불필요 발주 방지 | 0건 | **~81건** |
| 정확도 | ~12% | **~88%** |

### 노이즈 분류 분포

| noise_type | 건수 | 예시 |
|-----------|------|------|
| MODEL_TRANSITION | ~20 | B1→B1 Pro, B6→B6 Pro, M6→M6S, M3→M3S |
| DISCONTINUED | ~15 | Q1 HE, K1 MAX, JK08, GKG108R |
| CHANNEL_MISMATCH | ~8 | MG3090, TS3490 (캐논), G75 Pro (Mchose) |
| SEASONAL | ~36 | 로이체 (산리오, 잔망루피, 미피, 스누피) |
| LOW_VOLUME | ~12 | 월 1~3개 판매 액세서리 |
| NONE (실제 발주) | ~11 | Keychron 주력 상품 |

### 실제 발주 필요 품목 특성
- Keychron 주력 키보드 (Q 시리즈, K 시리즈, V 시리즈)
- 일평균 판매 0.5개 이상
- PO 미진행 상태
- 후속 모델 없음

---

## 6. MCP Tool 연동

```json
{
  "name": "get_smart_reorder",
  "description": "★ 스마트 발주 알림 — 노이즈 필터링 적용. 발주 판단 시 reorder_alerts 대신 반드시 이것을 사용!",
  "params": {
    "action": "REORDER/SKIP/VERIFY/MONITOR (선택)",
    "min_monthly": "최소 월판매량 (기본 0)"
  }
}
```

AI 어시스턴트는 발주 관련 질문 시:
1. `get_smart_reorder` (action=REORDER) → 실제 발주 필요 품목 목록
2. 각 품목에 대해 `get_trend_analysis` → 추세 가중 수요예측
3. 추세+재고+리드타임 종합 → 발주 수량/시기 권고
