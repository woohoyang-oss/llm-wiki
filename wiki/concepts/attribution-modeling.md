---
title: Attribution Modeling (기여도 모델링)
tags: [advertising, attribution, analytics]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# Attribution Modeling (기여도 모델링)

전환(구매)에 이르기까지 여러 광고 접점이 기여한 정도를 배분하는 방법론. GFA 퍼널 운영에서 각 캠페인의 실제 가치를 판단하는 핵심 도구이다.

## 주요 모델 유형

### Last-Click (마지막 클릭)

- 전환 직전 마지막 접점에 100% 기여 부여
- 가장 단순하고 널리 사용됨
- **한계**: BOF 리타게팅에 과도한 크레딧, TOF/MOF 과소평가

### First-Click (첫 클릭)

- 최초 접점에 100% 기여 부여
- 신규 유입 채널 평가에 유용
- **한계**: 전환 직전 영향력 무시

### Linear (선형)

- 모든 접점에 동일 비율 배분
- 편향 없는 평가가 장점
- **한계**: 실제 영향력 차이를 반영하지 못함

### Time-Decay (시간 감쇠)

- 전환에 가까울수록 높은 기여율
- 반감기 기반 지수 감소
- BOF 중심이지만 Last-Click보다 공정

### Position-Based (위치 기반, U자형)

- 첫 접점 40% + 마지막 접점 40% + 나머지 20% 균등 배분
- 신규 유입과 전환 모두 중시
- 중간 접점 과소평가 가능성

## GFA 맥락에서의 적용

### TOF 어시스트 기여

[[gfa-funnel-framework|GFA 퍼널 프레임워크]]에서 TOF 캠페인은 직접 전환이 적지만, [[messy-middle|Messy Middle]]에 소비자를 진입시키는 역할을 한다.

Last-Click 기준 ROAS만 보면 TOF 캠페인은 비효율적으로 보이지만, 어시스트 전환을 고려하면 실제 기여도가 훨씬 높다.

### adj_ROAS (보정 ROAS)

GFA OPS AI에서는 퍼널별 기여를 반영한 보정 ROAS를 사용한다:

- **TOF**: Last-Click ROAS에 어시스트 가중치 적용 (상향 보정)
- **MOF**: 장바구니 추가 기여 반영
- **BOF**: 리타게팅 할인율 적용 (하향 보정)

이를 통해 [[gfa-funnel-framework|5축 건강도 점수]]의 adj_ROAS 축이 산출된다.

### 실무 권장사항

| 상황 | 권장 모델 | 이유 |
|------|-----------|------|
| 일상 리포팅 | Last-Click | 단순, 플랫폼 기본값 |
| 퍼널 밸런스 점검 | Position-Based | TOF/BOF 모두 평가 |
| 예산 재배분 | Time-Decay | 최근 영향력 반영 |
| 신규 캠페인 평가 | First-Click | 유입 기여 측정 |

## ROAS 할인(Discount) 개념

TOF에서 높은 어시스트 기여가 있더라도, BOF 리타게팅의 직접 전환 ROAS를 그대로 쓰면 과대평가된다. 이를 보정하기 위해:

1. BOF 리타게팅 ROAS에서 TOF 어시스트분을 차감
2. TOF ROAS에 어시스트 기여를 가산
3. 전체 채널 ROAS와 합산 ROAS 간 차이를 정규화

이 과정이 GFA OPS AI의 [[gfa-funnel-framework|adj_ROAS]] 산출 로직의 핵심이다.

## 관련 개념

- [[gfa-funnel-framework]] — 퍼널 분류 및 건강도 점수
- [[messy-middle]] — 탐색-평가 반복 루프
- [[ad-decay-saturation]] — 광고 포화와 어트리뷰션 왜곡
- [[sales-data-pipeline]] — 전환 매출 데이터 소스
