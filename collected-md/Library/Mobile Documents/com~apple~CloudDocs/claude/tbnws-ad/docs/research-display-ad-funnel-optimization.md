# 디스플레이 광고 퍼널 최적화: 학술적/이론적 프레임워크 리서치

> 연구일: 2026-02-21
> 프로젝트: 네이버 GFA 키보드 광고 최적화
> 목적: DA 퍼널 전략 수립을 위한 학술 이론 기반 구축

---

## 1. 디지털 광고 퍼널 모델의 진화

### 1.1 AIDA Model (1898, Elias St. Elmo Lewis)

**구조:** Attention -> Interest -> Desire -> Action

**한계:**
- 선형적/일방향 모델로 디지털 환경의 비선형적 소비자 여정을 반영 못함
- 회사 중심(company-centric) 관점: "Attention을 획득하겠다"는 발상 자체가 광고주 시각
- 구매 후 관계(post-purchase)를 완전히 무시
- Kaushik의 비판: "AIDA 류 프레임워크는 기본적으로 Think 단계의 하단부터 시작하며, 첫 목표가 Acquisition"

**디스플레이 광고 적용:**
```
AIDA Stage    | DA 역할           | KPI
-----------   | ---------------   | --------
Attention     | 배너 노출(CPM)     | Impressions, Reach
Interest      | 클릭 유도(CPC)     | CTR, CPC
Desire        | 리타겟팅           | Engagement Rate
Action        | 전환 추적(CPA)     | CVR, CPA, ROAS
```

### 1.2 See-Think-Do-Care (Avinash Kaushik, Google)

**핵심 철학:** 인구통계/심리통계가 아닌 "행동 의도(behavioral intent)"로 오디언스를 정의

**구조:**
```
Stage | 정의                                    | 오디언스 크기
------|----------------------------------------|-------------
See   | 가장 큰 적격 도달가능 오디언스(qualified addressable) | ████████████ 최대
Think | 특정 것을 고려 중인 오디언스                     | ████████ 중간
Do    | 구매를 하려는 오디언스 하위집합                    | ████ 소규모
Care  | 기존 고객 (2회 이상 구매)                      | ██ 최소
```

**핵심 통찰 (실무 적용):**
- "많은 사이트가 Do 단계만 커버하여, See/Think 단계 소비자를 놓치고 있다" (Kaushik)
- "미디어 지출에서 See/Think에 돈을 쓰는 곳은 없고, Do에만 집중한다. 그것이 가장 어려운 곳인데도" (Grehan)
- 채널-스테이지 정렬: SEO는 See/Think/Do 효과적, 소셜은 See 중심, YouTube는 Think에 탁월

**네이버 GFA 키보드 광고 적용:**
```
See   -> GFA 배너: "키보드" 카테고리 전체 관심자 타겟 (넓은 관심사 타겟팅)
Think -> GFA 리타겟팅 1차: 키크론 관련 검색/방문 이력자 (좁은 행동 타겟팅)
Do    -> SA 검색광고: "키크론 구매", "키크론 K8 가격" 등 구매 의도 키워드
Care  -> GFA 리타겟팅: 기존 구매자 대상 신제품/액세서리 노출
```

### 1.3 Messy Middle (Google, 2020)

**핵심 개념:** 트리거(Trigger)와 구매(Purchase) 사이에 Explore(탐색, 확장)과 Evaluate(평가, 축소)가 반복되는 비선형 루프

```
[Trigger] --> [ Explore <--> Evaluate ] --> [Purchase]
                    (확장)      (축소)
                  반복 루프
```

**행동 편향 6가지 (인지적 바이어스):**
1. Category Heuristics: 카테고리 내 핵심 스펙으로 빠른 판단
2. Authority Bias: 전문가/권위자 의견 신뢰
3. Social Proof: 리뷰, 평점, 추천 수
4. Power of Now: 즉시 혜택의 매력
5. Scarcity Bias: 한정 수량/기간 제한
6. Power of Free: 무료 요소의 과대 평가

**2024년 최신 연구 결과 (Google/The Behavioural Architects):**
- 행동과학 원칙 + 1차 데이터를 결합하면 **최대 43%의 소비자가 브랜드 선호를 전환**
- 동일 예산을 5개 채널에 분산하면 브랜드 임팩트 **234% 증가**
- Google Media Lab: 중간 퍼널(디스플레이) 강화 시 **16배 더 많은 매출** 귀속 가능

**GFA 적용 시사점:**
- 디스플레이 광고 소재에 6가지 편향 중 적어도 2-3개를 적극 활용
- Category Heuristics: "무선 + 저소음 + 맥 호환" 등 핵심 스펙 강조
- Social Proof: "네이버 쇼핑 리뷰 4.8" 등 수치 삽입
- Power of Now: "지금 구매 시 키캡 세트 증정"

---

## 2. Attribution Modeling (기여도 모델링)

### 2.1 전통적 규칙 기반 모델

```
모델            | 수식                                    | DA에서의 문제점
---------------|----------------------------------------|---------------
Last-Click     | Credit(last) = 1, 나머지 = 0            | DA의 간접 기여 완전 무시
First-Click    | Credit(first) = 1, 나머지 = 0           | 전환 기여 과대평가
Linear         | Credit(i) = 1/n (모든 터치포인트 균등)    | 핵심 터치포인트 구별 불가
Time-Decay     | Credit(i) = 2^((t_i - t_conv)/halflife) | DA(초기 접점) 과소평가
Position-Based | First=40%, Last=40%, 중간=20%/(n-2)     | 임의적 가중치
```

### 2.2 Shapley Value Attribution

**이론적 배경:** Lloyd Shapley (1953, 2012 노벨 경제학상), 협력 게임 이론

**핵심 수식:**
```
phi_i(v) = SUM over S in N\{i} of [ |S|! * (n-|S|-1)! / n! ] * [ v(S union {i}) - v(S) ]
```

여기서:
- `phi_i(v)` = 플레이어 i의 Shapley 값 (채널 i의 기여도)
- `S` = i를 포함하지 않는 모든 부분집합
- `n` = 전체 플레이어(채널) 수
- `v(S)` = 연합 S의 가치 함수 (해당 채널 조합의 전환 가치)
- `v(S union {i}) - v(S)` = 채널 i의 한계 기여도

**4가지 공리적 속성:**
1. Efficiency(효율성): 전체 전환 가치가 빠짐없이 배분됨
2. Symmetry(대칭성): 동일 기여 채널은 동일 크레딧
3. Additivity(가법성): 두 게임의 합 = 각 Shapley 값의 합
4. Null Player(영 플레이어): 기여도 0인 채널은 크레딧 0

**코드 구현용 계산식 (Python):**
```python
from itertools import combinations
import math

def shapley_value(channels, value_function):
    """
    channels: list of channel names, e.g. ['GFA', 'SA', 'Blog']
    value_function: dict mapping frozenset of channels -> conversion value
                    e.g. {frozenset(['GFA']): 10, frozenset(['SA']): 30, ...}
    """
    n = len(channels)
    shapley = {ch: 0.0 for ch in channels}

    for ch in channels:
        for size in range(0, n):
            # ch를 제외한 나머지에서 size개를 뽑는 모든 조합
            others = [c for c in channels if c != ch]
            for subset in combinations(others, size):
                S = frozenset(subset)
                S_with_i = S | {ch}

                # 한계 기여도
                marginal = value_function.get(S_with_i, 0) - value_function.get(S, 0)

                # 가중치: |S|! * (n - |S| - 1)! / n!
                weight = math.factorial(len(S)) * math.factorial(n - len(S) - 1) / math.factorial(n)

                shapley[ch] += weight * marginal

    return shapley

# 예시: GFA(디스플레이), SA(검색), Blog(콘텐츠)
value_func = {
    frozenset(): 0,
    frozenset(['GFA']): 5,      # GFA만 단독: 전환 5건
    frozenset(['SA']): 30,       # SA만 단독: 전환 30건
    frozenset(['Blog']): 2,      # Blog만 단독: 전환 2건
    frozenset(['GFA', 'SA']): 45,        # GFA+SA 시너지: 전환 45건
    frozenset(['GFA', 'Blog']): 10,
    frozenset(['SA', 'Blog']): 35,
    frozenset(['GFA', 'SA', 'Blog']): 50  # 전체: 전환 50건
}
```

**계산량 문제:** 채널이 n개면 2^n개의 연합을 평가해야 함. n>15이면 근사 알고리즘 필요.

### 2.3 Markov Chain Attribution

**핵심 개념:** 고객 여정을 마르코프 체인으로 모델링. 채널 간 전이 확률(transition probability)로 각 채널의 기여도를 "제거 효과(removal effect)"로 측정.

**수학적 구조:**

1단계: 전이 행렬(Transition Matrix) 구축
```
         Start   GFA    SA    Blog   Conv   Null
Start  [  0    0.4    0.3   0.2    0      0.1  ]
GFA    [  0    0.1    0.4   0.1    0.2    0.2  ]
SA     [  0    0.1    0.1   0.05   0.6    0.15 ]
Blog   [  0    0.2    0.3   0.1    0.15   0.25 ]
Conv   [  0    0      0     0      1      0    ]  <- 흡수 상태
Null   [  0    0      0     0      0      1    ]  <- 흡수 상태
```

2단계: 전체 전환 확률 계산
```
P(conversion) = 흡수 마르코프 체인의 기본 행렬(fundamental matrix) N = (I - Q)^(-1) 에서
               Start에서 Conversion으로의 흡수 확률
```

3단계: Removal Effect 계산
```
Removal_Effect(channel_i) = [ P(conv_total) - P(conv_without_i) ] / P(conv_total)
```

4단계: 정규화하여 전환 배분
```
Attribution(channel_i) = Removal_Effect(i) / SUM(Removal_Effect(all)) * Total_Conversions
```

**코드 구현 (Python):**
```python
import numpy as np

def markov_removal_effect(transition_matrix, channel_names, channel_to_remove):
    """
    transition_matrix: (n x n) numpy array
    channel_names: list mapping index to name
    channel_to_remove: name of channel to remove

    Returns: P(conversion) with channel removed
    """
    idx = channel_names.index(channel_to_remove)
    # 해당 채널의 모든 전이를 Null(흡수 상태)로 리다이렉트
    modified = transition_matrix.copy()
    null_idx = channel_names.index('Null')
    modified[idx, :] = 0
    modified[idx, null_idx] = 1  # 해당 채널 도달 시 즉시 이탈

    # 흡수 마르코프 체인의 전환 확률 계산
    # Q = transient states 간 전이 확률 부분 행렬
    # N = (I - Q)^(-1) = fundamental matrix
    # B = N * R (흡수 확률)
    transient_idx = [i for i, name in enumerate(channel_names)
                     if name not in ['Conv', 'Null']]
    absorbing_idx = [channel_names.index('Conv'), channel_names.index('Null')]

    Q = modified[np.ix_(transient_idx, transient_idx)]
    R = modified[np.ix_(transient_idx, absorbing_idx)]

    N = np.linalg.inv(np.eye(len(transient_idx)) - Q)  # Fundamental matrix
    B = N @ R  # Absorption probabilities

    start_idx_in_transient = [i for i, idx_val in enumerate(transient_idx)
                              if channel_names[idx_val] == 'Start'][0]
    conv_idx_in_absorbing = 0  # Conv is first absorbing state

    return B[start_idx_in_transient, conv_idx_in_absorbing]
```

**Shapley vs Markov 비교:**
```
기준            | Shapley Value        | Markov Chain
---------------|---------------------|------------------
순서 고려        | 기본적으로 순서 무시    | 순서(전이 확률) 반영
계산 복잡도      | O(2^n) 지수적         | O(n^3) 행렬 역산
이론적 근거      | 게임 이론 공리          | 확률론적 모델
데이터 요구      | 채널 조합별 전환 데이터   | 개별 사용자 여정 시퀀스
DA 기여도 측정   | 연합(coalition) 기반    | 제거 효과(removal effect) 기반
```

**학술 참고:**
- Shao & Li (2011), "Data-Driven Multi-Touch Attribution Models", ACM SIGKDD
- Li & Kannan (2014), "Attributing conversions in a multichannel online marketing environment", Journal of Marketing Research
- Zhao et al. (2018), "Shapley Value Methods for Attribution Modeling in Online Advertising", arXiv:1804.05327
- WWW 2019: "Shapley Meets Uniform: An Axiomatic Framework for Attribution in Online Advertising"

---

## 3. 퍼널 단계별 최적 예산 배분

### 3.1 Les Binet & Peter Field - "The Long and the Short of It" (2013)

**연구 기반:** IPA Databank 996개 광고 효과 사례, 700개 브랜드, 83개 업종, 30년+

**60/40 법칙:**
```
최적 예산 배분 = Brand Building 60% : Sales Activation 40%

Brand Building (60%)              Sales Activation (40%)
- 장기적 효과 (6개월+)            - 단기적 효과 (즉시~수주)
- 감성적/정서적 메시지              - 이성적/정보적 메시지
- 넓은 타겟 (카테고리 전체)         - 좁은 타겟 (구매 의도자)
- 정신적 가용성 구축                - 즉각적 행동 유도
- Reach 극대화                    - Targeting 정밀화
```

**업종별 조정 가이드:**
```
업종/상황           | Brand : Activation | 근거
------------------|-------------------|------
일반 (기본값)       | 60 : 40           | IPA 데이터 평균
금융 서비스          | 70-80 : 20-30     | 높은 신뢰 필요
스타트업/신규 진입    | 30-40 : 60-70     | 즉각적 ROI 필요, 현금흐름 확보
B2B               | 55 : 45           | 긴 구매 사이클
FMCG              | 65 : 35           | 정서적 브랜드 연결 중요
```

**"두 가지를 동시에 하지 마라" (Binet의 핵심 주장):**
- Brand building과 performance를 한 광고에서 동시에 하면 둘 다 약해짐
- "브랜드가 강하면, 퍼포먼스 마케팅 반응도가 높아진다 - 이미 워밍업이 되어있으니까"
- 각각 별도 캠페인으로 운영해야 함

**GFA 적용:**
```
키크론 월 광고비 ~250만원 기준 (SA 포함 전체)
SA(검색광고) = Do 단계 = Sales Activation = ~40% = ~100만원
GFA(디스플레이) = See/Think 단계 = Brand Building = ~60% = ~150만원

단, 현재 SA ROAS 5,000~10,000%이므로:
- 단기: SA 비중 유지하면서 GFA를 추가 예산으로 집행
- 중기: 전체 예산 증가 시 60/40 비율로 수렴
```

### 3.2 Byron Sharp - "How Brands Grow" (Ehrenberg-Bass Institute)

**핵심 원칙들:**

1. **Reach > Frequency (도달 > 빈도)**
   - 브랜드 성장의 주요 동력은 기존 고객 충성도 심화가 아닌 구매자 기반 확대
   - "모든 브랜드에는 수많은 경량 구매자(light buyers)가 있다. 이들은 드물게 구매하지만 너무 많아서 매출에 크게 기여한다"

2. **Double Jeopardy Law (이중 위험 법칙)**
   - 시장 점유율이 낮은 브랜드는 (1) 구매자 수도 적고 (2) 구매 빈도도 낮음
   - 성장 = 구매자 수 증가 (빈도 증가 효과는 미미)

3. **Mental Availability + Physical Availability**
   - Mental Availability: "브랜드가 구매 상황에서 떠오를 확률" = 광고의 역할
   - Physical Availability: "브랜드를 실제로 구매할 수 있는 접근성" = 유통의 역할

4. **Distinctive Assets > Differentiation**
   - "의미 있는 차별화보다 무의미한 독특함을 추구하라"
   - 색상, 로고, 디자인 등 감각적 단서(sensory cues)의 일관성

**GFA 적용 시사점:**
```
Sharp의 원칙              | GFA 전략
------------------------|----------------------------------
Reach 극대화             | 넓은 관심사 타겟팅, CPM 입찰 전략
Light Buyer 도달         | 키보드 카테고리 전체 관심자 타겟
Distinctive Assets      | 키크론 브랜드 컬러/디자인 일관된 배너
연속적 노출               | Always-on 캠페인 (간헐적이 아닌 상시 운영)
```

### 3.3 예산 배분 최적화 공식 (Equimarginal Principle)

경제학의 한계 균등 원칙을 광고에 적용:

```
최적 조건: dR/dB_1 = dR/dB_2 = ... = dR/dB_n

여기서:
  R = 총 수익 (Revenue)
  B_i = 채널 i에 대한 예산
  dR/dB_i = 채널 i의 한계 수익 (Marginal Revenue)

즉, 모든 채널의 한계 수익(마지막 1원 투입의 수익)이 같아질 때 최적
```

**실무 계산 방법:**
```python
def optimal_budget_allocation(total_budget, channels, response_curves):
    """
    total_budget: 총 예산
    channels: 채널 목록
    response_curves: 각 채널의 response function (budget -> revenue)

    최적화: 모든 채널의 marginal ROAS가 같아질 때까지 조정
    """
    # 각 채널의 marginal ROAS = d(Revenue)/d(Budget)
    # Hill function 기반:
    # Revenue_i = R_max_i * (Budget_i^alpha_i) / (Budget_i^alpha_i + gamma_i^alpha_i)
    # Marginal ROAS_i = d(Revenue_i)/d(Budget_i)
    #                 = R_max_i * alpha_i * gamma_i^alpha_i * Budget_i^(alpha_i - 1)
    #                   / (Budget_i^alpha_i + gamma_i^alpha_i)^2
    pass
```

---

## 4. 전환 퍼널 최적화 수학적 모델

### 4.1 베이지안 전환율 추정 (Beta-Binomial Model)

**모델 구조:**

```
사전 분포 (Prior):     pi ~ Beta(alpha, beta)
데이터 (Likelihood):   y | pi ~ Binomial(n, pi)
사후 분포 (Posterior): pi | y ~ Beta(alpha + y, beta + n - y)
```

여기서:
- `pi` = 전환율 (추정 대상)
- `alpha, beta` = 사전 분포 파라미터 (사전 지식 반영)
- `n` = 총 시행 수 (방문/클릭 수)
- `y` = 성공 수 (전환 수)

**사후 분포의 통계량:**
```
사후 평균:    E[pi|y] = (alpha + y) / (alpha + beta + n)
사후 최빈값:  Mode[pi|y] = (alpha + y - 1) / (alpha + beta + n - 2)
사후 분산:    Var[pi|y] = (alpha + y)(beta + n - y) / [(alpha + beta + n)^2 * (alpha + beta + n + 1)]
95% 신뢰구간: Beta 분포의 2.5%, 97.5% 분위수
```

**무정보적 사전분포:** Beta(1, 1) = Uniform(0, 1)

**코드 구현:**
```python
from scipy import stats
import numpy as np

class BayesianFunnelEstimator:
    """퍼널 각 단계의 전환율을 베이지안으로 추정"""

    def __init__(self):
        # 각 단계별 사전 분포 파라미터 (무정보적 시작)
        self.priors = {
            'impression_to_click': {'alpha': 1, 'beta': 1},   # CTR
            'click_to_landing': {'alpha': 1, 'beta': 1},      # 랜딩 도달률
            'landing_to_conversion': {'alpha': 1, 'beta': 1},  # CVR
        }

    def update(self, stage, successes, trials):
        """새 데이터로 사후 분포 업데이트"""
        prior = self.priors[stage]
        posterior_alpha = prior['alpha'] + successes
        posterior_beta = prior['beta'] + (trials - successes)

        self.priors[stage] = {'alpha': posterior_alpha, 'beta': posterior_beta}

        return {
            'mean': posterior_alpha / (posterior_alpha + posterior_beta),
            'mode': (posterior_alpha - 1) / (posterior_alpha + posterior_beta - 2)
                    if posterior_alpha > 1 and posterior_beta > 1 else None,
            'std': np.sqrt(posterior_alpha * posterior_beta /
                          ((posterior_alpha + posterior_beta)**2 *
                           (posterior_alpha + posterior_beta + 1))),
            'ci_95': stats.beta.interval(0.95, posterior_alpha, posterior_beta),
            'alpha': posterior_alpha,
            'beta': posterior_beta,
        }

    def predict_conversions(self, stage, future_trials, n_samples=10000):
        """미래 전환 수 예측 (사후 예측 분포)"""
        prior = self.priors[stage]
        # 사후 분포에서 전환율 샘플링
        rate_samples = stats.beta.rvs(prior['alpha'], prior['beta'], size=n_samples)
        # 각 전환율에서 전환 수 샘플링
        conversion_samples = stats.binom.rvs(future_trials, rate_samples)
        return {
            'mean': np.mean(conversion_samples),
            'median': np.median(conversion_samples),
            'ci_95': (np.percentile(conversion_samples, 2.5),
                     np.percentile(conversion_samples, 97.5)),
        }

# 사용 예시: GFA 키크론 캠페인
estimator = BayesianFunnelEstimator()

# 지난 주 데이터: 100,000 노출, 500 클릭, 15 전환
estimator.update('impression_to_click', successes=500, trials=100000)  # CTR
estimator.update('click_to_landing', successes=480, trials=500)         # 랜딩 도달
estimator.update('landing_to_conversion', successes=15, trials=480)     # CVR

# 다음 주 20만 노출 시 예상 클릭 수
prediction = estimator.predict_conversions('impression_to_click', future_trials=200000)
```

**학술 참고:** Iyengar & Singal (2024), "Model-Free Approximate Bayesian Learning for Large-Scale Conversion Funnel Optimization", Production and Operations Management, Vol.33(3), pp.775-794

### 4.2 퍼널 단계 간 최적 오디언스 크기 비율

**기본 모델: 역삼각형 비율**

```
단계별 전환율을 c_1, c_2, ..., c_k라 하면:

Stage 1 (See):   Audience = N
Stage 2 (Think): Audience = N * c_1
Stage 3 (Do):    Audience = N * c_1 * c_2
...
Stage k:         Audience = N * PRODUCT(c_i, i=1..k-1)

최적 N을 결정하려면:
  목표 전환 수 = T
  필요 최상위 오디언스 = T / PRODUCT(c_i, i=1..k)
```

**실무 벤치마크 비율 (디스플레이 광고):**
```
See(인지)    : Think(고려)  : Do(전환) = 100 : 10~20 : 1~3

즉, 전환 1건을 얻으려면:
  - Do 단계 오디언스: ~33-100명 필요 (CVR 1-3%)
  - Think 단계 오디언스: ~500-1000명 필요 (CTR 1-2%)
  - See 단계 오디언스: ~50,000-100,000명 필요
```

**코드:**
```python
def calculate_required_audience(target_conversions, funnel_rates):
    """
    target_conversions: 목표 전환 수
    funnel_rates: dict of stage -> conversion rate
                  e.g. {'see_to_think': 0.005, 'think_to_do': 0.03, 'do_to_convert': 0.02}
    """
    total_rate = 1.0
    stages = {}
    for stage, rate in reversed(list(funnel_rates.items())):
        total_rate *= rate
        stages[stage] = target_conversions / total_rate

    return {
        'required_top_audience': target_conversions / total_rate,
        'stages': stages,
        'overall_conversion_rate': total_rate,
    }

# 예시: 키크론 GFA 월 50건 전환 목표
result = calculate_required_audience(
    target_conversions=50,
    funnel_rates={
        'see_to_think': 0.005,      # 배너 CTR 0.5%
        'think_to_do': 0.03,        # 리타겟팅 재방문율 3%
        'do_to_convert': 0.02,      # 전환율 2%
    }
)
# 필요 See 단계 오디언스: 50 / (0.005 * 0.03 * 0.02) = 16,666,667
```

### 4.3 Diminishing Returns (한계 효용 체감) 모델

**3가지 주요 함수:**

#### A. Power Function (멱함수, 기본형)
```
y = a * x^beta    (0 < beta <= 1)

여기서:
  y = 성과 (전환, 매출)
  x = 투입 (예산, 노출)
  a = 스케일 파라미터
  beta = 체감 파라미터 (1에 가까울수록 선형, 0에 가까울수록 급격한 체감)

한계 효용: dy/dx = a * beta * x^(beta-1)
```

#### B. Log Function (로그 함수)
```
y = a + b * ln(x)

한계 효용: dy/dx = b / x
  -> 투입이 2배면 한계 효용은 1/2
```

#### C. Hill Function (S-curve, MMM 표준)
```
y = R_max * x^alpha / (x^alpha + gamma^alpha)

여기서:
  R_max = 최대 도달 가능 성과 (포화점)
  alpha = 형상 파라미터 (S커브 기울기, 클수록 급격한 S자)
  gamma = 반포화점 (y = R_max/2 되는 x값, 변곡점)

한계 효용: dy/dx = R_max * alpha * gamma^alpha * x^(alpha-1) / (x^alpha + gamma^alpha)^2
```

**Meta Robyn 구현 (Hill Saturation):**
```
media_saturated = 1 / (1 + (gamma / media_adstocked)^alpha)

alpha: 형상 파라미터 (클수록 S자 형태)
gamma: 변곡점 (0~1 사이로 정규화, 미디어 변수의 inflection point)
```

**코드 구현:**
```python
import numpy as np

def hill_function(x, R_max, alpha, gamma):
    """Hill saturation function"""
    return R_max * (x ** alpha) / (x ** alpha + gamma ** alpha)

def hill_marginal(x, R_max, alpha, gamma):
    """Hill function의 한계 효용 (1차 도함수)"""
    return R_max * alpha * (gamma ** alpha) * (x ** (alpha - 1)) / \
           (x ** alpha + gamma ** alpha) ** 2

def find_optimal_spend(R_max, alpha, gamma, target_marginal_roas=1.0):
    """한계 ROAS가 target_marginal_roas가 되는 지출 수준 (최적 투입점)"""
    from scipy.optimize import brentq
    f = lambda x: hill_marginal(x, R_max, alpha, gamma) - target_marginal_roas
    # 한계 ROAS = 1이 되는 점 = 투입 중단 기준점
    return brentq(f, 1, gamma * 100)

# 예시: GFA 키크론 캠페인 파라미터 추정
# R_max = 500만원 (최대 가능 전환매출/월)
# alpha = 2.0 (S-curve 형태)
# gamma = 150만원 (150만원 투입 시 반포화)
spend_range = np.linspace(10000, 3000000, 100)
revenue = hill_function(spend_range, 5000000, 2.0, 1500000)
marginal = hill_marginal(spend_range, 5000000, 2.0, 1500000)
```

---

## 5. CPM과 퍼널 위치의 관계 모델

### 5.1 퍼널 위치별 비용-성과 구조

```
퍼널 단계 | 과금 모델 | CPM 수준 | 오디언스 크기 | CVR    | 핵심 지표
---------|---------|---------|-----------|--------|--------
Upper    | CPM     | 낮음     | 매우 넓음   | 매우 낮음 | Reach, CPM
Mid      | CPM/CPC | 중간     | 중간       | 낮음    | CTR, CPC
Lower    | CPC/CPA | 높음     | 매우 좁음   | 높음    | CVR, CPA, ROAS
```

### 5.2 수학적 관계식

**CPM -> CTR -> CPC -> CVR -> CPA -> ROAS 연쇄:**
```
CPC = CPM / (CTR * 1000)
CPA = CPC / CVR = CPM / (CTR * CVR * 1000)
ROAS = AOV / CPA = AOV * CTR * CVR * 1000 / CPM
```

**퍼널 전체 비용 효율:**
```
전체 ROAS = Total_Revenue / Total_Cost
         = SUM(Revenue_i) / SUM(Cost_i)  for all funnel stages i

여기서 각 단계의 Revenue는 직접 전환뿐 아니라 후속 단계에 대한 간접 기여를 포함
```

**Upper Funnel의 간접 기여 모델:**
```
Revenue_upper = Direct_Revenue_upper + Assisted_Revenue_upper

Assisted_Revenue_upper = P(upper -> mid) * P(mid -> lower) * P(lower -> conv) * AOV * N_upper

여기서:
  N_upper = upper funnel 도달 오디언스 수
  P(upper -> mid) = upper에서 mid로의 전환율 (리타겟팅 풀 진입)
  P(mid -> lower) = mid에서 lower로의 전환율
  P(lower -> conv) = lower에서 최종 전환율
  AOV = 평균 주문 가치 (Average Order Value)
```

### 5.3 실무 벤치마크

**네이버 GFA 예상 수치 (키보드 카테고리):**
```
퍼널 단계        | CPM (원)    | CTR      | CVR      | 예상 CPA
See (넓은 타겟)  | 1,000~3,000 | 0.1~0.3% | -        | - (직접전환 없음)
Think (리타겟팅)  | 3,000~8,000 | 0.5~1.5% | 0.5~1.5% | 30,000~100,000원
Do (구매의도)     | 5,000~15,000| 1.0~3.0% | 2.0~5.0% | 10,000~50,000원
```

**핵심 인사이트 (Measured.com 연구):**
- "라스트터치 어트리뷰션 플랫폼은 하위 퍼널을 거의 항상 과대보고하고 상위 퍼널을 과소보고한다"
- "브랜드들은 종종 리타겟팅에 과도하게 투자하며, 프로스펙팅/인지도에 예산을 이동하면 수익이 크게 증가한다"

---

## 6. Audience Overlap & Frequency Capping

### 6.1 Herbert Krugman의 Three-Hit Theory (1972)

**핵심 이론:** 광고 노출의 심리학적 3단계

```
노출 1: "What is it?"    - 호기심 (새로운 자극 인식)
노출 2: "What of it?"    - 인식 (개인적 관련성 평가)
노출 3: "Reminder/Sale"  - 결정/리마인더 (메시지 내면화)
노출 4+: 3번째의 반복     - 추가 심리적 효과 없음 (체감 시작)
```

**현대적 수정 (2024 연구):**
- Krugman(1972)은 일일 500건 메시지 노출 환경에서 연구, 현재는 5,000건+
- Qriously 연구: 빈도 15-25회가 최고 태도 변화 유발
- Oracle Data Cloud: 80% 캠페인에서 첫 노출이 가장 큰 증분 매출 기여, 단 후속 노출도 양의 증분 수익
- ARF(Advertising Research Foundation): 평균적으로 5-6회 노출 후 효과 정체

### 6.2 도달(Reach) 중복 제거 공식: Sainsbury Formula

**기본 공식:**
```
단일 매체, n회 노출:
  Reach = 1 - (1 - p)^n

여기서:
  p = 1회 노출당 도달 확률 (= 매체 도달률)
  n = 노출 횟수

복수 매체:
  Reach = 1 - (1 - p_1)(1 - p_2)(1 - p_3)...(1 - p_m)

여기서:
  p_i = 매체 i의 도달률 (Audience_i / Total_Universe)
```

**핵심 가정:** 각 매체의 도달은 독립적(random duplication)

**코드 구현:**
```python
def sainsbury_reach(audiences, universe):
    """
    audiences: list of audience sizes for each media vehicle
    universe: total target universe size
    Returns: estimated unique reach
    """
    prob_not_reached = 1.0
    for audience in audiences:
        p = audience / universe
        prob_not_reached *= (1 - p)
    reach_pct = 1 - prob_not_reached
    return reach_pct * universe

# 예시: GFA 3개 캠페인의 중복 제거 도달 추정
# 키보드 관심자 유니버스: 500만명
# 캠페인A (넓은 타겟): 100만명 도달
# 캠페인B (리타겟팅): 10만명 도달
# 캠페인C (유사 오디언스): 50만명 도달
unique_reach = sainsbury_reach([1000000, 100000, 500000], 5000000)
# = 5,000,000 * (1 - (1-0.2)(1-0.02)(1-0.1))
# = 5,000,000 * (1 - 0.8 * 0.98 * 0.9)
# = 5,000,000 * (1 - 0.7056)
# = 5,000,000 * 0.2944 = 1,472,000명 (단순 합산 160만 vs 중복 제거 147만)
```

### 6.3 최적 노출 빈도 (Optimal Frequency)

**빈도-반응 모델:**
```
Response(f) = R_max * (1 - e^(-k * f))

여기서:
  f = 노출 빈도 (frequency)
  R_max = 최대 반응률 (포화 수준)
  k = 학습 속도 파라미터

한계 반응: dResponse/df = R_max * k * e^(-k * f)
최적 빈도 = 한계 비용 = 한계 수익인 f*에서 결정
```

**실무 기준:**
```
플랫폼/채널      | 권장 주간 빈도 캡 | 근거
---------------|----------------|------
디스플레이 배너    | 3-7회/주        | Krugman 3-hit + 현대 보정
동영상           | 2-4회/주        | 높은 주목도로 낮은 빈도 가능
리타겟팅          | 5-10회/주       | 구매 의도 높아 높은 빈도 허용
소셜 피드         | 3-5회/주        | 피드 피로도 고려
```

### 6.4 Adstock Model (Simon Broadbent, 1979)

**기본 Adstock 공식:**
```
A_t = T_t + lambda * A_(t-1)

여기서:
  A_t = 시점 t의 Adstock (누적 광고 효과)
  T_t = 시점 t의 광고 투입량 (GRP, 노출, 예산)
  lambda = 감쇠율 (0~1, 클수록 효과 오래 지속)

반감기 = ln(0.5) / ln(lambda)
lambda = 0.5^(1/halflife)
```

**업종별 반감기 벤치마크:**
```
업종              | 반감기 (주)  | lambda
-----------------|------------|--------
FMCG             | 2.5        | 0.76
전자제품           | 3-4        | 0.80-0.84
B2B              | 6-8        | 0.88-0.92
고관여 제품 (키보드) | 3-5        | 0.80-0.87
```

**코드 구현:**
```python
def calculate_adstock(spending_series, decay_rate):
    """
    spending_series: 시계열 광고비 데이터 (list or array)
    decay_rate: lambda (감쇠율, 0~1)
    Returns: adstock series
    """
    adstock = [0] * len(spending_series)
    adstock[0] = spending_series[0]
    for t in range(1, len(spending_series)):
        adstock[t] = spending_series[t] + decay_rate * adstock[t-1]
    return adstock

# 예시: 키크론 GFA 주간 광고비
weekly_spend = [500000, 300000, 0, 0, 500000, 300000, 200000]  # 7주
decay = 0.82  # 전자제품 반감기 ~3.5주
adstock = calculate_adstock(weekly_spend, decay)
# 광고 중단 주에도 adstock > 0 (이월 효과)
```

---

## 7. Media Mix Modeling (MMM)

### 7.1 기본 모델 구조

**Multiplicative Model (가장 널리 사용):**
```
Sales = exp(alpha) * X_1^(beta_1) * X_2^(beta_2) * ... * X_n^(beta_n) * Seasonality * epsilon

로그 변환 후 선형 회귀:
ln(Sales) = alpha + beta_1*ln(X_1) + beta_2*ln(X_2) + ... + beta_n*ln(X_n) + controls + error
```

**Adstock + Saturation 결합 모델 (현대적 표준):**
```
Sales = Base + SUM_i [ f_sat_i( g_adstock_i(X_i) ) * beta_i ] + Controls + Error

여기서:
  g_adstock_i(X_i) = geometric adstock transformation
  f_sat_i(.) = Hill saturation function
  beta_i = 채널 i의 계수
  Controls = 계절성, 프로모션, 가격, 경쟁, 거시경제 등
```

### 7.2 SA-DA 시너지 효과 모델링

**교호작용 항(Interaction Term):**
```
Sales = Base + beta_SA*SA + beta_DA*DA + beta_SA_DA*(SA * DA) + Controls + Error

여기서:
  beta_SA = SA 단독 효과 계수
  beta_DA = DA 단독 효과 계수
  beta_SA_DA = SA와 DA의 시너지 계수

시너지 여부 판단:
  beta_SA_DA > 0 이면 양의 시너지 (SA와 DA를 함께 쓸 때 1+1>2)
  beta_SA_DA < 0 이면 대체 관계 (서로 카니발라이즈)
  beta_SA_DA = 0 이면 독립적
```

**Nielsen 연구 (2024):** TV + 디지털 결합 시 캠페인 효과 20% 증가

**코드 구현 (PyMC/Bayesian MMM):**
```python
import pymc as pm
import numpy as np

def bayesian_mmm(data):
    """
    data: DataFrame with columns:
      - sales: 일/주별 매출
      - sa_spend: 검색광고 비용
      - da_spend: 디스플레이광고 비용
      - seasonality: 계절 지표
    """
    with pm.Model() as mmm:
        # Priors
        intercept = pm.Normal('intercept', mu=0, sigma=10)
        beta_sa = pm.HalfNormal('beta_sa', sigma=5)
        beta_da = pm.HalfNormal('beta_da', sigma=5)
        beta_synergy = pm.Normal('beta_synergy', mu=0, sigma=2)
        sigma = pm.HalfNormal('sigma', sigma=1)

        # Adstock transformation
        theta_sa = pm.Beta('theta_sa', alpha=3, beta=3)  # SA decay
        theta_da = pm.Beta('theta_da', alpha=3, beta=3)  # DA decay

        # Hill saturation parameters
        alpha_sa = pm.Gamma('alpha_sa', alpha=3, beta=1)
        gamma_sa = pm.Beta('gamma_sa', alpha=2, beta=2)
        alpha_da = pm.Gamma('alpha_da', alpha=3, beta=1)
        gamma_da = pm.Beta('gamma_da', alpha=2, beta=2)

        # Transform: Adstock -> Saturation
        sa_adstocked = geometric_adstock(data['sa_spend'], theta_sa)
        da_adstocked = geometric_adstock(data['da_spend'], theta_da)
        sa_saturated = hill_saturation(sa_adstocked, alpha_sa, gamma_sa)
        da_saturated = hill_saturation(da_adstocked, alpha_da, gamma_da)

        # Model with interaction term
        mu = (intercept
              + beta_sa * sa_saturated
              + beta_da * da_saturated
              + beta_synergy * sa_saturated * da_saturated  # SA-DA 시너지
              )

        # Likelihood
        sales = pm.Normal('sales', mu=mu, sigma=sigma, observed=data['sales'])

        # Inference
        trace = pm.sample(2000, tune=1000, cores=2)

    return trace
```

### 7.3 채널 기여도(Contribution) 및 ROI 계산

```
Contribution_i = E[Sales | X_i = actual] - E[Sales | X_i = 0]
               = "해당 채널 제거 시 매출 감소분"

ROI_i = Contribution_i / Spend_i

Marginal ROI_i = dContribution_i / dSpend_i
               = 현 투입 수준에서 1원 추가 투입의 기대 수익
```

### 7.4 Open Source MMM 도구

```
도구                | 개발      | 특징
--------------------|---------|---------------------------
Meta Robyn          | Meta    | R 기반, 자동 하이퍼파라미터 튜닝
Google Meridian     | Google  | Python, Bayesian, Reach/Frequency 지원
PyMC-Marketing      | PyMC    | Python, 완전 Bayesian, 유연한 모델 구조
LightweightMMM      | Google  | Python, JAX 기반 (Meridian 전신)
```

---

## 8. 종합 적용: 네이버 GFA 키보드 광고 퍼널 전략

### 8.1 이론 기반 퍼널 설계

```
                            Binet & Field          Sharp
                            Brand 60%              Reach Maximization
                     ┌──────────────────────────────────────┐
                     │                                      │
Stage    Model       │  GFA 전략           예산 비율  KPI     │
------   --------    │  ---------          -------  -----   │
See      STDC-See    │  넓은 타겟 CPM 캠페인  35%     Reach   │
         MessyMiddle │  행동편향 소재 적용               CPM    │
                     │                                      │
Think    STDC-Think  │  리타겟팅 + 유사타겟   25%     CTR     │
         MessyMiddle │  Explore/Evaluate 촉진         CPC    │
                     │                                      │
                     └──────────────────────────────────────┘
                            Activation 40%
                     ┌──────────────────────────────────────┐
Do       STDC-Do     │  SA 검색광고          35%     CVR     │
                     │  + GFA 리마케팅 최종 전환       CPA    │
                     │                           ROAS       │
Care     STDC-Care   │  기존고객 크로스셀      5%      LTV    │
                     │  리텐션 캠페인               Repeat   │
                     └──────────────────────────────────────┘
```

### 8.2 핵심 계산식 요약 (코드 구현 체크리스트)

```
No | 계산식                        | 용도                    | 구현 우선순위
---|------------------------------|------------------------|----------
1  | Beta-Binomial Posterior      | 퍼널 단계별 전환율 추정     | ★★★ High
2  | Hill Saturation Function     | 예산 대비 수익 체감곡선     | ★★★ High
3  | Adstock Decay               | 광고 이월효과 반영          | ★★★ High
4  | Sainsbury Reach Formula      | 중복 제거 도달 추정         | ★★☆ Med
5  | Shapley Value               | 채널별 기여도 공정 배분      | ★★☆ Med
6  | Markov Removal Effect       | 채널 제거 시 전환 감소분     | ★★☆ Med
7  | Equimarginal Principle      | 최적 예산 배분 (한계ROAS 균등) | ★★★ High
8  | SA * DA Interaction Term    | SA-DA 시너지 계수 추정      | ★☆☆ Low (데이터 축적 후)
```

### 8.3 데이터 요구사항 (우리 DB 매핑)

```
이론 모델             | 필요 데이터                  | 현재 DB 테이블             | Gap
--------------------|--------------------------|--------------------------|------
Beta-Binomial       | 노출/클릭/전환 수             | naver_ad_campaign_daily   | GFA 데이터 추가 필요
Hill Saturation     | 일별 광고비 vs 매출           | naver_ad_campaign_daily   | GFA 데이터 추가 필요
Adstock             | 주별 광고비 시계열             | naver_ad_campaign_daily   | 집계 쿼리로 가능
Sainsbury Reach     | 캠페인별 도달 수              | GFA API 필요              | 신규 수집 필요
Shapley Value       | 채널 조합별 전환              | SA + GFA 교차 데이터       | 사용자 여정 데이터 필요
Markov Chain        | 사용자별 터치포인트 시퀀스       | 없음 (GA/GTM 필요)        | 장기 과제
MMM                 | 주별 채널별 비용 + 총매출      | 가능 (SA+GFA 합산)        | GFA 데이터 추가 필요
```

---

## 참고 문헌 및 출처

### 학술 논문
- Shao & Li (2011), "Data-Driven Multi-Touch Attribution Models", ACM SIGKDD
- Li & Kannan (2014), "Attributing conversions in a multichannel online marketing environment", Journal of Marketing Research
- Kannan, Reinartz, & Verhoef (2016), "The path to purchase and attribution modeling", International Journal of Research in Marketing
- Zhao et al. (2018), "Shapley Value Methods for Attribution Modeling in Online Advertising", arXiv:1804.05327
- WWW 2019, "Shapley Meets Uniform: An Axiomatic Framework for Attribution in Online Advertising"
- Iyengar & Singal (2024), "Model-Free Approximate Bayesian Learning for Large-Scale Conversion Funnel Optimization", Production and Operations Management, 33(3), 775-794
- Krugman, H.E. (1972), "Why Three Exposures May Be Enough", Journal of Advertising Research
- Broadbent, S. (1979), "One Way TV Advertisements Work", Journal of the Market Research Society

### 서적
- Binet, L. & Field, P. (2013), "The Long and the Short of It", IPA
- Sharp, B. (2010), "How Brands Grow", Oxford University Press
- Ehrenberg-Bass Institute publications (https://marketingscience.info)

### 프레임워크/도구
- Kaushik, A., "See-Think-Do-Care Framework" (https://www.kaushik.net/avinash/)
- Google, "Messy Middle" (https://www.thinkwithgoogle.com)
- Meta Robyn MMM (https://facebookexperimental.github.io/Robyn/)
- Google Meridian MMM (https://developers.google.com/meridian)
- PyMC-Marketing (https://www.pymc.io)

### 웹 리소스
- Statsig, "Digital Marketing Attribution Models: A Technical Survey"
- Measured.com, "From Tunnel Vision to Funnel Vision"
- AnalyticsArtist, "Advertising Diminishing Returns & Saturation"
- Evan Miller, "Formulas for Bayesian A/B Testing" (https://www.evanmiller.org/bayesian-ab-testing.html)
