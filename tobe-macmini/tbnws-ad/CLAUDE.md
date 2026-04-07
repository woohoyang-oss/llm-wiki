# TBNWS 네이버 광고 분석 대시보드 — 세션 기록

## 프로젝트 개요
Flask + Vanilla JS 기반 네이버 광고(SA/GFA) + 스마트스토어 통합 분석 대시보드.
URL: `http://tbe.kr:8087/`

---

## 인프라 / 배포

| 항목 | 값 |
|------|------|
| 서버 | AWS EC2 `13.125.219.231` |
| SSH | `SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@13.125.219.231` |
| 배포 경로 | `/home/ec2-user/naver-ad/` |
| 포트 | 8087 (Flask) |
| URL | `http://tbe.kr:8087/` |
| DB (메인) | MySQL `222.122.42.221:3306` / `TBNWS_ADMIN` / user: `tbnws22` |
| DB (실시간) | MySQL `222.122.42.221:3306` / `TBNWS_AD` / user: `tbnws22` |
| Git | `origin/main` |
| Python | 시스템 python3 (venv 없음) |
| 배포 방법 | `scp`로 변경 파일 복사 → Flask가 static 파일 자동 리로드 (Python 변경 시 수동 재시작: `bash /tmp/restart_flask.sh`) |
| 브라우저 탭 ID | `1012263561` (대시보드) |

### 배포 명령어 예시
```bash
# 단일 파일
SSH_AUTH_SOCK="" scp -i ~/Desktop/claude.pem app/static/index.html ec2-user@13.125.219.231:/home/ec2-user/naver-ad/app/static/index.html

# 전체 static 폴더
SSH_AUTH_SOCK="" scp -r -i ~/Desktop/claude.pem app/static/ ec2-user@13.125.219.231:/home/ec2-user/naver-ad/app/static/

# Python 파일 (routes 등)
SSH_AUTH_SOCK="" scp -i ~/Desktop/claude.pem app/routes/unified.py ec2-user@13.125.219.231:/home/ec2-user/naver-ad/app/routes/unified.py

# Flask 재시작 (Python 변경 시)
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@13.125.219.231 "bash /tmp/restart_flask.sh 2>&1"
```

### 백업 위치
- 로컬 파일: `~/Desktop/backups/tbnws-ad-backup-20260224_145548.tar.gz` (416KB)
- DB 덤프: `~/Desktop/backups/tbnws-db-backup-20260224_055821.sql.gz` (1.1MB, 116,039 rows)
- DB 백업 스크립트: `/tmp/db_backup.py` (pymysql 기반, mysqldump는 권한/플러그인 이슈로 사용 불가)

---

## 아키텍처

### 백엔드 (Flask)
- **엔트리**: `run.py` → `app/__init__.py` (`create_app()`)
- **인증**: `app/auth.py` — Google SSO (@tbnws.com 전용), `before_request` 미들웨어
- **SSO**: `app/routes/auth_sso.py` — Google OAuth2 로그인/콜백/로그아웃
- **설정**: `app/config.py` — 환경변수 기반 (`AUTH_USER`, `AUTH_PASS`, `FLASK_PORT` 등)
- **DB**: `app/db.py` — pymysql, `get_conn()`, `bf()` 브랜드필터 함수
- **Blueprint**: `app/routes/__init__.py`에서 25+개 등록
- **주요 라우트**:
  - `app/routes/overview.py` — SA 대시보드 (비용, ROAS, 캠페인, 키워드, cost_forecast)
  - `app/routes/unified.py` — 통합 대시보드 (YoY 비교, account-costs)
  - `app/routes/gfa.py` — GFA 대시보드 (overview, campaigns, waste_creatives)
  - `app/routes/smartstore.py` — 스마트스토어 (매출, 주문, data-freshness)
  - `app/routes/realtime.py` — 실시간 분석 (오늘 매출, 채널, 키워드, 상품)
  - `app/routes/yearly_spend.py` — 연도별 광고비 (overview, monthly, channels, categories, table)
  - `app/routes/chatbot.py` — AI 챗봇 (Claude API + 24개 도구 + SSE 스트리밍)
  - `app/routes/sheets_sync.py` — Google Sheets 양방향 동기화
  - `app/routes/settings.py` — API 키 관리 (CRUD, 연결테스트)
  - `app/routes/scoring_routes.py` — 광고그룹 스코어링
  - `app/routes/funnel.py` — 퍼널 분석
  - `app/routes/diagnosis.py` — 캠페인 진단

### 프론트엔드 (Vanilla JS + ES Modules)
- **엔트리**: `app/static/index.html` → `app/static/js/app.js`
- **설정**: `app/static/js/config.js` — `API_BASE`
- **유틸**: `app/static/js/utils.js` — `apiFetch()`, `getParams()`, `currentAccount`, `fmtWon()`, `fmt()`, `fmtRoas()`
- **라우터**: `app/static/js/router.js` — URL ↔ 탭/계정/날짜 동기화
- **챗봇**: `app/static/js/chatbot.js` — AI 마케팅 어시스턴트 플로팅 위젯
- **탭 모듈** (`app/static/js/tabs/`):
  - `unified.js` — 통합 대시보드 (KPI, YoY, 인사이트, 차트)
  - `overview.js` — SA 대시보드 (전환퍼널, 일별추이)
  - `smartstore.js` — 스마트스토어 대시보드
  - `gfa.js` — GFA 대시보드
  - `ad-yearly.js` — 연도별 광고비 대시보드
  - `ss-realtime.js` — 실시간 분석
  - `settings.js` — API 키 관리
  - 기타 20+개 탭
- **Chart.js**: 모든 차트 렌더링
- **TAB_LOADERS**: `app.js`에서 탭 이름 → 로더 함수 매핑

### 계정/기간 시스템
- 계정: `keychron` (기본), `gtgear`, `aiper` — `selAccount` 드롭다운
- 기간: `selDays` 드롭다운 (기본: 최근 7일) — `getParams()`가 `?account=...&days=...` 또는 `?start=...&end=...` 생성
- URL 동기화: `router.js`가 `?tab=unified&account=keychron&start=...&end=...` 형태로 관리

---

## AI 챗봇 (v2.19.0)

### 아키텍처
- **백엔드**: `app/routes/chatbot.py` — Claude API + Tool Use + SSE 스트리밍
- **프론트**: `app/static/js/chatbot.js` — 플로팅 위젯 (💬 버튼)
- **CSS**: `app/static/styles/chatbot.css`
- **API 키**: 설정 페이지에서 Anthropic API 키 등록 필요
- **모델**: `claude-haiku-4-5-20241022` (기본), credentials에서 변경 가능

### 도구 목록 (24개)
| 카테고리 | 도구명 | 내부 API |
|----------|--------|----------|
| SA | `get_sa_overview` | `/api/overview` |
| SA | `get_sa_campaigns` | `/api/campaigns` |
| SA | `get_sa_keywords` | `/api/keywords` |
| SA | `get_adgroups` | `/api/adgroups` |
| SA | `get_daily_trend` | `/api/daily` |
| SA | `get_dow_heatmap` | `/api/dow-heatmap` |
| SA | `get_scoring` | `/api/scoring` |
| GFA | `get_gfa_overview` | `/api/gfa/overview` |
| GFA | `get_gfa_campaigns` | `/api/gfa/campaigns` |
| GFA | `get_gfa_campaign_tree` | `/api/gfa/campaign-tree` |
| 스마트스토어 | `get_smartstore_overview` | `/api/smartstore/overview` |
| 스마트스토어 | `get_smartstore_orders` | `/api/smartstore/orders` |
| 통합 | `get_unified_yoy` | `/api/unified/yoy` |
| 통합 | `get_cross_channel` | `/api/smartstore/cross-channel` |
| 통합 | `get_campaign_diagnosis` | `/api/diagnosis` |
| 통합 | `get_funnel_analysis` | `/api/funnel/analysis` |
| 통합 | `get_bizmoney` | `/api/bizmoney` |
| 실시간 | `get_realtime_today` | `/api/realtime/today` |
| 실시간 | `get_realtime_channels` | `/api/realtime/channels` |
| 실시간 | `get_realtime_keywords` | `/api/realtime/keywords` |
| 실시간 | `get_realtime_products` | `/api/realtime/product-tracking` |
| 연도별 | `get_yearly_spend_overview` | `/api/yearly-spend/overview` |
| 연도별 | `get_yearly_spend_table` | `/api/yearly-spend/table` |
| 연도별 | `get_yearly_spend_channels` | `/api/yearly-spend/channels` |
| 연도별 | `get_yearly_spend_categories` | `/api/yearly-spend/categories` |

### 특수 파라미터
- 연도별 광고비 도구: `year` (정수), `start`/`end` (YYYY-MM 형식), 계정 구분 없음
- 실시간 도구: `account` 필수, `start`/`end` 선택적
- 일반 도구: `account`, `start`, `end` (YYYY-MM-DD)

---

## 최근 작업 내역

### 완료된 작업 (세션 8 — 연도별 광고비 + 챗봇 확장, 2026-02-26)

22. **연도별 광고비 숫자 포맷 변경** (v2.17.0)
    - `fmtCompact()` 만/억 단위 → 콤마 구분 형식(1,234,567) 변경

23. **채널별 랭킹 파이 차트 추가** (v2.17.0, 커밋 `cbcc2be`)
    - 채널별 비중(TOP 8) + 카테고리별 비중 도넛 차트 추가
    - CSS grid 레이아웃 (1fr + 우측 파이 컬럼)

24. **채널별 랭킹 기간 필터** (v2.18.0, 커밋 `cec22f4`)
    - 2019년 1월~현재까지 YYYY-MM 기간 선택 가능 (기본: 전체)
    - 백엔드 `/api/yearly-spend/channels`에 `start`/`end` 파라미터 추가
    - 모바일 반응형 레이아웃 추가

25. **파이 차트 크기 통일 + 레이아웃 개선** (v2.18.1, 커밋 `34d4da0`)
    - 두 파이 차트를 위아래 수직 배치, 동일 크기로 통일
    - 범례를 bottom → right로 이동, `maxWidth: 130` 고정

26. **Google Sheets 연동 계정 변경** (v2.18.2, 커밋 `c1ebf9e`)
    - 개인 시트 → 팀 공용 시트 (`1dJPQwcscKSapRZM-80Jhrt_dcJRMxWVnQzVplSdSE8M`)
    - 서비스 계정: `tbnws-sheets-sync@gen-lang-client-0744844189.iam.gserviceaccount.com`

27. **AI 챗봇 전체 데이터 연결** (v2.19.0, 커밋 `fd8390b`)
    - 챗봇 도구 10개 → 24개 확장
    - 연도별 광고비(4), 실시간(4), 스코어링, 퍼널, 비즈머니, 요일히트맵, 광고그룹, GFA트리 추가
    - `/help` 명령어 추가 (사용 가능한 질문 예시 표시)
    - 시스템 프롬프트에 사용 가능 데이터 범위 안내 추가

### 완료된 작업 (세션 9 — 스마트스토어 매출 데이터 백필, 2026-02-26)

28. **스마트스토어 매출 데이터 16개월 백필** (v2.19.1, 커밋 `418bfa6`)
    - 엑셀 파일 3개(판매성과_2024-08~2025-12)에서 누락 기간 추출
    - `naver_bizdata_hourly_sales`: +10,582건 (시간대별, 2024-08-24 ~ 2025-11-24)
    - `smartstore_realtime_hourly`: +10,582건 (실시간 대시보드용)
    - `naver_smartstore_order_daily`: +373건 (일별 집계, 2024-08-24 ~ 2025-08-31, keychron)
    - 통합 대시보드 "실제 매출(스토어)" 차트 라인 전 기간 표시 가능
    - 데이터 소스: `/api/smartstore/daily` → `naver_smartstore_order_daily`

### 완료된 작업 (세션 7 — 실시간 KPI 버그 수정 + GFA 레이아웃, 2026-02-26)

20. **상품별 매출 차트 TOP 10 확장 + 가독성 개선** (커밋 `59d334d`, `8dcdfc2`, v2.13.2)
    - 통합 대시보드 + 실시간분석 상품차트: TOP 6/8 → TOP 10 통일
    - 상위 3개 굵은 실선, 7위 이하 점선 구분, 툴팁 매출순 정렬

21. **실시간 예측 백필** — TBNWS_ADMIN `naver_bizdata_hourly_sales` 93일 → TBNWS_AD `smartstore_realtime_hourly`로 백필

18. **실시간 KPI 카드 max()→sum() 버그 수정** (커밋 `9515eec`, v2.13.1)

19. **GFA 캠페인 관리 차트 레이아웃 균형** (커밋 `702ffe3`, v2.13.1)

### 완료된 작업 (세션 4 — 실시간분석 DB 전환 + 안정성)

14. **실시간분석 데이터 소스 API → DB 전환** (v2.10.0)
15. **실시간 분석 전용 DB 분리** (TBNWS_AD)
16. **14일 이상 기간 조회 수정**
17. **Chart.js hidden container 렌더링 버그 수정**

### 완료된 작업 (세션 2 — 스파크라인)

8. **스코어링 90일 추이 스파크라인** (커밋 `61d637b`)
9. **트렌드 바 차트 높이 수정**
10. **모바일 레이아웃 수정** (커밋 `fa9ddc1`)
11. **공휴일/쇼핑이벤트 캘린더 마커** (커밋 `3ff6a11`)

### 완료된 작업 (세션 3 — GFA 캠페인 관리 탭 + 레이아웃)

12. **GFA 캠페인 관리 탭** (커밋 `ff19f66`)
13. **GFA 차트 여백 균형 개선** (커밋 `1f2ad79`, `d76f170`, `702ffe3`)

### 완료된 작업 (세션 1 — 인사이트/퍼널)

1-7. 통합 인사이트, SA 전환퍼널, 절감 카드, 기본 기간 변경, YoY 전환건수

### 미완료 / 다음 세션 작업

#### Google SSO 구현 (진행 중)

**목표**: Basic Auth → Google OAuth2 SSO (@tbnws.com 전용)

**현재 상태**:
- Google Cloud Console "test" 프로젝트 생성 완료 (project: `gen-lang-client-0744844189`)
- OAuth consent screen 페이지까지 진입 (아직 설정 미완료)
- 코드 구현 미시작

**Google Cloud Console 남은 작업**:
1. OAuth consent screen 설정:
   - User type: **Internal** (@tbnws.com 워크스페이스 전용)
   - App name, Authorized domains: `tbe.kr`
2. Credentials → OAuth 2.0 Client ID 생성:
   - Application type: Web application
   - Authorized redirect URI: `http://tbe.kr:8087/auth/callback`
3. CLIENT_ID, CLIENT_SECRET 발급받기

---

## 주요 코드 패턴

### API 호출 (프론트엔드)
```javascript
import { apiFetch } from '../utils.js';
const data = await apiFetch('/api/overview');
// → GET http://tbe.kr:8087/api/overview?account=keychron&days=7
// credentials: 'same-origin' (세션 기반 인증)
```

### 브랜드 필터 (백엔드)
```python
from app.db import get_conn, bf
conn = get_conn()
cur = conn.cursor(pymysql.cursors.DictCursor)
brand_filter = bf(account)  # campaign_name 기반 필터 SQL
cur.execute(f"SELECT ... FROM naver_ad_campaign_daily WHERE stat_date BETWEEN %s AND %s {brand_filter}", (start, end))
```

### 탭 추가 패턴
1. `app/routes/new_tab.py` 생성 (Blueprint)
2. `app/routes/__init__.py`에 Blueprint 등록
3. `app/static/js/tabs/new_tab.js` 생성 (loadNewTab export)
4. `app/static/js/app.js`에 import + TAB_LOADERS 등록
5. `app/static/index.html`에 nav-item + tab-content div 추가

### 차트 (Chart.js)
```javascript
import { charts, destroyChart } from '../utils.js';
destroyChart('myChart');
charts['myChart'] = new Chart(document.getElementById('myChart'), { ... });
```

### 챗봇 도구 추가 패턴
1. `app/routes/chatbot.py`의 `TOOLS` 배열에 도구 정의 추가
2. `TOOL_ENDPOINT_MAP`에 도구→API 매핑 추가
3. 특수 파라미터면 `_execute_tool()`에 분기 추가
4. Flask 재시작 필수

---

## DB 테이블 (주요)

| 테이블 | 용도 |
|--------|------|
| `naver_ad_campaign_daily` | SA 캠페인별 일별 실적 |
| `naver_ad_adgroup_daily` | SA 광고그룹별 일별 실적 |
| `naver_ad_keyword_daily` | SA 키워드별 일별 실적 |
| `naver_ad_bizmoney` | 비즈머니 잔액 |
| `naver_ad_gfa_campaign_daily` | GFA 캠페인별 일별 실적 |
| `naver_ad_gfa_budget` | GFA 예산 정보 |
| `naver_smartstore_order_daily` | 스마트스토어 일별 주문/매출 |
| `yearly_ad_spend` | 연도별 광고비 (year, month, channel, category, spend) |
| `campaign_brand_map` | 캠페인 → 브랜드(계정) 매핑 |
| `strategy_config` | 전략 설정 key-value (Google Sheets ID 등) |
| `api_credentials` | API 키 관리 (settings 페이지) |
| `chat_sessions` | AI 챗봇 세션 관리 |
| `chat_messages` | AI 챗봇 메시지 히스토리 |
| `naver_bizdata_channel_daily` | BizData 채널별 일별 유입/결제 (TBNWS_ADMIN) |
| `naver_bizdata_search_keyword` | BizData 검색 키워드 (TBNWS_ADMIN) |
| `naver_bizdata_product_performance` | BizData 상품별 성과 (TBNWS_ADMIN) |
| `naver_bizdata_hourly_sales` | BizData 시간대별 매출 (TBNWS_ADMIN) |
| `smartstore_realtime_hourly` | 실시간 시간대별 결제/방문 (TBNWS_AD) |
| `smartstore_realtime_products` | 실시간 상품별 매출 트래킹 (TBNWS_AD) |

---

## Google Sheets 연동

| 항목 | 값 |
|------|------|
| 시트 ID | `1dJPQwcscKSapRZM-80Jhrt_dcJRMxWVnQzVplSdSE8M` |
| 시트 이름 | `[TBNWS] 연도별 광고비` |
| 서비스 계정 | `tbnws-sheets-sync@gen-lang-client-0744844189.iam.gserviceaccount.com` |
| 크레덴셜 파일 | `app/google_credentials.json` |
| DB 키 | `strategy_config.google_sheets_yearly_id` |
| 라우트 | `app/routes/sheets_sync.py` |

---

## 업데이트 히스토리 자동 관리 (필수)

**모든 git commit + 서버 배포 시** 아래 절차를 반드시 수행:

1. `app/static/data/changelog.json` 배열 **맨 앞**에 새 항목 추가
2. version: semver 규칙 — 새 기능 `minor++`, 버그/개선 `patch++`, 대규모 변경 `major++`
3. 직전 버전 확인 후 다음 버전 부여

### changelog.json 항목 형식
```json
{
  "version": "X.Y.Z",
  "date": "YYYY-MM-DD",
  "title": "사용자 관점의 변경 제목 (한글)",
  "type": "feature|fix|security|improvement|data|refactor",
  "commits": ["abc1234"],
  "changes": [
    "사용자가 이해할 수 있는 변경 내용 1",
    "변경 내용 2"
  ]
}
```

### type 분류 기준
| type | 사용 시점 |
|------|-----------|
| `feature` | 새 탭, 새 기능, 새 API |
| `fix` | 버그 수정, 레이아웃 수정 |
| `security` | 보안 관련 변경 |
| `improvement` | 기존 기능 개선, UX 향상 |
| `data` | 데이터 수집/배치/백필 |
| `refactor` | 코드 구조 변경 (기능 동일) |

### changes 작성 규칙
- **사용자 관점**으로 작성 (개발 용어 지양)
- "~기능 추가", "~오류 수정", "~성능 개선" 형태
- 3~7개 항목 권장

---

## Dual DB 아키텍처

| DB | 커넥션 함수 | 용도 |
|-----|------------|------|
| `TBNWS_ADMIN` | `get_conn()` | SA/GFA 광고 데이터, BizData 크론 수집 데이터 (`naver_bizdata_*`), 연도별 광고비, 챗봇 세션 |
| `TBNWS_AD` | `get_realtime_conn()` | 실시간 분석 전용 (`smartstore_realtime_*`, `smartstore_channel_detail`, `smartstore_keyword_daily`) |

---

## 스마트스토어 매출 데이터 현황

| 테이블 | 범위 | 건수 | 비고 |
|--------|------|------|------|
| `naver_bizdata_hourly_sales` | 2024-08-24 ~ 2026-02-25 | 12,745 | 시간대별 (keychron) |
| `smartstore_realtime_hourly` | 2024-08-24 ~ 2026-02-26 | 12,770 | 실시간 대시보드용 |
| `naver_smartstore_order_daily` (keychron) | 2024-08-24 ~ 2026-02-25 | 551 | 일별 매출/주문 |
| `naver_smartstore_order_daily` (gtgear) | 2025-02-14 ~ 2026-02-25 | 115 | 일별 매출/주문 |

### 매출 데이터 소스 매핑
- 통합 대시보드 "실제 매출(스토어)" 차트 → `/api/smartstore/daily` → `naver_smartstore_order_daily`
- 실시간 분석 시간대별 차트 → `smartstore_realtime_hourly` (TBNWS_AD)
- 실시간 분석 예측 → `naver_bizdata_hourly_sales` (TBNWS_ADMIN)

---

## 현재 미커밋 변경

- 없음 (clean)
