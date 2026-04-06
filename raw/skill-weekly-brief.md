---
name: weekly-brief
description: 주간 경영 브리핑. 매출/채널/광고/CS 지표를 종합 요약하고 노션 페이지로 생성 가능. "주간 보고", "위클리", "주간 브리핑" 등의 요청 시 활성화.
user-invocable: true
allowed-tools: Bash, Read, Write
---

# 주간 경영 브리핑

## 1단계: 데이터 수집 (병렬 실행)

다음 MCP 도구를 병렬로 호출하세요:

- `get_sales_overview` — 매출 개요
- `get_sales_daily` — 일별 매출 (주간 추이)
- `get_sales_by_channel` — 채널별 매출
- `get_sales_by_product` — 상품별 매출
- `get_ad_overview` — 광고 성과 개요
- `get_gfa_performance` — GFA 성과
- `get_sa_performance` — SA 성과
- `get_cs_overview` — CS 현황
- `get_claims_summary` — 클레임 요약
- `get_margin_analysis` — 마진 분석
- `get_inventory_current` — 재고 현황
- `get_trend_analysis` — 트렌드

## 2단계: 주간 리포트 작성

### 리포트 형식

```
📋 ToBe Networks 주간 경영 브리핑
기간: YYYY-MM-DD ~ YYYY-MM-DD

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

■ 1. 매출 성과
- 주간 총매출: ₩XXX (전주 대비 +X%)
- 일평균 매출: ₩XXX
- 최고 매출일: X월 X일 (₩XXX)

채널별 매출:
| 채널 | 매출 | 비중 | 전주대비 |
|------|------|------|---------|
| ...  | ...  | ...  | ...     |

상품 TOP 5:
| 순위 | 상품명 | 매출 | 판매량 |
|------|--------|------|--------|
| ...  | ...    | ...  | ...    |

■ 2. 광고 성과
- 주간 광고비: ₩XXX
- 평균 ROAS: X.XX
- GFA 성과: [요약]
- SA 성과: [요약]

■ 3. CS/클레임
- 주간 문의: X건 (답변율 X%)
- 클레임: X건 (전주 대비 +/-X%)
- 주요 클레임 유형: [목록]

■ 4. 재고/수익성
- 품절 임박: X개 상품
- 평균 마진율: X%
- 문제 상품: X개

■ 5. 트렌드
- 급상승 키워드: [목록]
- 시장 동향: [요약]

■ 6. 이번 주 핵심 이슈
1. [이슈 1]
2. [이슈 2]
3. [이슈 3]

■ 7. 다음 주 액션 아이템
1. [액션 1] — 담당: [부서/담당자]
2. [액션 2] — 담당: [부서/담당자]
3. [액션 3] — 담당: [부서/담당자]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 3단계: 노션 저장 (선택)

사용자가 노션 저장을 요청하면:
1. `notion-search`로 주간 브리핑 페이지/DB 검색
2. `notion-create-pages`로 새 페이지 생성
3. 리포트 내용을 노션 페이지에 작성
4. 생성된 페이지 URL 공유
