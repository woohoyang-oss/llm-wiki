# Session 2026-02-22~23: GFA 퍼널 분석 고도화 + 모바일 반응형 수정

> 작업일: 2026-02-22(토) ~ 2026-02-23(일)
> 커밋: c0ed9a1 → f35a3b8 → 6161ddb → 6850664 → 4b7f9c5 → 7b05db8

---

## 작업 요약

이번 세션에서는 크게 **4가지 주요 영역**을 작업했습니다:

1. **GFA 대시보드 기능 보강** — DOW 히트맵, 시즈널 인사이트, 전환 데이터 보강, ROAS 개선 시뮬레이션
2. **GFA 퍼널 분석 엔진 전면 교정** — BENCHMARKS GFA 기준 교정, 분류 로직 개선, 어트리뷰션 보정, 레이아웃 3-section 개편
3. **모바일 반응형 버그 수정** — 사이드바 햄버거 버튼, KPI 바 레이아웃, 스테이지 메트릭 그리드, split-layout 오버플로우
4. **To-Be 퍼널 시나리오 플래너 설계** — 플랜 작성 완료 (구현 대기 중)

---

## 1. GFA 대시보드 기능 보강

### 커밋: c0ed9a1, f35a3b8

### 1-1. DOW 히트맵 (요일별 성과 히트맵)

**파일**: `app/routes/gfa.py`, `app/static/js/tabs/gfa.js`, `app/static/styles/main.css`

- SA/GFA 공통 DOW 히트맵 구현
- Diverging color scale 적용 (빨강 ← 흰 → 초록)
- 히트맵 테이블: `table-layout:fixed` + `min-width`로 빈 요일 컬럼 너비 유지
- 선택 기간 7일 미만 시 자동으로 최소 7일 확장 (API 단)

### 1-2. GFA 전환 데이터 보강

**파일**: `app/routes/gfa.py`

- `adgroup_daily` 테이블에 전환 데이터 없는 경우 `campaign_daily` fallback 추가
- GFA overview summary 쿼리 보강

### 1-3. ROAS 개선 시뮬레이션 카드

**파일**: `app/static/js/tabs/gfa.js`

- 기존 "절감 가능액" 카드 → "비효율 제거 시 ROAS 개선 시뮬레이션" 카드로 교체
- 구매ROAS before→after 비교 표시

### 1-4. SA 진단 요일 패턴 분석

**파일**: `app/routes/diagnosis.py` (349줄+)

- Welch's t-test 기반 통계 검증
- 요일별 성과 편차 분석 및 액션 아이템 자동 생성

### 1-5. 비효율 드릴다운 (GFA)

**파일**: `app/routes/gfa.py` (130줄+), `app/routes/overview.py` (127줄+)

- 캠페인 → 광고그룹 → 소재 계층 드릴다운
- SA overview 비효율 드릴다운 추가

### 1-6. 배치 히스토리 탭

**파일**: `app/static/js/tabs/batch-history.js` (신규, 175줄), `app/static/index.html`, `app/static/js/app.js`

- 크론 배치 수집 이력 조회 UI

---

## 2. GFA 퍼널 분석 엔진 전면 교정

### 커밋: 6161ddb, 6850664

### 2-1. BENCHMARKS 교정 (SA → GFA)

**파일**: `app/funnel_engine.py` (라인 28~53)

기존 SA 기준 → GFA 디스플레이 기준으로 전면 교정:

| 지표 | TOF | MOF | BOF |
|------|-----|-----|-----|
| CTR% | 0.05~0.5 | 0.2~1.2 | 0.5~8.0 |
| CPM원 | 500~4,000 | 2,000~10,000 | 5,000~25,000 |
| CVR% | 0~0.2 | 0.1~0.8 | 0.5~15.0 |
| Freq | 1.0~3.0 | 3.0~5.0 | 4.0~8.0 |
| ROAS%목표 | 50 | 100 | 300 |

### 2-2. FunnelClassifier 분류 로직 개선

**파일**: `app/funnel_engine.py` (FunnelClassifier 클래스, 라인 98~289)

- 분류 가중치 재조정: **강한 이름 매칭** 시 name 40% / conv 25% / metrics 20% / reach 15%
- 이름 **미매칭** 시 name 0%로 재분배 (conv 35% / metrics 35% / reach 30%)
- `TOF_PATTERNS`에 '1단계', '모수', '신규' 키워드 추가
- CPM 기반 `reach_freq` 신호 보강

### 2-3. GFA 어트리뷰션 보정 (View-Through)

**파일**: `app/funnel_engine.py` (라인 68~74)

GFA는 view-through 전환이 포함되므로 ROAS를 단계별로 할인:

```python
GFA_ATTRIBUTION_DISCOUNT = {
    'TOF': 0.3,   # 30%만 인정 (대부분 view-through)
    'MOF': 0.5,   # 50% 인정
    'BOF': 0.8,   # 80% 인정 (대부분 direct click)
}
```

- `adjusted_roas = raw_roas * discount_factor`
- 스테이지 카드, 분류 테이블, KPI 바에 보정ROAS 표시 추가

### 2-4. 베이지안/Hill/Adstock 조정

- **STAGE_PRIORS**: 베이지안 사전분포 GFA 기준 조정
- **HillSaturation**: `R_max` 보수적 추정 + 신뢰도 경고 추가
- **ActionItem**: `scale_opportunity` 상위 3개 제한 + 임계값 상향

### 2-5. 액션플랜 교체: '분류 불일치' → '뷰스루 기여도 분석'

**파일**: `app/funnel_engine.py` (ActionItemGenerator 클래스)

- 기존 "분류 불일치" 아이템 제거
- 새로운 "뷰스루 어트리뷰션 기여도 분석" 아이템 추가
- GFA 디스플레이 특성에 맞는 VT 기여도 해석 가이드

### 2-6. 퍼널 인사이트: 한계ROAS 기반 예산 재배분

**파일**: `app/funnel_engine.py` (`_generate_funnel_insights()`, 라인 1551~1795)

- 기존 "이상적 비율" 기반 인사이트 → **한계ROAS(Marginal ROAS) 기반 재배분 인사이트** 로 교체
- 등한계 원칙(Equimarginal Principle) 적용
- Hill 포화곡선에서의 한계수익 비교 로직

### 2-7. 레이아웃 3-Section 개편

**파일**: `app/static/js/tabs/funnel.js` (179줄 변경), `app/static/js/components/funnel-chart.js`

```
[Section 1] 컴팩트 KPI 바
  등급뱃지 + 주요 지표 1줄 (건강도 / 총비용 / 총전환 / 보정ROAS / 캠페인수)

[Section 2] 퍼널 + 단계카드
  좌(45%): SVG 깔때기 다이어그램 (360×280 축소)
  우(55%): 단계별 카드 (TOF/MOF/BOF 메트릭)

[Section 3] 서브탭 5개
  액션플랜 | 전환퍼널 | 포화곡선 | 광고잔존효과 | 분류상세
```

---

## 3. 모바일 반응형 버그 수정

### 커밋: 6850664, 4b7f9c5, 7b05db8

### 3-1. 모바일 사이드바 햄버거 버튼 안 보이는 문제

**파일**: `app/static/styles/main.css`

**원인**: CSS 순서 문제

```
라인 319 (media query): .mobile-menu-btn { display: block; }
라인 557 (base rule):   .mobile-menu-btn { display: none; }
→ 나중에 선언된 base rule이 항상 우선!
```

**해결**: `display: block !important` 적용

```css
/* @media (max-width: 768px) */
.mobile-menu-btn { display: block !important; }
```

### 3-2. 모바일 스크롤 불가 (하단 잘림)

**파일**: `app/static/styles/main.css`

**원인**: `.app-layout`에 `height:calc(100vh - 52px); overflow:hidden` 설정

**해결**:
```css
/* @media (max-width: 768px) */
.app-layout {
    height: auto;
    min-height: 100vh;
    overflow: visible;
    flex-direction: column;
}
.main-content {
    min-height: 0;
    overflow-y: auto;
    -webkit-overflow-scrolling: touch;
}
```

### 3-3. KPI 바 레이아웃 정리

**파일**: `app/static/styles/main.css`, `app/static/js/tabs/funnel.js`

**변경**: `flex-wrap` → CSS Grid

```css
.fkb-items {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(110px, 1fr));
    flex: 1;
    padding: 0;
    gap: 0;
}
```

- `<div class="fkb-divider">` 요소 모두 제거 (JS 템플릿에서 삭제)
- 반응형: `@media 900px` → 3열, `@media 480px` → 2열

### 3-4. 스테이지 메트릭 그리드 모바일 깨짐

**파일**: `app/static/styles/main.css`

**원인**: CSS 특이성 문제

```
.fa-split-right .fa-stage-metrics (특이성 0,2,0 — 라인 795/801)
@media 768px .fa-stage-metrics   (특이성 0,1,0 — 라인 1011)
→ 0,2,0이 항상 우선!
```

**해결**: 미디어쿼리 셀렉터도 `.fa-split-right .fa-stage-metrics` 포함

```css
@media (max-width: 768px) {
    .fa-stage-metrics,
    .fa-split-right .fa-stage-metrics {
        grid-template-columns: repeat(3, 1fr) !important;
        gap: 6px;
    }
}
```

### 3-5. 퍼널 split-layout 오버플로우 (16px 초과)

**파일**: `app/static/styles/main.css`

**원인**: `flex: 0 0 45%` + `flex: 0 0 55%` + `gap: 16px` = 100% + 16px

```javascript
// 검증: scrollWidth(1375) > clientWidth(1359) → 16px 초과
```

**해결**: 비율 기반 flex로 변경 (shrink 허용)

```css
.fa-split-left  { flex: 45 1 0; min-width: 0; }
.fa-split-right { flex: 55 1 0; min-width: 0; }
```

**검증**: `scrollWidth === clientWidth` 확인 완료

### 3-6. 보정ROAS 서브텍스트 축약

**파일**: `app/static/js/components/funnel-chart.js`

- `'view-through 보정'` → `'VT보정'` (모바일 공간 절약)

---

## 4. URL 라우터 날짜 파라미터 동기화

### 커밋: 6850664

**파일**: `app/static/js/router.js`, `app/static/js/app.js`

- URL 쿼리 파라미터(`start`, `end`)가 탭 전환 시 유지되도록 동기화 개선
- `router.js`에 날짜 파라미터 파싱/적용 로직 추가

---

## 5. To-Be 퍼널 시나리오 플래너 (설계 완료, 구현 대기)

### 설계 문서: `~/.claude/plans/wise-foraging-harp.md`

사용자 요구사항: **현재 퍼널(As-Is)은 유지하면서, 향후 퍼널을 어떻게 발전시킬지(To-Be) 가이드 제시**

### 핵심 기능

1. **승격 후보 식별**: TOF/MOF 중 효율이 너무 좋은 캠페인 → 상위 단계 승격 추천
   - TOF→MOF: adj_ROAS ≥ 200%, CVR ≥ 0.3%, 전환 ≥ 3건
   - TOF→BOF: adj_ROAS ≥ 500%, CVR ≥ 1.0%, 전환 ≥ 10건
   - MOF→BOF: adj_ROAS ≥ 400%, CVR ≥ 0.8%, 전환 ≥ 5건

2. **예산 갭 분석**: To-Be 총예산 = 현재 × 1.2, IDEAL_BUDGET_RATIOS 대비 갭 산출

3. **신규 캠페인 생성 가이드**: 캠페인명, 목표, 타겟(모수/연령/성별), 예산(일/월/비중), 예상지표(CPM/CPC/CTR/전환), 소재가이드(포맷/메시지/CTA/사이즈)

4. **As-Is vs To-Be 시각적 비교**: 좌우 깔때기 다이어그램 + 건강도 변화

5. **실행 타임라인**: 주차별 단계 체크리스트

### 수정 파일 계획 (5개)

| 파일 | 변경 내용 | 예상 규모 |
|------|----------|----------|
| `app/funnel_engine.py` | `ToBeScenarioPlanner` 클래스 | ~350줄 |
| `app/routes/funnel.py` | `/api/funnel/tobe` 엔드포인트 | ~40줄 |
| `app/static/js/tabs/funnel.js` | "To-Be 시나리오" 서브탭 + lazy loading | ~120줄 |
| `app/static/js/components/funnel-chart.js` | `ToBeComparisonChart` 클래스 | ~350줄 |
| `app/static/styles/main.css` | `.fa-tobe-*` CSS 클래스 | ~100줄 |

---

## 수정 파일 요약

| 파일 | 변경 범위 | 주요 변경 |
|------|----------|----------|
| `app/funnel_engine.py` | 510줄+ / 224줄+ | BENCHMARKS GFA 교정, 분류 로직 개선, VT 보정, 한계ROAS 인사이트 |
| `app/routes/funnel.py` | (변경 없음) | — |
| `app/routes/gfa.py` | 130줄+ / 8줄 | DOW 히트맵, 전환 fallback, 7일 최소 확장 |
| `app/routes/diagnosis.py` | 349줄+ | SA 요일 패턴 Welch's t-test |
| `app/routes/overview.py` | 127줄+ | SA 비효율 드릴다운 |
| `app/static/js/tabs/funnel.js` | 179줄 / 71줄 / 8줄 | 레이아웃 3-section, VT 보정 표시, KPI divider 제거 |
| `app/static/js/components/funnel-chart.js` | 9줄 / 17줄 / 2줄 | SVG 축소, VT보정 서브텍스트 |
| `app/static/js/tabs/gfa.js` | 187줄+ / 21줄 | DOW 히트맵, ROAS 시뮬레이션 |
| `app/static/js/tabs/overview.js` | 288줄+ | SA DOW 히트맵, 비효율 드릴다운 |
| `app/static/js/tabs/batch-history.js` | 175줄 (신규) | 배치 이력 UI |
| `app/static/index.html` | 43줄+ | 배치 이력 탭, 사이드바 항목 |
| `app/static/js/app.js` | 5줄 / 2줄 | 탭 등록, 라우터 연동 |
| `app/static/js/router.js` | 9줄 | 날짜 파라미터 동기화 |
| `app/static/styles/main.css` | 69줄 / 60줄 / 45줄 / 17줄 | 반응형 수정 다수 |

---

## CSS 교훈 (Lessons Learned)

### 1. CSS 순서 > 미디어쿼리
미디어쿼리 안의 규칙이라도, 나중에 선언된 동일 특이성 규칙에 밀린다.
**해결**: `!important` 또는 미디어쿼리를 파일 끝으로 이동.

### 2. CSS 특이성과 미디어쿼리
`.parent .child` (0,2,0) > `@media .child` (0,1,0)
미디어쿼리는 특이성을 높이지 않는다.
**해결**: 미디어쿼리 내에서도 동일한 셀렉터 복잡도 사용.

### 3. Flex 퍼센트 + Gap = 오버플로우
`flex: 0 0 45%` + `flex: 0 0 55%` + `gap: 16px` = 100% + 16px (오버플로우)
**해결**: `flex: 45 1 0` / `flex: 55 1 0` (비율 기반 + shrink 허용)

---

## 현재 퍼널 엔진 아키텍처 (funnel_engine.py)

```
funnel_engine.py (~1850줄)
│
├── BENCHMARKS          (GFA 디스플레이 기준)
├── IDEAL_BUDGET_RATIOS (TOF 15~35%, MOF 15~25%, BOF 40~60%)
├── GFA_ATTRIBUTION_DISCOUNT (TOF 0.3, MOF 0.5, BOF 0.8)
│
├── 1. FunnelClassifier      — 4-signal 확률적 분류 (이름/전환/메트릭/도달)
├── 2. BetaBinomialEstimator — 베이지안 CTR/CVR 추정
├── 3. HillSaturation        — Hill 포화곡선 (예산 vs 전환 체감)
├── 4. AdstockDecay          — 광고 이월효과 감쇠
├── 5. BudgetOptimizer       — 등한계 원칙 예산 재배분
├── 6. FunnelHealthScorer    — 퍼널 건강도 (100점)
├── 7. ActionItemGenerator   — 액션 아이템 자동 생성
├── 8. CampaignMatrixScorer  — 캠페인 종합 점수
│
├── run_funnel_analysis()    — 통합 분석 함수
├── _generate_funnel_insights() — 한계ROAS 기반 인사이트
└── 유틸리티 함수들
```

---

## 배포 기록

| 시각 | 대상 | 방식 |
|------|------|------|
| 02-22 18:10 | c0ed9a1 전체 | SCP + Flask 재시작 |
| 02-23 00:06 | f35a3b8 전체 | SCP + Flask 재시작 |
| 02-23 10:34 | 6161ddb 전체 | SCP + Flask 재시작 |
| 02-23 13:53 | 6850664 전체 | SCP + Flask 재시작 |
| 02-23 13:58 | 4b7f9c5 CSS+JS | SCP (재시작 불필요) |
| 02-23 14:37 | 7b05db8 CSS+JS | SCP (재시작 불필요) |

---

## 다음 작업

- [ ] `ToBeScenarioPlanner` 클래스 구현 (funnel_engine.py)
- [ ] `/api/funnel/tobe` API 엔드포인트 (routes/funnel.py)
- [ ] "To-Be 시나리오" 서브탭 프론트엔드 (funnel.js)
- [ ] `ToBeComparisonChart` 시각화 컴포넌트 (funnel-chart.js)
- [ ] `.fa-tobe-*` CSS 스타일 (main.css)
- [ ] EC2 배포 + 테스트
