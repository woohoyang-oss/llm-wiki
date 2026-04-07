# GFA(보장형 디스플레이 광고) 운영 AI 지식 베이스
> 네이버 GFA(Guaranteed Fixed-price Advertising) 운영 AI 학습용 종합 참조 문서  
> 출처: 학술 논문 종합 (Choi et al. 2020, Yang & Zhai 2022, Fang et al. 2019, Chen et al. 2023 외)

---

## 목차

1. [디스플레이 광고 생태계 개요](#1-디스플레이-광고-생태계-개요)
2. [GFA(보장형 광고)의 구조와 메커니즘](#2-gfa보장형-광고의-구조와-메커니즘)
3. [RTB(비보장형) vs GFA(보장형) 비교](#3-rtb비보장형-vs-gfa보장형-비교)
4. [GFA 인벤토리 예측 모델](#4-gfa-인벤토리-예측-모델)
5. [CTR 예측 이론 및 모델](#5-ctr-예측-이론-및-모델)
6. [CVR(전환율) 예측 및 Delayed Feedback](#6-cvr전환율-예측-및-delayed-feedback)
7. [예산 Pacing 알고리즘](#7-예산-pacing-알고리즘)
8. [타겟팅 이론 및 오디언스 세그먼트](#8-타겟팅-이론-및-오디언스-세그먼트)
9. [광고 성과 측정 및 KPI 체계](#9-광고-성과-측정-및-kpi-체계)
10. [GFA 가격 책정(Pricing) 모델](#10-gfa-가격-책정pricing-모델)
11. [GFA 계약 배분 최적화 알고리즘](#11-gfa-계약-배분-최적화-알고리즘)
12. [AI 운영 의사결정 프레임워크](#12-ai-운영-의사결정-프레임워크)
13. [핵심 수식 및 통계 모델 참조](#13-핵심-수식-및-통계-모델-참조)
14. [참고 논문 목록](#14-참고-논문-목록)

---

## 1. 디스플레이 광고 생태계 개요

### 1.1 시장 구조 (Two-Sided Market)

디스플레이 광고 시장은 **양면 시장(Two-Sided Market)** 구조로 운영된다.

| 참여자 | 역할 | 목표 |
|--------|------|------|
| **광고주 (Advertiser)** | 광고 인상(Impression)을 구매하여 잠재 소비자에게 도달 | ROI 극대화, 브랜드 인지도 향상 |
| **퍼블리셔 (Publisher)** | 사용자 임프레션을 광고 인벤토리로 제공 | 광고 수익 극대화 |
| **중간자 (Intermediary)** | 광고주-퍼블리셔 매칭 최적화 | 거래 효율화 |
| **DSP** (Demand Side Platform) | 광고주의 입찰 및 집행 자동화 | 광고주 ROI 지원 |
| **SSP** (Supply Side Platform) | 퍼블리셔의 인벤토리 관리 및 수익 최적화 | 퍼블리셔 수익 극대화 |
| **DMP** (Data Management Platform) | 오디언스 데이터 통합 및 세그먼트 생성 | 타겟팅 정확도 향상 |

### 1.2 판매 채널 구분

디스플레이 광고는 크게 두 가지 판매 채널로 나뉜다:

```
┌─────────────────────────────────────────────────────────────┐
│                   디스플레이 광고 판매 채널                     │
├──────────────────────────┬──────────────────────────────────┤
│  보장형 채널 (GD/GFA)     │   비보장형 채널 (RTB/NGD)         │
│  - 사전 계약              │   - 실시간 경매                   │
│  - 고정 가격 (CPM)        │   - 가변 가격 (Auction)          │
│  - 노출 수 보장           │   - 노출 수 미보장                │
│  - 프리미엄 인벤토리      │   - 롱테일 인벤토리               │
└──────────────────────────┴──────────────────────────────────┘
```

---

## 2. GFA(보장형 광고)의 구조와 메커니즘

### 2.1 GFA 정의

**GFA(Guaranteed Fixed-price Advertising)** = **GD(Guaranteed Delivery)** 의 한국형 명칭으로, 광고주와 퍼블리셔 간 사전 계약을 통해 특정 기간 동안 특정 타겟 조건을 만족하는 노출 수를 고정 가격으로 보장하는 광고 방식이다.

> 예시: "7월 한 달간 서울 거주 20-35세 여성 대상 1,000,000회 노출을 CPM ₩5,000에 보장"

### 2.2 GFA 계약 구성 요소

```
GFA 계약 (GD Contract)
├── 타겟팅 조건 (Targeting Criteria)
│   ├── 인구통계 (성별, 연령, 지역)
│   ├── 관심사 / 행동 데이터
│   └── 디바이스 유형
├── 보장 지표 (Guaranteed Metric)
│   ├── 노출 수 (Impressions) - 가장 일반적
│   ├── 클릭 수 (Clicks, CPC 기반) - 덜 일반적
│   └── 노출 기간 (Delivery Period)
├── 가격 구조 (Pricing)
│   ├── CPM (Cost Per Mille) - 1,000회 노출당 비용
│   └── 고정 총 예산 (Fixed Total Budget)
└── 게재 조건 (Delivery Terms)
    ├── 광고 소재 형식 (Banner, Video, Native)
    ├── 게재 위치 (지면, 플레이스먼트)
    └── 게재 시간대
```

### 2.3 GFA의 운영 사이클

```
[계약 체결] → [인벤토리 예측] → [계약 배분] → [실시간 게재] → [페이싱 조정] → [성과 리포팅]
     ↑                                                                              ↓
     └──────────────────── 다음 캠페인 피드백 ─────────────────────────────────────┘
```

#### 단계별 설명

| 단계 | 내용 | 핵심 기술 |
|------|------|-----------|
| **계약 체결** | 광고주와 노출 수, 단가, 기간, 타겟 협의 | 가격 협상 알고리즘 |
| **인벤토리 예측** | 계약 기간 동안 가용한 타겟 유저 트래픽 예측 | Time-series 예측, ML 모델 |
| **계약 배분** | 여러 캠페인 간 인벤토리 최적 할당 | LP, 최적화 알고리즘 |
| **실시간 게재** | 유저 방문 시 조건에 맞는 광고 서빙 | 실시간 매칭 엔진 |
| **페이싱 조정** | 예산 소진 속도 조절, 균등 게재 보장 | Pacing 알고리즘 |
| **성과 리포팅** | CTR, CVR, ROAS 등 KPI 집계 | 통계 분석, 대시보드 |

---

## 3. RTB(비보장형) vs GFA(보장형) 비교

### 3.1 핵심 차이점 비교표

| 항목 | GFA (보장형) | RTB (비보장형) |
|------|-------------|----------------|
| **가격 결정** | 사전 고정 (CPM) | 실시간 경매 |
| **타겟팅 정밀도** | 코스한 타겟 묶음 구매 | 개인 수준 정밀 타겟팅 |
| **노출 보장** | 계약 수량 보장 | 미보장 (경매 낙찰 시에만) |
| **인벤토리 품질** | 프리미엄 (Front page 등) | 다양 (롱테일 포함) |
| **브랜드 안전성** | 높음 (사전 지면 지정) | 상대적으로 낮음 |
| **광고주 리스크** | 낮음 (노출 보장) | 높음 (경매 결과 불확실) |
| **퍼블리셔 리스크** | 높음 (미래 인벤토리 커밋) | 낮음 |
| **집행 속도** | 사전 계획 필요 | 즉시 집행 가능 |
| **최소 예산** | 높음 (통상 1M원 이상) | 낮음 |
| **최적화 타겟** | 브랜드 인지도 (상위 퍼널) | 직접 전환 (하위 퍼널) |

### 3.2 GFA 선택이 유리한 시나리오

1. **리스크 회피형 광고주**: 불확실한 경매 결과 대신 확정된 노출 수 선호
2. **브랜드 안전 중시**: 광고가 노출될 지면을 사전에 검증하고자 할 때
3. **장기 파트너십**: 퍼블리셔와의 관계에서 커스텀 포맷, 가격 협상 혜택
4. **대규모 캠페인**: 특정 날짜(블랙프라이데이, 시즌 이벤트)에 대량 노출 필요
5. **독점 지면 확보**: 홈페이지 타겟오버, 메인 배너 등 프리미엄 지면

---

## 4. GFA 인벤토리 예측 모델

### 4.1 예측 필요성

GFA는 미래 트래픽에 대해 노출을 보장하는 구조이므로, **인벤토리 예측의 정확도가 수익성과 직결**된다.

- 예측 과소 → 과매도(Oversell) → 게재 미달, 광고주 불만, 패널티
- 예측 과대 → 과소판매(Undersell) → 기회 손실, 수익 감소

### 4.2 인벤토리 예측 모델 유형

#### 4.2.1 통계 기반 시계열 모델

```
인벤토리(t) = Trend(t) + Seasonality(t) + Residual(t)
```

- **ARIMA**: 자기회귀 통합 이동평균 모델, 단기 예측에 적합
- **Prophet (Facebook)**: 트렌드 + 계절성 + 이벤트 효과 분리, 광고 플랫폼에 널리 사용
- **SARIMA**: 계절성을 명시적으로 포함한 ARIMA 확장

#### 4.2.2 기계학습 기반 모델

| 모델 | 특징 | 적합 상황 |
|------|------|-----------|
| **GBDT** (XGBoost, LightGBM) | 피처 중요도 해석 가능, 빠른 훈련 | 구조화된 특성이 많을 때 |
| **LSTM / GRU** | 장기 시계열 패턴 학습 | 복잡한 시계열 의존성 |
| **Transformer** | 어텐션 기반 장거리 의존성 | 대규모 데이터, 복잡한 패턴 |

#### 4.2.3 End-to-End 통합 모델 (NLS: Neural Lagrangian Selling)

Chen et al. (2023, KDD)이 제안한 모델로, **인벤토리 예측과 계약 배분을 동시에 최적화**한다.

```
전통적 방법:    [예측 모델] → [배분 최적화]  (순차적, 오차 누적)
NLS 방법:       [예측 + 배분 통합 학습]       (동시 최적화, 성능 향상)
```

**NLS 핵심 구조:**
- **Lagrangian Dual Layer**: 제약 조건(impression 보장)을 미분 가능한 형태로 학습에 통합
- **GCN Module**: 캠페인 간 인벤토리 경쟁 관계를 그래프로 모델링

### 4.3 인벤토리 불확실성 처리

현실의 트래픽은 불확실하므로, 확률적 프로그래밍(Stochastic Programming)을 사용한다.

```
max  E[Σ reach(campaign_i)]
s.t. P(delivery(campaign_j) ≥ contract_j) ≥ 1 - ε  ∀j
     (Chance Constraint: 각 계약의 게재 달성 확률이 1-ε 이상)
```

- **Chance-Constrained Programming**: 인벤토리 공급의 불확실성을 확률 제약으로 처리
- **Robust Optimization**: 최악의 인벤토리 시나리오에서도 계약을 이행할 수 있는 배분 계획 수립

---

## 5. CTR 예측 이론 및 모델

### 5.1 CTR 예측 문제 정의

GFA에서 CTR 예측은 **광고 성과 보고**, **인벤토리 가치 산정**, **최적화 지표 설정**에 사용된다.

**수식 정의:**

```
입력: 트리플 집합 {(u_i, x_i, y_i)}, i = 1, ..., N
  - u_i: i번째 인스턴스의 유저
  - x_i: i번째 인스턴스의 광고
  - y_i: 클릭 여부 (1 = 클릭, 0 = 미클릭)

목표: p = f(u_j, x_j) 예측 (j번째 인스턴스의 CTR)

손실 함수 (Cross Entropy):
L = -(1/N) Σ [y_i × log(p_i) + (1 - y_i) × log(1 - p_i)]
```

### 5.2 CTR 예측 입력 피처

| 피처 카테고리 | 세부 항목 | 예시 |
|---------------|-----------|------|
| **광고 피처** | 광고 소재 타입, 크기, 형식, 소재 ID | 배너 300x250, 비디오 15초 |
| **유저 피처** | 인구통계, 관심사, 과거 클릭 이력 | 30대 남성, 게임 관심사 |
| **컨텍스트 피처** | 방문 시각, 디바이스, 운영체제, 위치 | 오후 9시, 모바일, iOS |
| **쿼리 피처** | 검색 키워드, 탐색 의도 | "갤럭시 구매" 검색 후 |
| **퍼블리셔 피처** | 게재 지면, 페이지 카테고리 | 뉴스 섹션, IT 채널 |

### 5.3 CTR 예측 모델 발전사

#### 1세대: 통계 기반 선형 모델

**Logistic Regression (LR)**
```
Φ_LR(w, x) = w^T × x + b
p = σ(Φ_LR(w, x))    ← 시그모이드 활성화

장점: 빠른 훈련, 해석 가능, 대규모 데이터 효율적
단점: 고차 피처 상호작용 포착 불가
```

#### 2세대: 인수분해 기계 (Factorization Machine)

**FM (Factorization Machine)**
```
p_FM(w, x) = w_0 + Σ w_i × x_i + Σ_i Σ_j <v_i, v_j> × x_i × x_j

- <v_i, v_j>: 임베딩 벡터 내적으로 2차 피처 상호작용 학습
- Sparse 데이터에서도 효과적 (광고 데이터의 특성)
```

**FFM (Field-aware FM)**
```
p_FFM = w_0 + Σ w_i × x_i + Σ_i Σ_j <v_i,f_j, v_j,f_i> × x_i × x_j

- 피처 필드(Field)를 명시적으로 구분하여 상호작용 학습
- CTR 예측 대회(KDD Cup 2012)에서 우수한 성능 기록
```

#### 3세대: 딥러닝 기반 하이브리드 모델

**Wide & Deep (Google, 2016)**
```
Wide 파트:  Logistic Regression  →  암기(Memorization) 강화
Deep 파트:  Deep Neural Network  →  일반화(Generalization) 강화
최종 출력:  p = σ(w_wide^T [x, φ(x)] + w_deep^T a^(l) + b)
```

**DeepFM (Huawei, 2017)**
```
FM 파트:  저차 피처 상호작용 학습 (명시적)
Deep 파트: 고차 피처 상호작용 학습 (암시적)
특징: Wide & Deep과 달리 FM과 Deep이 임베딩 공유 → 더 효율적
```

**DCN (Deep & Cross Network, Google, 2017)**
```
Cross Network:  명시적 고차 피처 교차 학습
Deep Network:   암시적 비선형 패턴 학습
```

**DIN (Deep Interest Network, Alibaba, 2018)**
```
핵심 아이디어: 타겟 광고와 관련 있는 유저 행동에만 어텐션 집중
Attention(v_a, e_u) = Σ_i a(e_u_i, v_a) × e_u_i
```

### 5.4 CTR 모델 평가 지표

| 지표 | 수식 | 의미 |
|------|------|------|
| **AUC-ROC** | Area Under ROC Curve | 랭킹 능력. 0.5 = 랜덤, 1.0 = 완벽 |
| **Log Loss** | -Σ[y log(p) + (1-y)log(1-p)] / N | 예측 확률의 정확도 (낮을수록 좋음) |
| **RMSE** | √(Σ(y-p)² / N) | 예측값과 실제값의 평균 오차 |
| **RelaImpr** | (AUC_new - 0.5) / (AUC_base - 0.5) - 1 | 기준 모델 대비 상대적 향상률 |
| **Calibration** | predicted CTR / observed CTR | 예측 CTR이 실제 CTR과 얼마나 일치하는지 |

> **GFA 운영 시 중요**: Calibration이 낮으면 입찰 가격 산정이 잘못되어 과지출/과소지출 발생

---

## 6. CVR(전환율) 예측 및 Delayed Feedback

### 6.1 Delayed Feedback 문제

GFA에서 CVR을 학습할 때 **전환 시차(Delayed Feedback)** 문제가 발생한다.

```
광고 노출 → 클릭 → [수일/수주 후] → 구매 전환

문제: 전환이 기록되기 전에 학습 데이터로 사용하면
      "전환 없음"으로 잘못 라벨링됨 → 모델 편향
```

### 6.2 Delayed Feedback 해결 모델

**DFM (Delayed Feedback Model, Criteo KDD 2014)**
```
실제 전환률: λ(x) = P(conversion | impression, features x)
지연 분포:   d ~ Exponential(λ_d)  ← 전환 지연 시간 모델링

수정된 손실 함수에서 관측되지 않은 전환을 확률적으로 처리
```

**ES-DFM (Elapsed-Time Sampling, Alibaba AAAI 2021)**
```
핵심 아이디어: 경과 시간(elapsed time)을 샘플링 가중치로 사용
→ 최근 데이터일수록 전환 미관측 확률이 높으므로 가중치 조정
```

### 6.3 GFA에서의 CVR 활용

- **캠페인 성과 예측**: 광고주에게 예상 전환 수 제시
- **과금 방식 결정**: CPM vs CPC vs CPA 전환 시
- **최적화 알고리즘**: CVR × 전환 가치 = 광고 기대 수익

---

## 7. 예산 Pacing 알고리즘

### 7.1 Pacing의 필요성

GFA 계약은 캠페인 기간 동안 **균등하게 노출을 게재(smooth delivery)** 해야 한다.

```
나쁜 예: 첫날 전체 예산의 80% 소진 → 남은 기간 광고 미게재
좋은 예: 매일 균일하게 계약 노출의 1/30씩 게재
```

### 7.2 Pacing 알고리즘 수식

**기본 비율 조절 Pacing**
```
target_rate(t) = remaining_budget(t) / remaining_time(t)
current_rate = recent_delivery_rate

if current_rate > target_rate:
    throttle_probability = target_rate / current_rate  ← 확률적 필터링
else:
    increase_bid_probability = 1.0  ← 모든 요청 응답
```

**RCPacing (Risk-Constrained Pacing, Taobao)**
```
핵심 기법: Lagrangian Dual Multiplier를 사용한 확률적 쓰로틀링
       단조 매핑 함수(monotonic mapping)로 퍼센타일 공간에서 조정

목표: min Σ (delivery - contract)²
s.t. budget ≤ total_budget
     P(underdelivery) ≤ risk_threshold

성능: O(√T) dynamic regret 달성
```

**RL 기반 Pacing (최신 연구)**
```
State:    현재 게재율, 잔여 예산, 시간, 트래픽 분포
Action:   입찰 확률 조정 (0~1)
Reward:   계약 달성률 + 균등 게재 지수 - 과게재 패널티
```

### 7.3 Pacing 품질 지표

| 지표 | 수식 | 목표 |
|------|------|------|
| **게재 달성률** | delivered_impressions / contracted_impressions | ≥ 95% |
| **균등 게재 지수** | 1 - Std(daily_delivery) / Mean(daily_delivery) | ≥ 0.9 |
| **과게재율** | max(0, delivered - contracted) / contracted | ≤ 3% |
| **예산 소진율** | spent_budget / total_budget | ≈ 100% (±3%) |

---

## 8. 타겟팅 이론 및 오디언스 세그먼트

### 8.1 GFA 타겟팅 유형

#### 인구통계 타겟팅 (Demographic Targeting)
```
세그먼트 기준: 성별, 연령대, 지역, 디바이스
특징: 예측 가능성 높음, 인벤토리 규모 산정 용이
한계: 개인 수준 관심사 반영 어려움
```

#### 관심사/문맥 타겟팅 (Contextual Targeting)
```
세그먼트 기준: 방문 컨텐츠 카테고리, 검색어
특징: 쿠키 없이도 작동, 개인정보 보호법 대응
한계: 구매 의도 추정 정확도 낮음
```

#### 행동 타겟팅 (Behavioral Targeting)
```
세그먼트 기준: 과거 클릭/전환 이력, 사이트 방문 패턴
특징: 구매 의도 추정 정확도 높음
한계: 쿠키 의존도 높음, 시간에 따른 관심사 변화
```

### 8.2 GFA 타겟팅의 통계적 과제

```
문제: 광고주가 수개월 전 계약 시 정의한 타겟 세그먼트가
      실제 캠페인 기간에 충분히 존재하지 않을 수 있음

해결책:
1. 확률적 인벤토리 예측으로 세그먼트 규모 추정
2. 유사 오디언스(Lookalike Audience)로 타겟 확장
3. 과거 시계열 데이터로 계절성 반영
```

### 8.3 오디언스 클러스터링

대규모 인벤토리 관리를 위해 유저 세그먼트를 클러스터링으로 집약한다.

```python
# 개념적 오디언스 클러스터링 프로세스
segments = user_features_matrix  # N_users × D_features

# K-means 또는 계층적 클러스터링
clusters = cluster(segments, k=K)

# 클러스터 단위로 GFA 계약 배분 → 계산 복잡도 감소
# N_users 규모 → K_clusters 규모로 축소 (K << N)
```

---

## 9. 광고 성과 측정 및 KPI 체계

### 9.1 GFA 핵심 KPI

```
┌─────────────────────────────────────────────────────────┐
│                   GFA 성과 KPI 체계                       │
├──────────────────┬──────────────────────────────────────┤
│  노출/게재 지표  │  CTR/전환 지표         │  비용 지표  │
├──────────────────┼────────────────────────┼─────────────┤
│  - Impressions   │  - CTR (Click Rate)    │  - CPM      │
│  - Reach         │  - CVR (Conv. Rate)    │  - CPC      │
│  - Frequency     │  - VTR (View Rate)     │  - CPA      │
│  - 달성률        │  - ROAS                │  - 소진율   │
└──────────────────┴────────────────────────┴─────────────┘
```

### 9.2 핵심 지표 수식

**CTR (Click-Through Rate)**
```
CTR = Clicks / Impressions × 100%
```

**CVR (Conversion Rate)**
```
CVR = Conversions / Clicks × 100%
```

**ROAS (Return On Ad Spend)**
```
ROAS = Revenue / Ad Spend × 100%
```

**Reach & Frequency**
```
Reach     = 순 유니크 유저 수 (중복 제거)
Frequency = Total Impressions / Reach  ← 평균 노출 빈도
```

**CPM (Cost Per Mille)**
```
CPM = (Total Cost / Total Impressions) × 1,000
```

**효과적 CPM (eCPM)**
```
eCPM = (Total Revenue / Total Impressions) × 1,000
     = CPM × CTR × (CPC / CPM)
```

### 9.3 GFA 광고주 목표별 최적 KPI

| 광고 목표 | 주요 KPI | 보조 KPI |
|-----------|----------|----------|
| **브랜드 인지도** | Reach, 순 유저 수, Impression | Frequency, VTR |
| **브랜드 고려도** | CTR, 랜딩페이지 체류시간 | Frequency ≥ 3 |
| **구매 전환** | CVR, CPA, ROAS | CTR |
| **리타겟팅** | CVR, ROAS | Frequency |

### 9.4 광고 효과 측정의 인과 추론 과제

단순 CTR/CVR 집계는 **광고 노출의 인과 효과**를 측정하지 못할 수 있다.

```
문제: 광고를 보지 않아도 구매했을 사람들이 포함됨
     → 광고의 실제 기여도(Incrementality) 과대추정

해결책:
- A/B 테스트 (실험군 vs 대조군)
- Ghost Ads (대조군에게 가상 광고 노출 추적)
- Difference-in-Differences (이중차분법)
```

---

## 10. GFA 가격 책정(Pricing) 모델

### 10.1 GFA 가격 결정 요소

```
CPM 가격 결정 함수:
price = f(targeting_specificity, inventory_scarcity, 
          historical_performance, campaign_period, 
          advertiser_tier, competitive_demand)
```

### 10.2 통계 기반 가격 책정

**인벤토리 공급 기반 가격 (Najafi-Asadolahi & Fridgeirsdottir)**
```
publisher 최적화 문제:
max  Σ revenue(contract_j) 
s.t. Σ impressions_j ≤ available_inventory
     P(delivery_j ≥ commitment_j) ≥ confidence_level

→ 각 캠페인의 그림자 가격(shadow price)으로 적정 CPM 추정
```

**동적 가격 책정 (Guaranteed + Spot 혼합)**
```
reserved_price(t) = spot_market_value(t) × premium_factor
premium_factor = f(brand_safety, targeting_quality, inventory_exclusivity)
```

### 10.3 GFA vs RTB 균형 최적화

퍼블리셔 입장에서 GFA와 RTB 채널 간 인벤토리 배분 최적화:

```
max  GFA_revenue + RTB_revenue
s.t. GFA_impressions + RTB_impressions ≤ total_inventory
     GFA_delivery ≥ contracted_amount (보장 이행)

해석: GFA 계약의 option value vs RTB의 동적 경매 가치 간 트레이드오프
```

---

## 11. GFA 계약 배분 최적화 알고리즘

### 11.1 기본 배분 문제 (LP 정식화)

```
결정 변수: x_ij = 세그먼트 i를 캠페인 j에 배분하는 비율

최대화:   Σ_j utility_j(Σ_i x_ij × supply_i)

제약 조건:
  Σ_j x_ij ≤ 1          ∀i  (인벤토리 초과 배분 금지)
  Σ_i x_ij × supply_i ≥ demand_j  ∀j  (계약 노출 수 보장)
  0 ≤ x_ij ≤ 1          ∀i,j  (비율 범위)
```

### 11.2 대규모 문제: Dantzig-Wolfe 분해

수백만 규모의 세그먼트-캠페인 배분 문제를 분해하여 효율적으로 풀기 위해 Dantzig-Wolfe Decomposition을 사용한다.

```
Master Problem: 전체 배분 균형 최적화
Sub-problems:  각 캠페인별 최적 배분 독립 계산

→ 병렬 처리 가능, 대규모 광고 플랫폼에 실용적
```

### 11.3 GD 계약의 공정 배분 (Fairness)

```
Gini 기반 공정성 지표:
Gini(delivery) = 2 × area between Lorenz curve and equality line

목표: 모든 유저 세그먼트에 균등하게 광고 노출
     (특정 세그먼트에 광고가 집중되는 것 방지)

Stochastic Programming 정식화:
  max   E[Σ_j Gini_spread_j]
  s.t.  P(Σ delivery_j ≥ contract_j) ≥ 1 - ε  (기회제약)
```

### 11.4 Adaptive Allocation (적응형 배분)

캠페인 진행 중 실시간으로 배분을 조정하는 Dynamic Allocation:

```
주기적 재배분 루프:
1. 현재 게재 현황 수집
2. 잔여 인벤토리 재예측
3. 언더딜리버리 위험 캠페인 식별
4. 배분 비율 재조정
5. 실시간 서빙 엔진에 업데이트 배포
```

---

## 12. AI 운영 의사결정 프레임워크

### 12.1 GFA 운영 AI가 처리해야 할 의사결정 트리

```
GFA 캠페인 수신
      │
      ▼
[인벤토리 충분?]──No──→ [대안 제안: 기간 연장, 타겟 완화, 물량 축소]
      │ Yes
      ▼
[가격 적정?]──No──→ [적정 CPM 산출 → 역제안]
      │ Yes
      ▼
[계약 배분 최적화] → LP/NLS 알고리즘 실행
      │
      ▼
[실시간 게재 모니터링]
      │
      ├──[언더딜리버리 감지]──→ [Pacing 상향 조정]
      │
      ├──[오버딜리버리 감지]──→ [Pacing 하향 조정]
      │
      └──[CTR 이상 감지]──→ [소재 최적화 제안]
```

### 12.2 자동 알람 임계값 (Threshold)

| 상황 | 임계값 | AI 액션 |
|------|--------|---------|
| 게재 달성률 저조 | < 80% (캠페인 중반) | Pacing 상향 알림 |
| 예산 소진 과속 | > 120% 목표 소진율 | Pacing 하향 조정 |
| CTR 급락 | 기준 CTR의 50% 미만 | 소재 변경 제안 |
| 인벤토리 고갈 위험 | 예측 인벤토리 < 계약의 90% | 관리자 에스컬레이션 |
| 캠페인 만료 임박 | 잔여 기간 < 3일 & 달성률 < 90% | 긴급 부스팅 모드 |

### 12.3 A/B 테스트 설계 원칙

GFA 운영에서 광고 효과를 측정하는 A/B 테스트:

```
실험 설계:
- 처리군 (Treatment): 광고 노출
- 대조군 (Control):   광고 미노출 (또는 Ghost Ads)

통계 검정:
- 최소 유의수준: α = 0.05
- 최소 검정력:   β = 0.80
- 최소 효과 크기: Δ = 상대적 CTR 향상 5% 이상

표본 크기 계산:
n = 2 × (z_α/2 + z_β)² × p(1-p) / Δ²
  여기서 p = 기준 CTR
```

---

## 13. 핵심 수식 및 통계 모델 참조

### 13.1 Logistic Regression (LR) - CTR 베이스라인

```
p = 1 / (1 + exp(-(w^T × x + b)))

학습 방법: FTRL (Follow-The-Regularized-Leader) - 온라인 학습
정규화:    L1 + L2 혼합 정규화
```

### 13.2 Factorization Machine (FM) - 피처 상호작용

```
y_FM(x) = w_0 + Σ_i w_i × x_i + Σ_i Σ_{j>i} <v_i, v_j> × x_i × x_j

파라미터: w_0 ∈ ℝ, w ∈ ℝ^n, V ∈ ℝ^{n×k}
계산복잡도: O(kn) (나이브 구현 대비 O(kn²) → 효율화 가능)
```

### 13.3 DeepFM - FM + DNN 통합

```
y_DeepFM = sigmoid(y_FM + y_DNN)

FM 파트:  저차 상호작용 (1차 + 2차)
DNN 파트: embedding → hidden layers → output
공유 임베딩: FM과 DNN이 동일한 임베딩 레이어 사용
```

### 13.4 Pacing 최적화 (Lagrangian Dual)

```
원 문제:
min   Σ_t (delivery_t - target_t)²
s.t.  Σ_t cost_t ≤ B (예산 제약)

Lagrangian:
L(x, λ) = Σ_t (delivery_t - target_t)² + λ(Σ_t cost_t - B)

최적 입찰 확률:
p*(t) = argmin_p L(p, λ*)
→ 이진 탐색으로 λ* 찾기 (경계 조건 만족)
```

### 13.5 Chance-Constrained 인벤토리 배분

```
확률적 인벤토리 s_i ~ N(μ_i, σ_i²) 가정 시:

P(Σ_i x_ij × s_i ≥ d_j) ≥ 1 - ε
→ 정규분포 가정 시 SOC(Second-Order Cone) 제약으로 변환:

Σ_i x_ij × μ_i - Φ^{-1}(1-ε) × √(Σ_i x_ij² × σ_i²) ≥ d_j

SOCP (Second-Order Cone Program)으로 효율적 풀이 가능
```

### 13.6 Reach & Frequency 모델

```
Reach(n회 이상 노출) = 1 - (1 - p_exposure)^n  (독립 가정 시)

Binomial 분포 기반 Frequency 분포:
P(freq = k | impressions = N, users = M) = C(N,k) × (1/M)^k × (1-1/M)^(N-k)

→ 실제 사용: Beta-Binomial 모델 (사용자 접속 빈도 이질성 반영)
```

---

## 14. 참고 논문 목록

### 핵심 논문 (Core References)

| 논문 | 저자 | 학회/년도 | 주제 | 링크 |
|------|------|-----------|------|------|
| Online Display Advertising Markets: A Literature Review | Choi, Mela, Balseiro, Leary | Information Systems Research, 2020 | 디스플레이 광고 생태계 종합 리뷰 | [PDF](https://hanachoi.github.io/research-papers/choi_et_al_review_display_advertising.pdf) |
| Large-Scale Personalized Delivery for GD Advertising | Fang et al. | IEEE/KDD, 2019 | 개인화 GD 게재 + 실시간 Pacing | [IEEE](https://ieeexplore.ieee.org/document/8970726/) |
| End-to-End Inventory Prediction and Contract Allocation | Chen et al. | ACM KDD, 2023 | NLS 통합 예측·배분 모델 | [ACM](https://dl.acm.org/doi/10.1145/3580305.3599332) |
| Click-Through Rate Prediction in Online Advertising | Yang & Zhai | IP&M, 2022 | CTR 예측 모델 종합 리뷰 | [arXiv](https://arxiv.org/pdf/2202.10462) |
| Robust Ad Delivery Plan for GD Advertising | Zhang et al. | IEEE, 2014 | 강건 게재 계획, 확률적 최적화 | [IEEE](https://ieeexplore.ieee.org/document/6932818) |
| Pricing Guaranteed Contracts in Display Advertising | Turner | CIKM, 2010 | GFA 가격 책정 알고리즘 | [ResearchGate](https://www.researchgate.net/publication/221614607) |

### 모델별 원본 논문

| 모델명 | 논문 | 출처 |
|--------|------|------|
| **Logistic Regression (FTRL)** | Ad Click Prediction: A View from the Trenches | McMahan et al., KDD 2013 |
| **FM** | Factorization Machines | Rendle, ICDM 2010 |
| **FFM** | Field-aware Factorization Machines | Juan et al., RecSys 2016 |
| **Wide & Deep** | Wide & Deep Learning for Recommender Systems | Cheng et al., DLRS 2016 |
| **DeepFM** | DeepFM: A Factorization-Machine based Neural Network for CTR Prediction | Guo et al., IJCAI 2017 |
| **DCN** | Deep & Cross Network for Ad Click Predictions | Wang et al., ADKDD 2017 |
| **DIN** | Deep Interest Network for CTR Prediction | Zhou et al., KDD 2018 |
| **DFM** | Modeling Delayed Feedback in Display Advertising | Chapelle, KDD 2014 |
| **ES-DFM** | Capturing Delayed Feedback in CVR Prediction | Gu et al., AAAI 2021 |
| **RCPacing** | Robust Risk-Constrained Pacing | Taobao Team |
| **NLS** | End-to-End Inventory Prediction and Contract Allocation | Chen et al., KDD 2023 |

---

## 부록 A: 용어 사전 (Glossary)

| 용어 | 영문 | 설명 |
|------|------|------|
| **GFA** | Guaranteed Fixed-price Advertising | 네이버의 보장형 디스플레이 광고 상품 |
| **GD** | Guaranteed Delivery | 노출 수 보장형 광고 (GFA와 동의어) |
| **RTB** | Real-Time Bidding | 실시간 경매 기반 광고 |
| **CPM** | Cost Per Mille | 1,000회 노출당 비용 |
| **CPC** | Cost Per Click | 클릭당 비용 |
| **CPA** | Cost Per Action | 전환(액션)당 비용 |
| **CTR** | Click-Through Rate | 클릭률 = 클릭 수 / 노출 수 |
| **CVR** | Conversion Rate | 전환율 = 전환 수 / 클릭 수 |
| **ROAS** | Return On Ad Spend | 광고비 대비 매출 |
| **Pacing** | - | 예산 소진 속도 조절 알고리즘 |
| **Impression** | - | 광고 1회 노출 |
| **Reach** | - | 순 유니크 유저 수 |
| **Frequency** | - | 유저당 평균 노출 빈도 |
| **DMP** | Data Management Platform | 오디언스 데이터 관리 플랫폼 |
| **DSP** | Demand Side Platform | 광고주용 구매 플랫폼 |
| **SSP** | Supply Side Platform | 퍼블리셔용 판매 플랫폼 |
| **FM** | Factorization Machine | 피처 상호작용 학습 모델 |
| **AUC** | Area Under Curve | 모델 예측 성능 지표 |
| **Logloss** | Log Loss | 이진 분류 손실 함수 |
| **Throttle** | - | 광고 요청에 응답하는 확률 조절 |
| **Shadow Price** | - | LP 최적화에서 제약 조건의 한계 가치 |
| **Lagrangian Dual** | - | 제약 최적화 문제의 이중 문제 접근법 |

---

## 부록 B: 네이버 GFA 운영 시나리오별 AI 대응 가이드

### 시나리오 1: 캠페인 언더딜리버리 감지

```
조건: 캠페인 진행률 50% 시점에서 게재 달성률 < 70%

AI 분석:
1. 타겟 세그먼트 인벤토리 재예측 (현재 트래픽 추세 반영)
2. 경쟁 캠페인과의 인벤토리 충돌 확인
3. 게재 시간대 분포 분석 (특정 시간대 집중 여부)

AI 액션:
- Pacing 계수 상향 조정 (+30~50%)
- 게재 시간대 확대 (야간 포함)
- 타겟 완화 제안 (광고주 승인 필요)
- 잔여 기간 달성 가능성 재산출 → 리포트
```

### 시나리오 2: CTR 이상 감지 (급락)

```
조건: 최근 3일 CTR이 기준 CTR 대비 40% 이하

AI 분석:
1. 소재별 CTR 분해 (소재 A vs B vs C)
2. 시간대별 CTR 분석 (특정 시간대 집중 하락 여부)
3. 세그먼트별 CTR 분석 (특정 타겟에서만 하락 여부)
4. Frequency 분석 (광고 피로도 증가 여부)

AI 액션:
- 저성과 소재 자동 교체 제안
- Frequency Cap 조정 (3회 → 2회)
- 광고주에게 소재 갱신 알림 발송
```

### 시나리오 3: 인벤토리 고갈 위험

```
조건: 예측 인벤토리 < 계약 잔여 노출 수 × 1.1

AI 분석:
1. 고갈 예상 시점 계산 (days_to_exhaustion)
2. 유사 세그먼트로의 확장 가능 인벤토리 추정
3. 경쟁 캠페인 배분 재조정 여지 검토

AI 액션:
- 광고주에게 인벤토리 상황 즉시 알림
- 대안 타겟 세그먼트 제안 (예상 도달 차이 명시)
- 기간 연장 시 추가 인벤토리 확보 가능 여부 제시
```

---

*문서 버전: v1.0 | 작성일: 2026-03 | 출처: 학술 논문 종합 분석*  
*본 문서는 GFA 운영 AI 학습용으로 작성되었으며, 주기적으로 업데이트 예정*
