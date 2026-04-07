# 세션 기록: 공휴일/쇼핑이벤트 캘린더 마커 통합

**날짜**: 2026-02-24
**커밋**: `3ff6a11` feat: 공휴일/쇼핑이벤트 캘린더 마커 + 인사이트 통합

---

## 작업 요약

광고 성과 분석 시 공휴일(설날, 추석 등)이나 쇼핑 이벤트(블프, 11.11 등)가 성과에 큰 영향을 미치지만, 대시보드에 이런 정보가 없어 성과 변동 원인을 파악하기 어려웠음. 일별 차트에 이벤트 마커(세로 점선 + 라벨)를 표시하고, 인사이트에 이벤트 문맥을 추가.

---

## 구현 내용

### 1. 백엔드: 이벤트 데이터 모듈

**`app/utils/calendar_events.py`** (신규)
- Python 딕셔너리로 이벤트 데이터 내장 (외부 라이브러리 없음)
- **고정 공휴일**: 신정, 삼일절, 어린이날, 현충일, 광복절, 개천절, 한글날, 크리스마스
- **음력 공휴일 (2024-2030 하드코딩)**: 설날(3일), 추석(3일), 부처님오신날
- **쇼핑 이벤트**: 밸런타인데이, 화이트데이, 어버이날, 광군제(11.11), 블랙프라이데이, 사이버먼데이
- 헬퍼: `get_events_for_range(start, end)` → `[{date, name, type, key}, ...]`

### 2. 백엔드: API 엔드포인트

**`app/routes/calendar.py`** (신규)
- `GET /api/calendar/events?start=...&end=...` → 기간 내 이벤트 목록 반환
- 기존 `get_date_params()` 활용

### 3. 프론트엔드: 유틸 모듈

**`app/static/js/calendar-events.js`** (신규)
- `fetchEvents()`: API 호출 + URL 파라미터 기반 캐시
- `buildAnnotations(events, rawDates)`: Chart.js annotation 설정 생성
  - 공휴일: 빨간 점선 + 빨간 라벨 (`rgba(186,5,23,0.55)`)
  - 쇼핑이벤트: 파란 점선 + 파란 라벨 (`rgba(1,118,211,0.55)`)
  - 라벨은 -90도 회전되어 차트 상단에 세로 표시
- `getEventInsights(events)`: 인사이트 배열 반환

### 4. 통합

| 파일 | 변경 내용 |
|------|-----------|
| `app/static/index.html` | chartjs-plugin-annotation CDN 추가 |
| `app/static/js/tabs/unified.js` | 인사이트 + 메인차트 마커 연동 |
| `app/static/js/tabs/overview.js` | 광고비/ROAS, CPC/CVR/CTR 차트 마커 |
| `app/static/js/tabs/gfa.js` | 일별추이, 광고비vsROAS 차트 마커 |
| `app/routes/__init__.py` | calendar Blueprint 등록 |

---

## 색상 설계

대시보드 기존 색상 팔레트(`--sf-error`, `--sf-info`)에 맞춰 반투명 톤 적용:
- 공휴일 점선: `rgba(186,5,23,0.22)`, 라벨: `rgba(186,5,23,0.55)`
- 쇼핑이벤트 점선: `rgba(1,118,211,0.22)`, 라벨: `rgba(1,118,211,0.55)`

---

## 디자인 반복 과정

1. **초기**: 텍스트 라벨 직접 표시
2. **사용자 피드백**: "글자 말고 작은 태그 + 마우스오버 툴팁" → H/S 태그 + hover 구현
3. **사용자 피드백**: "영상 편집기 타임라인 마커처럼 노란색으로" → ▼ 마커 구현
4. **최종**: "글자 나오던거로 원복" → 텍스트 라벨 + 부드러운 색상 톤

---

## 인사이트 예시

- 📅 "이 기간에 공휴일 포함: 설날 (3일). 공휴일에는 검색량·전환율이 변동될 수 있으므로 성과 해석 시 참고하세요."
- 🛒 "이 기간에 쇼핑 이벤트 포함: 밸런타인데이. 경쟁 입찰 심화 가능성이 있으므로 CPC 추이와 예산 소진을 모니터링하세요."

---

## 파일 변경 목록

| 파일 | 상태 |
|------|------|
| `app/utils/__init__.py` | 신규 |
| `app/utils/calendar_events.py` | 신규 |
| `app/routes/calendar.py` | 신규 |
| `app/routes/__init__.py` | 수정 (+2줄) |
| `app/static/index.html` | 수정 (+1줄 CDN) |
| `app/static/js/calendar-events.js` | 신규 |
| `app/static/js/tabs/unified.js` | 수정 |
| `app/static/js/tabs/overview.js` | 수정 |
| `app/static/js/tabs/gfa.js` | 수정 |
