# GFA 퍼널 분석 엔진 — 이론적 기반 레퍼런스

## 적용된 학술 이론 & 실무 프레임워크

### 1. 퍼널 구조 모델

**STDC Framework (See-Think-Do-Care)**
- Google의 Avinash Kaushik이 제안한 4단계 프레임워크
- 기존 AIDA(Attention-Interest-Desire-Action) 대비 디지털 환경에 맞게 재구성
- 코드 적용: TOF(See) / MOF(Think) / BOF(Do) 3단계로 단순화

**Google Messy Middle (2020)**
- 소비자 의사결정은 선형 퍼널이 아닌 "탐색(Explore) ↔ 평가(Evaluate)" 반복
- 코드 적용: 확률적 분류(hard classification 대신 probability-based)로 비선형성 반영

---

### 2. 예산 배분

**Binet & Field — "The Long and the Short of It" (IPA, 2013)**
- 최적 예산 비율: 브랜드 빌딩 60% / 단기 활성화 40%
- 성장 브랜드(IT주변기기)는 TOF+MOF 50-60% / BOF 40-50%로 조정
- 코드: `IDEAL_BUDGET_RATIOS` 설정값

**Byron Sharp — "How Brands Grow" (2010)**
- 핵심: 구매자 기반 확장(reach maximization)이 성장의 핵심
- Mental/Physical Availability가 시장 점유율 결정
- 코드: 오디언스 형태 점수(See:Think:Do 비율 평가)

**Equimarginal Principle**
- 경제학 원리: 한계 수익이 동일해지도록 자원 배분 시 전체 수익 극대화
- dR/dB_TOF = dR/dB_MOF = dR/dB_BOF
- 코드: `BudgetOptimizer` 클래스

---

### 3. 수학적 모델

**Beta-Binomial (베이지안 전환율 추정)**
- posterior ~ Beta(alpha + successes, beta + trials - successes)
- 데이터 부족 시 사전분포(prior)가 안정적 추정 제공
- IT주변기기 informative prior: TOF CTR ~1%, MOF engagement ~10%, BOF CVR ~2%
- data_strength: 관측 데이터 vs 사전분포의 상대적 영향력
- 코드: `BetaBinomialEstimator` 클래스

**Hill Saturation Function**
- `y = R_max * x^alpha / (x^alpha + gamma^alpha)`
- Meta Robyn / Google Meridian MMM(Media Mix Modeling)에서 표준 사용
- R_max: 이론적 최대 수익, gamma: 반포화점, alpha: 곡선 기울기
- 한계 수익(marginal response): dy/dx — 1원 추가 투입 시 추가 수익
- 코드: `HillSaturation` 클래스
- 제한: 현재 단일 관측점 휴리스틱 추정 (데이터 축적 시 proper fitting 가능)

**Broadbent Adstock Decay**
- `A_t = T_t + lambda * A_{t-1}`
- 광고 이월효과: 오늘의 광고비가 내일 이후에도 효과 지속
- IT주변기기/전자제품: lambda = 0.82 (반감기 ~3.5주)
- FMCG 대비 높은 감쇠율 — 고관여 제품의 긴 구매 결정 주기 반영
- 코드: `AdstockDecay` 클래스

---

### 4. 빈도 이론

**Krugman 3-Hit Theory (1972)**
- 3회 노출이 인지 효과 발생의 최소 조건
  - 1회: "이게 뭐지?" (인지)
  - 2회: "이건 나와 관련 있어" (관련성)
  - 3회: 행동 유도
- 디지털 환경 적용: 최적 빈도 3-7회 (매체 혼잡도 고려)
- 7회 초과 시 광고 피로도(ad fatigue) 증가
- 코드: `FunnelHealthScorer` 빈도 건강 점수, `ActionItemGenerator` 빈도 액션

---

### 5. 건강도 점수 설계

**5축 100점 체계 (자체 설계, scoring.py 패턴 참조)**

| 영역 | 배점 | 근거 |
|------|------|------|
| 예산 균형 | 20점 | Binet & Field 이상 비율 대비 편차 |
| 전환 흐름 | 25점 | 업계 벤치마크 CTR/CVR 범위 부합도 |
| 오디언스 형태 | 20점 | Byron Sharp See:Think:Do 비율 |
| 단계별 효율 | 20점 | 각 단계 ROAS 목표 대비 달성도 |
| 빈도 건강 | 15점 | Krugman 3-7회 적정 범위 |

등급: S(>=80) / A(>=65) / B(>=50) / C(>=35) / D(<35)

---

### 6. 자동 분류 알고리즘

**4-시그널 가중 합산 (자체 설계)**

| 시그널 | 가중치 | 방법 |
|--------|--------|------|
| 캠페인명 패턴 | 40% | "인지/branding/영상" -> TOF, "트래픽/관심" -> MOF, "전환/리타겟/구매" -> BOF |
| 전환유형 비율 | 25% | purchases 비중 높으면 BOF, content_views 높으면 TOF |
| 성과지표 패턴 | 20% | CTR/CPM/CVR이 각 단계 벤치마크에 부합하는 정도 |
| 도달/빈도 | 15% | 높은 reach + 낮은 frequency -> TOF, 낮은 reach + 높은 frequency -> BOF |

결과: 확률적 분류 (예: BOF 82% / MOF 13% / TOF 5%)

---

## IT주변기기/키보드 카테고리 벤치마크

| 지표 | TOF | MOF | BOF | 출처 |
|------|-----|-----|-----|------|
| CTR | 0.3-0.8% | 0.8-2.0% | 1.0%+ | Naver GFA 내부 데이터 + 업계 평균 |
| CPM | ~3,000원 이하 | 3,000-8,000원 | 8,000원+ | 네이버 GFA 경매 특성 |
| CVR | 0-0.3% | 0.3-1.5% | 1.5%+ | 이커머스 평균 (Shopify: 1.4%) |
| 예산비율 | 25-30% | 25-30% | 40-50% | Binet & Field 조정 |
| 적정빈도 | 1-3회 | 3-5회 | 5-7회 | Krugman + Meta 가이드 |

---

## 향후 개선 방향

1. **Hill 파라미터 proper fitting**: 4주+ 일별 데이터 축적 후 least-squares 또는 bayesian fitting
2. **Markov Chain Attribution**: 채널/캠페인 간 전환 경로 기여도 분석
3. **Shapley Value**: 협력 게임 이론 기반 공정한 기여도 배분
4. **Uplift Modeling**: 광고 노출 vs 비노출 그룹 간 증분 효과 측정
5. **LTV 기반 최적화**: 단기 ROAS 대신 고객 생애 가치 기반 퍼널 최적화
6. **자동 학습**: 분류 결과를 피드백으로 활용하여 가중치 자동 조정
