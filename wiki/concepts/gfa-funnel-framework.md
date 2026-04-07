---
title: GFA 퍼널 프레임워크
tags: [gfa, funnel, advertising, stdc]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# GFA 퍼널 프레임워크

[[keychron]] GFA(디스플레이 광고) 운영에서 캠페인을 퍼널 단계별로 분류하고 건강도를 측정하는 체계.

## STDC 모델

Google의 **See-Think-Do-Care** 프레임워크를 기반으로 한다.

| 단계 | 목적 | 핵심 KPI |
|------|------|----------|
| **See** | 브랜드 인지 확대 | CPM, Reach, Impression |
| **Think** | 관심·고려 유도 | CTR, 페이지뷰, 체류시간 |
| **Do** | 전환 (구매) | ROAS, CPA, 전환수 |
| **Care** | 기존 고객 재구매 | LTV, 재구매율 |

## TOF / MOF / BOF 분류 체계

GFA 캠페인은 다음 3단계 + 보조 유형으로 분류된다.

- **TOF (Top of Funnel)**: 브랜드 인지, 신규 유저 유입. See 단계에 대응.
- **MOF (Middle of Funnel)**: 관심 유도, 장바구니 추가. Think 단계에 대응.
- **BOF (Bottom of Funnel)**: 전환 최적화, 리타게팅. Do 단계에 대응.
- **ADVoost**: 기존 고객 대상 부스트 캠페인.
- **리타게팅**: 사이트 방문자 재노출.

### 4-시그널 자동분류

캠페인을 퍼널에 자동 배정할 때 4가지 시그널에 가중치를 둔다.

| 시그널 | 가중치 | 설명 |
|--------|--------|------|
| 캠페인명 키워드 | 40% | 이름에 포함된 TOF/MOF/BOF 등 키워드 |
| 전환 유형 | 25% | 설정된 전환 목표 (인지/트래픽/구매) |
| 성과 지표 패턴 | 20% | CPM, CTR, ROAS 등 실적 패턴 |
| CTR/CPM/CVR 임계값 | 15% | 수치 기반 자동 판별 |

## Binet & Field 60/40 규칙

Les Binet & Peter Field의 연구에 따르면 최적의 광고 예산 배분은:

- **60%** → 브랜드 빌딩 (TOF, 장기 성장)
- **40%** → 퍼포먼스 (BOF, 단기 전환)

GFA 운영에서는 이 비율을 기준으로 퍼널 밸런스를 모니터링한다. TOF 비중이 지나치게 낮으면 장기 파이프라인이 고갈되고, BOF에만 집중하면 [[ad-decay-saturation|광고 포화]]가 빨리 온다.

## 5축 100점 건강도 점수

GFA OPS AI가 캠페인별로 산출하는 효율 스코어:

| 축 | 측정 대상 | 점수 범위 |
|----|-----------|-----------|
| adj_ROAS | 보정 ROAS (퍼널 가중) | 0~100 |
| 볼륨 | 지출 규모 적정성 | 0~100 |
| 추세 | 최근 7일 성과 방향 | 0~100 |
| 페이싱 | 예산 소진 속도 | 0~100 |
| 포화도 | [[ad-decay-saturation|Adstock 포화]] 수준 | 0~100 |

종합 점수에 퍼널 밸런스를 가산하여 최종 운영 판단에 활용한다.

## 관련 개념

- [[messy-middle]] — 소비자 탐색-평가 루프
- [[attribution-modeling]] — 퍼널별 기여도 배분
- [[ad-decay-saturation]] — 광고 이월효과와 포화
- [[sales-data-pipeline]] — 전환 매출 데이터 흐름
