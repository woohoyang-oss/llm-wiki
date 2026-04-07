# Session 2026-02-21: GFA 대시보드 + 사이드바 재구성

## 작업 요약

이번 세션에서는 3가지 주요 작업을 완료했습니다:
1. **GFA (성과형 디스플레이 광고) 대시보드 신규 구축**
2. **좌측 사이드바 트리 네비게이션 적용**
3. **예산 기반 비용 예측 기능 추가**

---

## 1. GFA 대시보드 구축

### 배경
- Naver GFA는 API가 없어 CSV 수동 다운로드 방식
- gfa.naver.com에서 분석단위별(캠페인/광고그룹/소재) 리포트 다운로드
- CSV 인코딩: UTF-8 with BOM (`utf-8-sig`), 날짜: `YYYY.MM.DD.` (trailing dot)

### DB 테이블 생성 (3개)
```sql
-- naver_ad_gfa_campaign_daily: 캠페인 일별
-- naver_ad_gfa_adgroup_daily: 광고그룹 일별 (도달/도달비용/노출빈도 추가)
-- naver_ad_gfa_creative_daily: 소재 일별
-- 모두 UNIQUE KEY (account, entity_id, stat_date) for upsert
```

### 백엔드: `app/routes/gfa.py` (신규, ~657줄)
- **POST `/api/gfa/upload`**: CSV/ZIP 업로드, 자동 분석단위 감지
- **GET `/api/gfa/overview`**: 종합 대시보드 (일별합산/캠페인요약/광고그룹TOP15/소재TOP15/비효율소재)
- **GET `/api/gfa/campaigns`**: 캠페인 목록
- **GET `/api/gfa/adgroups`**: 광고그룹 목록 (도달/노출빈도 포함)
- **GET `/api/gfa/creatives`**: 소재 목록
- **GET `/api/gfa/data-status`**: 업로드 현황

### 프론트엔드: `app/static/js/tabs/gfa.js` (신규, ~434줄)
- KPI 카드 8종 (광고비/노출/클릭/전환/매출/ROAS/구매ROAS/CPA)
- 차트 4종 (일별추이/광고비vsCTR·CVR/캠페인파이/캠페인일별스택)
- 테이블 4종 (캠페인/광고그룹/소재/비효율소재)
- CSV/ZIP 업로드 UI

### 업로드된 데이터
- 캠페인: 63건, 광고그룹: 95건, 소재: 633건
- 기간: 2026-02-14 ~ 2026-02-20 (7일)

---

## 2. 좌측 사이드바 트리 네비게이션

### 변경 내용
- **기존**: 상단 탭 바 (가로 스크롤)
- **변경**: 좌측 220px 사이드바 + 트리구조 메뉴

### 메뉴 구조
```
SA 검색광고
  ├── 개요
  ├── 일별비교
  ├── 캠페인
  ├── 광고그룹
  ├── 키워드
  ├── 트렌드
  ├── 통합분석
  ├── 스코어링
  ├── 액션
  └── 로그

GFA 디스플레이
  └── 개요

설정
  ├── 전략설정
  └── 텔레그램
```

### 수정된 파일
- `app/static/index.html`: `<div class="app-layout">` flex 레이아웃 + `<aside class="sidebar">`
- `app/static/styles/main.css`: 사이드바 CSS (nav-group, nav-item, collapsed 상태)
- `app/static/js/app.js`: `toggleNavGroup()`, `toggleSidebar()` 함수 + window 브릿지

---

## 3. 예산 기반 비용 예측

### 신규: `app/routes/budget.py`
- 월 예산 대비 실제 지출 추적
- 남은 기간 예측 비용 계산
- overview 탭에 예측 차트 연동

### 수정: `app/routes/overview.py`
- 비용 예측 데이터 overview API에 통합 (+135줄)

### 수정: `app/static/js/tabs/overview.js`
- 예측 차트 렌더링 추가 (+67줄)

---

## 4. 기타 변경사항

### `app/routes/strategy_config.py`
- 전략 설정 기능 고도화 (+69줄 변경)

### `app/routes/diagnosis.py`
- 진단 로직 소폭 수정 (+21줄)

### `app/scoring.py`
- 스코어링 로직 개선 (+28줄 변경)

### `requirements.txt`
- 의존성 1개 추가

---

## 파일 변경 요약

### 신규 파일 (3개)
| 파일 | 설명 |
|------|------|
| `app/routes/budget.py` | 예산 기반 비용 예측 API |
| `app/routes/gfa.py` | GFA 대시보드 전체 API |
| `app/static/js/tabs/gfa.js` | GFA 프론트엔드 탭 모듈 |

### 수정 파일 (11개)
| 파일 | 변경량 |
|------|--------|
| `app/routes/__init__.py` | +4줄 (GFA, budget 블루프린트 등록) |
| `app/routes/diagnosis.py` | +21줄 |
| `app/routes/overview.py` | +135줄 (예측 통합) |
| `app/routes/strategy_config.py` | +69줄 |
| `app/scoring.py` | +28줄 |
| `app/static/index.html` | +278줄 (사이드바 + GFA 탭 HTML) |
| `app/static/js/app.js` | +36줄 (GFA import + 사이드바 함수) |
| `app/static/js/tabs/overview.js` | +67줄 (예측 차트) |
| `app/static/js/tabs/strategy.js` | +23줄 |
| `app/static/styles/main.css` | +78줄 (사이드바 스타일) |
| `requirements.txt` | +1줄 |

**총 변경: +581줄 추가, -159줄 삭제 (순 +422줄)**

---

## 미완료 / 다음 세션 작업

### GFA 서브탭 확장 (계획됨, 미실행)
- GFA를 SA와 유사한 구조로 서브탭 분리: 개요/캠페인/광고그룹/소재
- 각 레벨별 전용 차트+테이블 (SA의 campaigns.js/adgroups.js 패턴 참조)
- GFA 고유 분석: 도달/빈도 분석, 소재 성과 랭킹, 전환 퍼널
- 매일 업로드 현황 관리 페이지
- 향후 API 연동 대비 아키텍처 (데이터소스 추상화)

### GFA 최대 기간 데이터
- gfa.naver.com에서 3년치 데이터 다운로드 필요
- 브라우저 보안 제한으로 자동화 불가 → 수동 다운로드 후 업로드

---

## 인프라 정보
- **EC2**: 13.125.219.231:8087
- **DB**: host=222.122.42.221, port=3306, db=TBNWS_ADMIN
- **배포**: SCP → `/home/ec2-user/naver-ad/`
- **서버 재시작**: `pkill -f run.py && nohup python3 run.py > api.log 2>&1 &`
