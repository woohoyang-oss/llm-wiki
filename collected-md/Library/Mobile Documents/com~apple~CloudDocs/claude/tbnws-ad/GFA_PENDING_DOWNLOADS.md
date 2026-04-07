# GFA 미완료 다운로드 — Claude 실행용

> 생성일: 2026-02-21
> 계정: 키크론 (26343)
> GFA 탭 ID: gfa.naver.com 탭 사용 (tabs_context_mcp로 확인)

## 🤖 Claude 자동 실행 지침

이 파일을 읽으면 아래 절차대로 광고그룹 데이터를 다운로드하고 DB에 업로드하세요.

### 핵심 주의사항
1. **navigate 도구 사용 불가** — gfa.naver.com은 보안 제한으로 `navigate` 사용 불가. `javascript_tool`에서 `window.location.href`로 이동
2. **확인 버튼 셀렉터** — `.ant-modal-footer` 내 확인 버튼이 아닌, 좌표 클릭 사용 (약 x=948, y=274)
3. **동시 요청 제한** — 한 번에 10개 이하만 요청 (초과시 "생성 실패")
4. **각 청크 간 15초 이상 대기** — 큐 처리 여유 확보

### 자동화 순서
1. GFA 페이지에서 `javascript_tool`로 `window.location.href` 이동 (adUnit=AD_GROUP)
2. 8초 대기 (페이지 로드)
3. "다운로드 요청" 버튼 좌표 클릭 (약 x=1393, y=560)
4. 2초 대기 (모달 렌더링)
5. "확인" 버튼 좌표 클릭 (약 x=948, y=274)
6. 15초 대기 후 다음 청크
7. 10개 요청 후 다운로드 목록에서 확인 → 다운로드 → 업로드 → 삭제 → 나머지 요청

### 업로드 명령
```bash
# ZIP 추출 후 각 CSV 업로드
curl -s -u tobe:tobe1245 -F "file=@result.csv;filename=result.csv" -F "account=keychron" "http://tbe.kr:8087/api/gfa/upload"
```

### 검증
```bash
curl -s -u tobe:tobe1245 "http://tbe.kr:8087/api/gfa/data-status?account=keychron" | python3 -m json.tool
# 기대: adgroup min_date=2024-03-04 이하, cnt 대폭 증가
```

---

## 현황 요약

| 분석 단위 | DB 데이터 범위 | 상태 |
|-----------|---------------|------|
| 캠페인 | 2024-03-04 ~ 2026-02-21 | ✅ 완료 (5,612건) |
| 소재 | 2024-03-04 ~ 2026-02-21 | ✅ 완료 (28,702건) |
| 광고그룹 | 2024-10-18 ~ 2026-02-20 | ❌ 미완료 (이전 세션 데이터만 1,319건) |

> 참고: 2024-03-04 이전 기간은 키크론 계정 활동 데이터 없음

## 광고그룹 — 재다운로드 필요 (19청크)

JS 자동화에서 확인 버튼 셀렉터 버그로 요청이 실패함.
- 버그: `.ant-modal-footer` 내 확인 버튼을 찾았으나, 다운로드 확인 모달은 해당 구조를 사용하지 않음
- 해결: 좌표 기반 클릭 또는 올바른 셀렉터 사용 필요

### 수동 다운로드 방법

1. GFA 성과 리포트 접속: https://gfa.naver.com/adAccount/accounts/26343/report/performance
2. 분석 단위: **광고 그룹** 선택
3. 기간 단위: **일** 선택
4. 각 청크별 날짜 입력 후 **확인** → **다운로드 요청** → 모달에서 **확인**
5. 다운로드 요청 목록에서 파일 다운로드
6. 업로드: `curl -s -u tobe:tobe1245 -F "file=@result.csv" -F "account=keychron" http://tbe.kr:8087/api/gfa/upload`

### 청크 목록 (60일 단위, 총 19청크)

| # | 시작일 | 종료일 | 비고 |
|---|--------|--------|------|
| 1 | 2025-12-24 | 2026-02-21 | 데이터 있을 것 |
| 2 | 2025-10-25 | 2025-12-23 | 데이터 있을 것 |
| 3 | 2025-08-26 | 2025-10-24 | 데이터 있을 것 |
| 4 | 2025-06-27 | 2025-08-25 | 데이터 있을 것 |
| 5 | 2025-04-28 | 2025-06-26 | 데이터 있을 것 |
| 6 | 2025-02-27 | 2025-04-27 | 데이터 있을 것 |
| 7 | 2024-12-29 | 2025-02-26 | 데이터 있을 것 |
| 8 | 2024-10-30 | 2024-12-28 | 데이터 있을 것 |
| 9 | 2024-08-31 | 2024-10-29 | 데이터 있을 것 |
| 10 | 2024-07-02 | 2024-08-30 | 데이터 있을 것 |
| 11 | 2024-05-03 | 2024-07-01 | 데이터 있을 것 |
| 12 | 2024-03-04 | 2024-05-02 | 데이터 있을 것 |
| 13 | 2024-01-04 | 2024-03-03 | 데이터 없을 수 있음 |
| 14 | 2023-11-05 | 2024-01-03 | 데이터 없을 수 있음 |
| 15 | 2023-09-06 | 2023-11-04 | 데이터 없을 수 있음 |
| 16 | 2023-07-08 | 2023-09-05 | 데이터 없을 수 있음 |
| 17 | 2023-05-09 | 2023-07-07 | 데이터 없을 수 있음 |
| 18 | 2023-03-10 | 2023-05-08 | 데이터 없을 수 있음 |
| 19 | 2023-02-21 | 2023-03-09 | 데이터 없을 수 있음 |

### JS 자동화 (수정된 패턴)

각 청크를 브라우저 콘솔에서 실행:

```javascript
// Step 1: 페이지 이동
window.location.href = '/adAccount/accounts/26343/report/performance?' +
  new URLSearchParams({
    adUnit: 'AD_GROUP',
    dateRange: '시작일,종료일',  // 예: '2025-12-24,2026-02-21'
    dateUnit: 'DAY',
    placeUnit: 'TOTAL',
    audience: 'TOTAL'
  }).toString();

// Step 2: 페이지 로드 후 (7초 대기), 다운로드 요청 버튼 클릭
const d = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent.includes('다운로드 요청') && !b.textContent.includes('목록'));
if(d) d.click();

// Step 3: 모달 뜬 후 (2초 대기), 확인 클릭
// 주의: .ant-modal-footer 셀렉터 사용 금지! 좌표 클릭 또는 아래 셀렉터 사용
const c = document.querySelector('.ant-modal-confirm-btns .ant-btn-primary')
  || Array.from(document.querySelectorAll('.ant-btn-primary.ant-btn-lg'))
       .find(b => b.textContent.trim() === '확인');
if(c) c.click();
```

### 업로드 후 검증

```bash
curl -s -u tobe:tobe1245 "http://tbe.kr:8087/api/gfa/data-status?account=keychron" | python3 -m json.tool
```

기대 결과: adgroup의 min_date가 2024-03-04 이하, cnt가 대폭 증가
