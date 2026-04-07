# QA 검증 문서: 투비AI 웹챗 전 도메인 기능 검증

> 작성일: 2026-03-01
> 검증 대상: 투비AI 웹챗 (http://localhost:3000/) + Data API (https://tbe.kr/data/)
> 목적: 5개 도메인(매출/재고/원가/CS/광고)의 AI 응답 품질 + 데이터 정확성 체계적 검증

---

## 검증 방법론

1. **웹챗 질문** → AI 응답 수집
2. **API 직접 호출** → 실제 데이터 확인
3. **응답 vs 실데이터 대조** → 정확성 판정
4. **시스템 프롬프트 반영 여부** → 워크플로우/규칙 준수 확인
5. **발견 이슈 기록** → 수정 → 재검증

---

## 1. 재고 도메인 (Inventory) — 1/1 PASS

### TC-INV-01: EX75 추가발주 필요 여부
- **질문**: "EX75 추가발주 필요해?"
- **기대 행동**: trend_analysis 호출 → PO 확인 → 현재고 확인 → 종합 판단
- **웹챗 응답**: trend_analysis + smart_reorder 호출. 발주 K260123 존재, option_stocks 포함 확인
- **API 검증 결과**: A6 옵션만 재고 542개/월판매 334개/49일분. 나머지 3옵션 품절.
- **판정**: ✅ PASS (option_stocks 자동 포함으로 옵션별 분석 가능)

### TC-INV-02~04: 추가 검증 대기
- 재고 위험 상품, 특정 상품 재고 상세, 옵션별 분석 — 추후 검증 예정

---

## 2. 매출 도메인 (Sales) — 4/4 PASS

### TC-SALES-01: 이번 달 매출 요약
- **질문**: "이번 달 매출 요약해줘"
- **기대 행동**: sales_overview → 총매출/주문수/AOV
- **웹챗 응답**: AI가 Sales Overview 호출, 총매출/주문수/AOV 정확히 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS

### TC-SALES-02: 상품별 매출
- **질문**: "이번 달 가장 많이 팔린 상품 Top 5"
- **기대 행동**: sales_by_product → 실거래가 기반 랭킹
- **웹챗 응답**: AI가 Sales By Product 호출, Top 5 상품 랭킹 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS

### TC-SALES-03: 실거래가 조회
- **질문**: "Q1 HE 실거래가 얼마야?"
- **기대 행동**: product_real_price(keyword=Q1 HE) → 평균/최저/최고
- **웹챗 응답**: AI가 Product Real Price 호출, 평균/최저/최고 가격 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS

### TC-SALES-04: 채널별 매출
- **질문**: "채널별 매출 비교해줘"
- **기대 행동**: sales_by_channel → 전체 채널 비중
- **웹챗 응답**: AI가 Sales By Channel 호출, 채널별 매출 비중 정확히 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS

---

## 3. CS 도메인 (Customer Service) — 4/4 PASS

### TC-CS-01: 미답변 고객 문의
- **질문**: "미답변 고객 문의 확인"
- **기대 행동**: cs_overview → 미답변 목록 + 대기일수
- **웹챗 응답**: AI가 Cs Overview + Cs Inquiries Unanswered 호출, 미답변 수 및 대기일수 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS

### TC-CS-02: 반품/클레임 현황
- **질문**: "이번 달 반품/클레임 현황 알려줘"
- **기대 행동**: claims_rate → 12% 이하 정상 기준 판단
- **웹챗 응답**: AI가 Cs Claims Summary + Cs Claims Rate 호출, 클레임 유형별 건수 + 비율 반환
- **API 검증**: 데이터 일치 확인
- **판정**: ✅ PASS
- **⚠️ 발견 버그**: Claims Rate 500 오류 (빈 날짜 범위) → IFNULL 처리 + Pydantic 모델 기본값 수정

### TC-CS-03: AS 현황
- **질문**: "AS 처리 현황 어때?"
- **기대 행동**: as_status + as_diagnosis → 완료율/고장패턴
- **웹챗 응답**: AI가 Cs As Status 호출, 완료율/진행/대기 수치 반환 (API 데이터 정확히 일치)
- **API 검증**: total 1,029건, completed 835건, completion_rate 81.1% 확인
- **판정**: ✅ PASS
- **⚠️ 발견 버그**: AS Diagnosis/Products 500 오류 — `a.as_seq` → `a.seq` JOIN 수정

### TC-CS-04: QnA 미답변
- **질문**: "QnA 미답변 목록 보여줘"
- **기대 행동**: qna_unanswered 호출
- **웹챗 응답**: AI가 Cs Qna Unanswered 호출, 38건 반환 (API 데이터 정확히 일치)
- **API 검증**: 38건 일치 확인
- **판정**: ✅ PASS

---

## 4. 광고 도메인 (Advertising) — 1/1 PASS

### TC-AD-01: 광고 ROAS 분석
- **질문**: "광고 ROAS 분석해줘"
- **기대 행동**: ad_overview(SA+GFA) → ROAS 수치 + 판단 기준
- **웹챗 응답**: AI가 SA Performance + GFA Performance + GFA Funnel 호출.
  - SA: spend 19,152,655원, ROAS 193.1%, CTR 0.99%
  - GFA: spend 23,696,474원, ROAS 29.3%
  - Binet & Field 60/40 퍼널 분석 포함
- **API 검증**: SA/GFA 수치 API 데이터 정확히 일치
- **판정**: ✅ PASS
- **⚠️ 발견 버그 (3건)**:
  1. SA_PERFORMANCE: `impressions`→`imp_cnt`, `clicks`→`clk_cnt` 컬럼명 오류 수정
  2. WASTE_KEYWORDS: 동일 컬럼명 오류 수정
  3. Advertising Overview: None.get() 오류 → `(sa or {}).get()` 패턴 수정

### TC-AD-02~04: 추가 검증 대기
- 낭비 키워드, 비즈머니 잔액, SA 일별 추이 — 추후 검증 예정 (API 엔드포인트는 200 OK 확인)

---

## 5. 원가 도메인 (Cost) — 1/1 PASS

### TC-COST-01: 상품 원가/마진 조회
- **질문**: "K10 PRO SE 원가랑 마진 알려줘"
- **기대 행동**: product_cost + margin_analysis → 원가(KRW/USD/VAT) + 마진율
- **웹챗 응답**: AI가 4개 도구 호출 (Cost Product, Cost Margin, Sales Product Real Price, Cost Products Batch)
  - 원가: 79,423원 (USD 52.46, 환율 1,377원, VAT ×1.1)
  - 마진율: 34.4% (평균 실거래가 121,079원)
- **API 검증**: 원가 79,423원, 마진율 34.4% API 데이터 정확히 일치
- **판정**: ✅ PASS

### TC-COST-02: 마진 분석
- 추후 검증 예정

---

## 6. 크로스 도메인

### TC-CROSS-01: 복합 질문
- **질문**: "이번 달 가장 매출 높은 상품의 재고 상황은?"
- **기대 행동**: sales 도메인 → inventory 도메인 순차 호출
- **웹챗 응답**: (대기)
- **판정**: ⏳ 대기

---

## 발견된 이슈 로그

| # | 도메인 | 심각도 | 설명 | 상태 |
|---|--------|--------|------|------|
| 1 | 전체 | 🔴 HIGH | aiomysql LIMIT %s → 문자열 래핑 오류 (`LIMIT '10'`). f-string `LIMIT {int(limit)}`으로 수정 | ✅ 수정완료 |
| 2 | CS | 🔴 HIGH | crm_as PK는 `seq`인데 SQL에서 `a.as_seq`로 JOIN → 1054 오류 | ✅ 수정완료 |
| 3 | 광고 | 🔴 HIGH | naver_ad_campaign_daily 컬럼명 `imp_cnt`/`clk_cnt`인데 `impressions`/`clicks`로 사용 | ✅ 수정완료 |
| 4 | 광고 | 🔴 HIGH | naver_ad_keyword_daily 동일 컬럼명 오류 | ✅ 수정완료 |
| 5 | 광고 | 🟡 MED | Ad Overview: SA/GFA 데이터 없으면 None.get() → AttributeError | ✅ 수정완료 |
| 6 | CS | 🟡 MED | ClaimRate: 빈 날짜 범위에서 SUM=None → Pydantic int 검증 실패 | ✅ 수정완료 |
| 7 | 배포 | 🟡 MED | SSH 배포 시 ed25519 키 인증 실패 → claude.pem으로 해결 | ✅ 해결 |

---

## 수정 이력

| 시각 | 파일 | 변경 내용 | 커밋 |
|------|------|-----------|------|
| 03-01 | sql/cs_queries.py | AS JOIN: `a.as_seq` → `a.seq` (2곳), CLAIMS_RATE IFNULL 추가, LIMIT %s 제거 | 6c8f998 |
| 03-01 | sql/ad_queries.py | SA_PERFORMANCE/WASTE_KEYWORDS 컬럼명 수정, LIMIT %s 제거 | 6c8f998 |
| 03-01 | sql/sales_queries.py | LIMIT %s 제거 | 6c8f998 |
| 03-01 | sql/cost_queries.py | LIMIT %s 제거 | 6c8f998 |
| 03-01 | routers/cs.py | LIMIT f-string 패턴 적용 (5곳) | 6c8f998 |
| 03-01 | routers/advertising.py | LIMIT f-string + overview None 처리 | 6c8f998 |
| 03-01 | routers/sales.py | LIMIT f-string 패턴 적용 | 6c8f998 |
| 03-01 | routers/cost.py | LIMIT f-string 패턴 적용 | 6c8f998 |
| 03-01 | models/cs.py | ClaimRate 기본값(0/None) 추가 | 6c8f998 |

---

## 배포 상태

| 환경 | 커밋 | 상태 | 검증 |
|------|------|------|------|
| 로컬 (localhost:8100) | 6c8f998 | ✅ 적용됨 | 전 엔드포인트 200 OK |
| 프로덕션 (tbe.kr/data) | 6c8f998 | ✅ 배포완료 | 6개 주요 엔드포인트 스모크테스트 200 OK |

---

## QA 종합 결과

| 도메인 | 테스트 수 | 통과 | 실패 | 결과 |
|--------|----------|------|------|------|
| 재고 (Inventory) | 1 | 1 | 0 | ✅ PASS |
| 매출 (Sales) | 4 | 4 | 0 | ✅ PASS |
| CS (고객서비스) | 4 | 4 | 0 | ✅ PASS |
| 광고 (Advertising) | 1 | 1 | 0 | ✅ PASS |
| 원가 (Cost) | 1 | 1 | 0 | ✅ PASS |
| **총계** | **11** | **11** | **0** | **✅ ALL PASS** |

**발견 버그**: 7건 → 전량 수정 완료 (커밋 6c8f998)

---

## 시스템 프롬프트 반영 체크리스트

| 도메인 | 워크플로우 | 판단기준치 | 균등분할금지 | 실거래가규칙 |
|--------|-----------|-----------|-------------|-------------|
| Sales | ✅ 반영 | ✅ 반영 | N/A | ✅ 반영 |
| Inventory | ✅ 반영 | ✅ 반영 | ✅ 반영 | N/A |
| Cost | ✅ 반영 | ✅ 반영 | N/A | ✅ 반영 |
| CS | ✅ 반영 | ✅ 반영 | N/A | N/A |
| Advertising | ✅ 반영 | ✅ 반영 | N/A | N/A |
