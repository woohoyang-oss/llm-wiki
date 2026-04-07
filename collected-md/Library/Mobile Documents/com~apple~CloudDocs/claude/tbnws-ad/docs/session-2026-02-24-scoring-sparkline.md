# 세션 기록: 스코어링 90일 추이 스파크라인

**날짜**: 2026-02-24
**커밋**: `61d637b` feat: 스코어링 90일 추이 스파크라인 + 트렌드 차트 높이 수정

---

## 작업 요약

스코어링 페이지의 캠페인/광고그룹 테이블에 90일 주간 롤링 점수 추이를 보여주는 인라인 스파크라인 미니 차트 추가. 등급이 현재 낮더라도 최근 상승 추세인지 한눈에 파악 가능.

---

## 구현 내용

### 1. 백엔드: `/api/scoring/history` 확장

**`app/routes/scoring_routes.py`**
- 기존 캠페인별 히스토리 + **광고그룹별 히스토리** 추가
- 쿼리: `naver_ad_adgroup_daily`에서 90일 데이터 한 번에 조회
- `adgroup_id`, `adgroup_name` 기준 GROUP BY 추가
- 공통 `_calc_weekly()` 내부 함수로 리팩터링
- 응답: `{"ok": true, "history": {...}, "adgroup_history": {...}, "weeks": [...]}`

### 2. 프론트엔드: 스파크라인 렌더링

**`app/static/js/tabs/scoring.js`**
- `_drawSparkline(canvas, scores)` 함수 신규 추가
  - **HiDPI 대응**: `devicePixelRatio` 스케일링으로 레티나 디스플레이 선명 렌더링
  - **추세 판단**: 전반부 평균 vs 후반부 평균 비교
  - **색상**: 상승(초록 `rgb(46,132,74)`), 하락(빨강 `rgb(186,5,23)`), 보합(회색 `rgb(147,147,147)`)
  - **그라데이션 채움**: 위 30% → 아래 5% 투명도
  - **선**: 2px 두께, `lineJoin: round`, `lineCap: round`
  - **마지막 점**: 2.5px 원형 마커
- 캠페인 테이블: `<td>` 내 `<canvas width="80" height="24">` (80px)
- 광고그룹 테이블: 별도 "90일 추이" 컬럼, `<canvas width="120" height="24">` (120px)
- history API 호출 시 `?days=90` 강제 (현재 기간 설정 무관하게 항상 90일)

### 3. 트렌드 차트 높이 수정

**`app/static/js/tabs/trends.js`**
- 수평 바 차트가 세로로 너무 길었던 문제 수정
- 동적 높이: `Math.max(150, Math.min(300, groupNames.length * 32 + 40))`
- `barThickness: 18` 추가

### 4. 캐시 버스팅

- `index.html`: `<script src="js/app.js?v=20260224e">`
- `app.js`: `import ... from './tabs/scoring.js?v=20260224e'`

---

## 디버깅 과정

### 문제 1: 스파크라인이 안 보임 (초기)
- **원인**: `${color}18` (8자리 hex) → alpha 0x18 = 9.4% 투명도 (거의 투명)
- **해결**: hex → RGB 직접 사용, `rgba(r,g,b,0.30)` 그라데이션

### 문제 2: 특정 기간에서만 스파크라인 표시
- **원인**: `apiFetch('/api/scoring/history')`가 현재 기간(`days=7`)을 전달 → 7일치만 조회 → 주간 버킷 1개 → 스파크라인 최소 2포인트 미달
- **해결**: `apiFetch('/api/scoring/history?days=90')` — 항상 90일 고정

### 문제 3: 광고그룹 스파크라인 너비 부족
- **원인**: 종합 점수와 같은 `<td>`에 캔버스 삽입 → 테이블 자동 레이아웃이 너비 압축
- **해결**: 별도 "90일 추이" 컬럼 분리 (`<th>` + `<td>`)

---

## 파일 변경 목록

| 파일 | 상태 | 설명 |
|------|------|------|
| `app/routes/scoring_routes.py` | 수정 | adgroup_history 추가, _calc_weekly 리팩터링 |
| `app/static/js/tabs/scoring.js` | 수정 | _drawSparkline 함수, 캠페인/광고그룹 스파크라인 |
| `app/static/js/tabs/trends.js` | 수정 | 바 차트 동적 높이 계산 |
| `app/static/index.html` | 수정 | 90일 추이 컬럼 헤더 추가, 캐시 버스팅 |
| `app/static/js/app.js` | 수정 | scoring.js 캐시 버스팅 |

---

## 기술 노트

### Canvas 스파크라인 vs Chart.js
- Chart.js는 오버헤드가 크고 테이블 인라인에 적합하지 않음
- Canvas 2D API 직접 사용 → 경량, 빠른 렌더링
- 각 캔버스 독립적으로 DPR 스케일링 적용

### apiFetch와 기간 파라미터
- `apiFetch(endpoint)`는 `getParams()`로 현재 UI의 `account`, `days`(또는 `start`/`end`)를 자동 추가
- history API처럼 고정 기간이 필요한 경우 URL에 직접 `?days=90` 포함 필요
- `apiFetch`는 `endpoint.includes('?') ? '&' : '?'`로 separator 처리

### EC2 venv 경로
- `/home/ec2-user/naver-ad/venv/` 없음 (시스템 python3 사용)
- Flask 재시작: `sudo kill $(pgrep -f 'python3 run.py'); cd /home/ec2-user/naver-ad && nohup python3 run.py &>/tmp/flask.log & disown`
