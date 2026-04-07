# 세션 기록: 2026-02-25 — KDI 시장 지수 + 실시간분석

## 작업 요약

### 1. GFA 캠페인 관리 모바일 반응형 (v2.7.0)
- 모바일(768px↓)에서 12컬럼 → 6컬럼 축소 (ROAS추이, 노출, CTR, CPM, CVR, CPA 숨김)
- `table-layout: auto` 전환, 폰트/패딩 축소, 서브테이블 동일 적용

### 2. KDI 키보드 시장 수요 지수 (v2.7.0)
- **목적**: 12개 키워드 검색 트렌드를 하나의 시장 대표 지표로 합성
- **알고리즘**: 3-Tier 가중 z-score 합성
  - T1(50%): 기계식키보드, 무선키보드, 저소음키보드, 무선기계식키보드 (핵심 시장)
  - T2(35%): 커스텀, 래피드트리거, 자석축, 인체공학, 텐키리스 (확장 시장)
  - T3(15%): 키보드 (대분류 맥락)
  - 제외: 키크론(자사 브랜드), 무접점(브랜드와 너무 유사)
- **처리**: z-score 표준화 → Tier 내 균등평균 → 7일 MA → 가중 합성 → 0~100 정규화
- **표시**: KPI 카드 3장 + 듀얼 라인 차트(시장 vs 브랜드) + 시장점유 방향
- **기존 트렌드 차트에도 KDI/브랜드 지수 오버레이 추가**
- 파일: `app/routes/trends.py` (API), `app/static/js/tabs/trends.js` (차트)

### 3. 스마트스토어 실시간분석 페이지 (v2.8.0)
- **메뉴 위치**: 스마트스토어 > 마케팅분석 다음
- **BizData Stats API 5개 엔드포인트 활용**:
  - `realtime/daily` — 오늘 시간대별 결제/방문 + 상품 TOP
  - `marketing/all/detail` — 채널×디바이스 상세 (42개 채널, 3개 디바이스)
  - `marketing/all/daily` — 일별 채널 추이
  - `marketing/search/keyword` — 검색 키워드 TOP 10
  - `marketing/website/daily` — 웹사이트 유입 일별
- **4개 섹션**: 실시간 현황, 유입 채널, 검색 키워드, 일별 추이
- 파일: `app/routes/realtime.py`, `app/static/js/tabs/ss-realtime.js`

### 4. 실시간 데이터 DB 자동 수집 (v2.8.0)
- **크론**: 매시 01분 `scripts/smartstore_realtime_cron.py` 실행
- **DB 테이블 4개 자동 생성**:
  - `smartstore_realtime_hourly` — 시간대별 결제·방문 (매시 갱신)
  - `smartstore_realtime_products` — 일별 상품 TOP 10 (매시 갱신)
  - `smartstore_channel_detail` — 채널×디바이스 상세 성과 (일 1회)
  - `smartstore_keyword_daily` — 검색 키워드 성과 (일 1회)
- **백필**: `--backfill 7`로 최근 7일 740건 저장 완료
- UPSERT 방식으로 중복 안전

### 5. 버그 수정
- `auth.py`: `/data/` 경로 정적 파일 인증 예외 추가 (changelog.json 접근 차단 해결)
- `update-history.js`: loading display 방식 수정

## 변경 파일

| 파일 | 작업 |
|------|------|
| `app/routes/trends.py` | KDI market-index API 추가 |
| `app/routes/realtime.py` | **신규** — 실시간분석 API 4개 |
| `app/routes/__init__.py` | realtime Blueprint 등록 |
| `app/auth.py` | /data/ 경로 인증 예외 |
| `app/static/index.html` | 실시간분석 메뉴+탭, KDI 영역 |
| `app/static/js/tabs/trends.js` | KDI 차트 렌더링 + 오버레이 |
| `app/static/js/tabs/ss-realtime.js` | **신규** — 실시간분석 탭 |
| `app/static/js/tabs/update-history.js` | loading 표시 수정 |
| `app/static/js/app.js` | 실시간분석 탭 로더 등록 |
| `app/static/styles/main.css` | KDI + 실시간 + GFA 모바일 CSS |
| `scripts/smartstore_realtime_cron.py` | **신규** — 실시간 데이터 수집 크론 |

## 커밋

| 버전 | 커밋 | 내용 |
|------|------|------|
| v2.7.0 | `c29db6f` | KDI 시장 수요 지수 + GFA 모바일 반응형 |
| v2.8.0 | `903d313` | 실시간분석 페이지 + BizData 자동 수집 |

## 서버 크론탭 현황

```
0  * * * *  naver_ad_cron.py          # 매시 SA 광고 데이터
1  * * * *  smartstore_realtime_cron.py # 매시 01분 실시간 데이터
2  * * * *  smartstore_cron.py         # 매시 02분 스마트스토어
0  1 * * *  naver_trend_cron.py        # 새벽 1시 트렌드
0  1 * * *  telegram_ad_report.py      # 새벽 1시 텔레그램 리포트
0  4 * * *  ebs_snapshot.sh            # 새벽 4시 EBS 스냅샷
0  6 * * *  cleanup.sh                 # 새벽 6시 정리
```
