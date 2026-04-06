---
name: daily-report
description: 일일 경영 대시보드. 매출, 광고, CS 핵심 지표를 종합 리포트로 제공. "오늘 현황", "일일 리포트", "매출 요약" 등의 요청 시 자동 활성화.
user-invocable: true
allowed-tools: Bash, Read, Write
---

# 일일 경영 리포트 생성

아래 순서로 데이터를 수집하고 종합 리포트를 작성하세요.

## 1단계: 데이터 수집 (병렬 실행)

다음 MCP 도구를 병렬로 호출하세요:

- `get_sales_overview` — 매출 개요
- `get_sales_daily` — 일별 매출 추이
- `get_sales_by_channel` — 채널별 매출
- `get_ad_overview` — 광고 성과 개요
- `get_cs_overview` — CS 현황 개요
- `get_claims_summary` — 클레임 요약

## 2단계: 리포트 작성

수집된 데이터를 아래 형식으로 정리하세요:

### 리포트 형식

```
📊 ToBe Networks 일일 리포트 (YYYY-MM-DD)

■ 매출 현황
- 오늘 매출: ₩XXX (전일 대비 +X%)
- 채널별: [스마트스토어/쿠팡/자사몰 등 채널별 매출]
- 주요 상품: [매출 상위 3개 상품]

■ 광고 성과
- 총 광고비: ₩XXX
- ROAS: X.XX
- 주요 이슈: [비효율 캠페인 또는 특이사항]

■ CS 현황
- 미답변 문의: X건
- 클레임: X건 (처리율 X%)
- 긴급 사항: [즉시 대응 필요 건]

■ 요약 및 액션 아이템
1. [가장 중요한 액션]
2. [두 번째 액션]
3. [세 번째 액션]
```

## 3단계: 전달

- 리포트를 텍스트로 출력
- 사용자가 노션 저장을 요청하면 Notion MCP를 활용하여 페이지 생성
