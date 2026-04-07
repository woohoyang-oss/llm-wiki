# TBNWS 네이버 광고 분석 대시보드 — 세션 기록

## 프로젝트 개요
Flask + Vanilla JS 기반 네이버 광고(SA/GFA) + 스마트스토어 통합 분석 대시보드.
URL: `http://tbe.kr:8087/` (nginx: `tbe.kr/`)

---

## tbe.kr 전체 라우팅 (nginx)

같은 EC2(tbe.kr, Elastic IP 변동 가능)에서 여러 Flask 앱이 포트별로 동작하고, nginx가 URL 경로별로 분배.

| URL | 서비스 | nginx → | Flask 앱 경로 |
|-----|--------|---------|---------------|
| `tbe.kr/` | **투비 마케팅 어드민 포털** (이 프로젝트) | `127.0.0.1:8087` | `/home/ec2-user/naver-ad/` |
| `tbe.kr/j/` | Voice Chat Bot POC (테스트) | `127.0.0.1:8088` | `/home/ec2-user/voice/` |
| `tbe.kr/voice/` | TBNWS Marketing AI (하이브리드) | `127.0.0.1:8088` | `/home/ec2-user/voice/` |
| `tbe.kr/voice/customer/` | Keychron CS AI (하이브리드) | `127.0.0.1:8088` | `/home/ec2-user/voice/` |

> voice 관련 상세는 `/home/ec2-user/voice/CLAUDE.md` 참고 (로컬: `~/voice/CLAUDE.md`)

---

## 인프라 / 배포

| 항목 | 값 |
|------|------|
| 서버 | AWS EC2 `tbe.kr` (IP 변동 가능 — 항상 `tbe.kr` 호스트명 사용) |
| SSH | `SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr` |
| 배포 경로 | `/home/ec2-user/naver-ad/` |
| 포트 | 8087 (Flask) |
| URL | `http://tbe.kr:8087/` |
| DB (메인) | MySQL `222.122.42.221:3306` / `TBNWS_ADMIN` / user: `tbnws22` |
| DB (실시간) | MySQL `222.122.42.221:3306` / `TBNWS_AD` / user: `tbnws22` |
| Git | `origin/main` |
| Python | 시스템 python3 (venv 없음) |
| 배포 방법 | `scp`로 변경 파일 복사 → static 자동 리로드 / Python 변경 시 `sudo systemctl restart tbnws-ad.service` |
| 브라우저 탭 ID | `1012263561` (대시보드) |

### 배포 명령어 예시
```bash
# 단일 파일
SSH_AUTH_SOCK="" scp -o IdentitiesOnly=yes -i ~/Desktop/claude.pem app/static/index.html ec2-user@tbe.kr:/home/ec2-user/naver-ad/app/static/index.html

# 전체 static 폴더
SSH_AUTH_SOCK="" scp -r -o IdentitiesOnly=yes -i ~/Desktop/claude.pem app/static/ ec2-user@tbe.kr:/home/ec2-user/naver-ad/app/static/

# Python 파일 (routes 등)
SSH_AUTH_SOCK="" scp -o IdentitiesOnly=yes -i ~/Desktop/claude.pem app/routes/unified.py ec2-user@tbe.kr:/home/ec2-user/naver-ad/app/routes/unified.py

# Flask 재시작 (Python 변경 시) — systemd 사용!
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr "sudo systemctl restart tbnws-ad.service"

# 전체 서비스 상태 확인
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr "sudo systemctl is-active tbnws-ad sabang-dashboard tbnws-data-api tobe-ai-v2"
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
  - `ss-realtime.js` — SS 실시간 분석
  - `gfa-realtime.js` — GFA 실시간분석
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

### 완료된 작업 (세션 20 — MCP 서버 GFA 실시간 도구 + REST API, 2026-03-10)

56. **MCP 서버에 GFA 실시간분석 + OPS AI 도구 7개 추가** (`tbnws-ai` 레포)
    - `ai/mcp_server.py`에 MCP 도구 7개 구현:
      - `get_gfa_realtime_kpi` — 오늘 GFA 실시간 핵심 지표 (광고비, Cart매출, Cart ROAS, 장바구니, 예산소진율)
      - `get_gfa_realtime_hourly` — 시간대별 누적 광고비·Cart매출 + 시간당 delta + 과거 7일 기반 예측
      - `get_gfa_realtime_campaigns` — 캠페인별 오늘 누적 성과 + 퍼널 태그(TOF/MOF/BOF/ADV)
      - `get_gfa_realtime_alerts` — 6종 자동 알림 (예산소진, ROAS하락, CPM급등, CartRate, 지출예측, GFA↔SS갭)
      - `get_gfa_ops_status` — OPS AI 엔진 현황 (효율스코어, 활성 제안, 위임 현황)
      - `get_gfa_ops_scores` — 캠페인별 6차원 효율 스코어 (0~100)
      - `get_gfa_ops_proposals` — AI 기반 예산 증액/감액/유지 제안 + 근거·시뮬레이션
    - GFA 실시간 4개: TBNWS_AD DB 직접 SQL 쿼리 (`sql/gfa_realtime_queries.py`)
    - OPS AI 3개: Flask API(`tbe.kr:8087/api/gfa-ops-ai/*`) httpx 프록시 + `X-MCP-Internal` 인증

57. **TBNWS_AD 듀얼 DB 풀 구현** (`database.py`)
    - `dev` 사용자 TBNWS_AD 접근 권한 없음 → 별도 `tbnws22` 계정 사용
    - `config.py`: `db_user_ad`, `db_pass_ad`, `db_name_ad` 설정 분리
    - `database.py`: `init_ad_pool()` / `close_ad_pool()` + `fetch_all_ad()` / `fetch_one_ad()` 함수 추가
    - `main.py` lifespan에 AD 풀 초기화/종료 연결

58. **Python 3.9 호환성 수정**
    - `X | None` → `Optional[X]` 타입 힌트 변경 (`database.py`, `mcp_server.py`)
    - EC2 시스템 Python 3.9에서는 MCP SDK 사용 불가 → `.venv` (Python 3.12) 사용 확인

59. **REST API 라우터 추가 → Swagger docs 표시** (`routers/gfa_realtime.py`)
    - MCP 도구 7개를 REST GET 엔드포인트로 래핑 (`_dispatch_tool()` 활용)
    - `/api/v1/gfa-realtime/kpi`, `/hourly`, `/campaigns`, `/alerts` — GFA 실시간분석 (4개)
    - `/api/v1/gfa-realtime/ops/status`, `/ops/scores`, `/ops/proposals` — GFA OPS AI (3개)
    - `advertising` 스코프 API Key 인증 적용
    - Swagger docs (`tbe.kr/data/docs`) 정상 표시 확인

60. **Flask MCP 내부 인증 우회** (`app/auth.py`)
    - `X-MCP-Internal` 헤더 + `/api/gfa-ops-ai/` 경로 조합 시 인증 바이패스
    - MCP 서버 → Flask OPS AI API 호출 시 Basic Auth 없이 접근 가능

### 완료된 작업 (세션 18 — GFA 실시간분석 페이지, 2026-03-06)

53. **GFA 실시간분석 페이지 구현** (v2.32.0)
    - DB: `naver_ad_gfa_campaign_hourly` 테이블 생성 (TBNWS_AD, 매시간 누적 스냅샷)
    - 백엔드: `app/routes/gfa_realtime.py` — 6개 API 엔드포인트
      - `/api/gfa-realtime/today` — 오늘 KPI + 시간별 누적 + 예측
      - `/api/gfa-realtime/campaigns` — 캠페인별 시간 상세
      - `/api/gfa-realtime/funnel` — 퍼널별 집계 + 시간 추이 (Binet & Field 60/40)
      - `/api/gfa-realtime/pacing` — 예산 소진률 + 예측 (campaign_name 기반)
      - `/api/gfa-realtime/compare` — 비교일 시간별 데이터 (hourly/daily fallback)
      - `/api/gfa-realtime/alerts` — 자동 알림 6종 (예산소진, ROAS하락, CPM급등, CartRate, 지출예측, GFA↔SS)
    - 프론트: `app/static/js/tabs/gfa-realtime.js` — 전체 탭 모듈
      - 5대 KPI 카드 (광고비, Cart매출, Cart ROAS, 장바구니, 예산소진)
      - 시간별 듀얼축 콤보 차트 (광고비 bar + Cart매출 line) + 비교일 오버레이
      - 퍼널별 분석 (도넛 + 멀티라인)
      - 캠페인별 예산 소진 프로그레스 바
      - 캠페인 성과 테이블 (정렬 가능, 퍼널 배지)
      - 과거 7일 패턴 기반 예측
    - 예산 쿼리: `naver_ad_gfa_budget`은 `campaign_name` 기반 (campaign_id 없음)
    - 누적 데이터 carry-forward: 데이터 없는 시간은 직전 값 이월 (음수 delta 방지)

54. **GFA hourly 스냅샷 크론 구현**
    - GFA daily 테이블이 30분마다 ON DUPLICATE KEY UPDATE로 누적 갱신되는 구조 파악
    - `cron/gfa_hourly_snapshot.py`: daily 현재값 → hourly 테이블에 시간대별 스냅샷 저장
    - 크론탭 등록: `5,35 * * * *` (매시 5분, 35분)
    - 로그: `/home/ec2-user/naver-ad/gfa_hourly_snapshot.log`
    - 데이터 흐름: GFA 플랫폼 → daily(30분 갱신) → 크론 스냅샷 → hourly(시간대별 축적) → 실시간 페이지

55. **퍼널 분류 접두사 우선 매칭**
    - 버그: "TOF 1127 - 웹사이트 전환 실330만"이 "웹사이트 전환" 패턴으로 BOF 오분류
    - 수정: `_classify_funnel()` — 캠페인명 접두사(TOF/MOF/BOF) 1순위 체크 후 패턴 매칭 fallback
    - 현재 9개 캠페인 분류 결과: ADV 1, BOF 3, MOF 1, TOF 4

### 완료된 작업 (세션 17 — 전체 원복 테스트 Day 1-2 분석 + BOF#2 OFF, 2026-03-05)

47. **3/5 GFA 전체 원복 테스트 중간 분석**
    - 테스트 전체 그림: 2/26 이전 상태로 원복(0113·1211·BOF#2 전부) + 0223만 80K 감액
    - 3/4(Day1): 0113=108K(목표 근접), 0223=88K, BOF#2=108K(재개), 1211=48K, 총 944K(+32%)
    - 3/5(Day2, 18시 기준): 0113=78K(풀데이 ~104K), 0223=52K(풀데이 ~69K), 총 649K(풀데이 ~865K)
    - 0223 감액 효과 확인: 52K에서 Cart 18건, ROAS 78x (기준선 32x의 2.4배 개선)
    - 0113 일간 변동 관찰: 3/4에 51x로 정상, 3/5에 18x로 급락 — 1일 이상치 가능성
    - ADVoost CartRate 21% 정상 유지 (기준 17%+)
    - 전체 Cart Rev 풀데이 ~79M 추정 → 기준선 76.2M 회복 확인

48. **BOF #2 비효율 진단 → OFF 결정**
    - BOF #2 CPM: 3/4 25,244원, 3/5 12,019원 (다른 캠페인 338~2,003원의 6~30배)
    - 근본 원인: 리타게팅 풀 고갈 — 노출 5,869건(BOF#1은 85K), BOF#1과 같은 소수 타겟 경쟁
    - 학습기간 문제가 아닌 구조적 문제 → 시간이 지나도 개선 불가 판단
    - BOF #2 OFF 확정, 절감분 ~71K는 총 광고비 절감에 반영

49. **BOF 0305 장바구니 전환 캠페인 파악**
    - 링크프라이스(대행사)가 운영하는 신규 리타게팅 캠페인, 신규 소재로 발굴 목적
    - 3/5 시작(5K), BOF#2 대체 포지션으로 모니터링 예정
    - BOF #1 + BOF 0305 2개 체제로 운영, 성과 보며 예산 조정

50. **테스트 후 운영 현황 정리**
    - ADVoost ~215K, BOF#1 ~197K, 0113 110K(테스트), 0223 80K(테스트), 1127 ~78K, 1211 ~65K(원복), MOF 0225 ~37K, BOF 0305 신규
    - BOF#2 OFF, 총 예산 ~780K 예상
    - 3/6 체크 포인트: 0113 ROAS 50x대 복귀 여부, BOF#1 CPM 하락 여부(BOF#2 경쟁 해소), BOF 0305 초기 지표

51. **SA 쇼핑검색_풀배열_기계식 효율 분석 → 입찰가 조정**
    - 대장주 K10 PRO SE 바나나축 대상 쇼핑검색 광고 (BOF 퍼널 마지막 단계)
    - Phase 1(1/12~2/26, 입찰 450원): 일 14K, ROAS 26x, CVR 8.6%, 전환 3건/일
    - Phase 2(2/27~3/4, 입찰 600원): 일 50K, ROAS 12x, CVR 4.3%, 전환 5건/일
    - 예산 3.5배 증액 시 클릭 ×3.3 but 전환 ×1.7 → 한계효율 급감, CVR 반토막
    - 쇼핑검색 내 다른 광고그룹 대비 압도적 1위 (ROAS 26x vs 4.7x/9.8x)
    - TOF 확장 + BOF 캡처 전략 의도였으나, 쇼핑검색은 예산이 아닌 수요가 스케일 결정
    - **조정: 예산 50K→35K, 입찰가 600원→480원** (3/5 실행)
    - TOF 효과로 검색량 증가하면 쇼핑검색 전환이 자연 증가 → 그때 증액 검토

52. **0113 예산 판단: 110K 유지**
    - 0113 vs 0223 같은 예산대 비교: 0113이 Cart건수·ROAS 모두 우위
    - 0113@110K: ROAS 54x, Cart 30건/일 (역대) vs 0223@125K: ROAS 32x, Cart 19건/일
    - 110K가 검증된 스윗스팟, 증액 시 한계효율 체감 예상
    - 0223을 50~60K로 추가 감액하고, 절감분은 총 광고비 절감으로 가져가는 방향

### 완료된 작업 (세션 16 — 대장주 제품 분석 + 혼란변수 검토, 2026-03-05)

39. **K10 PRO SE 레트로 대장주 분석 (product_no: 10310833443)**
    - 네이버 상품 10310833443 일별 매출 추이 분석 (2/1~3/4, 31일)
    - 스토어 매출 대비 제품 비중 변화: 23.0%(기준) → 24.5%(0223 이후 평일) — 연휴 제거 시 +1.5%p 소폭
    - SA/GFA 일별 광고비와 제품 매출 상관관계 확인 (GFA 중단 시 1일 lag으로 급락)
    - 요일별 패턴: 주중 7.0~7.7M, 토 5.3M(-28%), 일 6.6M(-11%)
    - 3/3(화) 9.8M 역대 최고 = 삼일절 대체휴일 후 보복소비 + 새학기 효과 (비광고 요인)
    - 3/4(수) ~6.6M = 정상 회귀 (수요일 평균 7.4M 범위)

40. **0223/0113 타겟 비교 실험 프레임 정립**
    - 핵심 발견: 0223과 0113은 **같은 제품 앵커, 같은 인지도 유형, 타겟만 다른 광고**
    - 테스트(3/5~7)의 본질 = 타겟 효율 비교 (0223 타겟 → 0113 타겟 비중 이동)
    - TOF 합계: 174K → 207K (+33K 증액 효과)
    - 판별 지표: 제품 비중 24.5% 유지 여부, 0113 ROAS 54x 이상 유지 여부

41. **혼란 변수(Confounding Variables) 분석**
    - 삼일절(3/1) + 대체휴일(3/2) → 3/3 보복소비 스파이크
    - 새학기 3월 첫 출근 효과, 월초 결제 여유
    - 설날 때도 동일 패턴: 연휴 후 첫 출근 +32~41% 스파이크
    - 테스트 기간(3/5~7)에 토요일 포함 → 결과 왜곡 가능성
    - **결론: 3일 테스트로 깨끗한 결론 어려움. 같은 요일 대비 -15% 이상이면 영향, 비슷하면 1주 연장**

42. **GFA_INSIGHTS.md 섹션 14 추가** — 대장주 분석 전문 기록

43. **KDI 시장지수 단기 감도 부족 발견 + 개선 과제 정립**
    - 네이버 데이터랩 raw: 3/3 "무선키보드" +17%, "키크론" +36% 급등 확인
    - KDI(MA7+Z-score+0~100)는 해당 스파이크를 감지하지 못함
    - 원인: MA7 이동평균이 1일 스파이크를 7일에 걸쳐 희석
    - 개선안: "Market Heat" 단기 경보 레이어 추가 필요
    - GFA_INSIGHTS.md 섹션 14-8 + 15-1 체크리스트 #4 + 15-8 헛다리 #7 추가

44. **분석 프레임워크(섹션 15) 작성** — 헛다리 7건 방지 가이드

45. **액션아이템 재검증: Market Heat 반영** (GFA_INSIGHTS.md 섹션 16)
    - 기존 A1~A3, A5: 변경 없음 (테스트 계획 자체는 유효)
    - A4 판정 기준 수정: 시장 수요 보정 조건 추가 (검색량 ±15% 기준)
    - 신규 A6: 일별 DataLab Raw 트렌드 모니터링 (3/5~7 필수)
    - 신규 A7: 시장 보정 판정표 작성 (테스트 종료 후)
    - 신규 A8: KDI Market Heat 구현 (차순위)
    - 보정 공식: `보정 매출 = 실제 매출 × (기준 검색지수 / 테스트 검색지수)`
    - 3/7(토) 비교 대상: 3/1(삼일절) 대신 2/22(토) 사용

46. **KDI Market Heat 코드 구현** — `trends.py`에 단기 스파이크 감지 레이어 추가

### 완료된 작업 (세션 15 — 예산 테스트 인사이트 반영 + 캐시 개선, 2026-03-04)

37. **GFA 예산 테스트 계획 대시보드 반영** (v2.30.0~v2.30.1)
    - GFA 대시보드: 예산 테스트 현황 패널 추가 (0223 80K, 0113 110K, 분석 완료사항 4건)
    - GFA 대시보드: 예산 조정 테이블에 테스트 오버라이드 (0223→80K 감액, 0113→110K 복구, ADVoost/MOF 0225 테스트기간 증액 보류)
    - 통합 대시보드 "오늘의 마케팅": 동일한 테스트 오버라이드 반영
    - 통합 대시보드: GFA 인사이트에 테스트 진행 배너 추가
    - 장바구니↓매출↑ 디커플링 검증 결과 (BOF 중단 원인, 광고효율 28x→33x) 표시

38. **캐시 버스팅 시스템 도입 + index.html 캐시 방지**
    - `static_routes.py`: `_send_index()` 함수에 `Cache-Control: no-cache, no-store, must-revalidate` 헤더 추가
    - `app.js`: ES module import에 `?v=YYYYMMDD[x]` 버전 파라미터 추가 (gfa.js, unified.js)
    - `index.html`: `app.js` 로드에 버전 파라미터 추가
    - 배포 후 일반 새로고침만으로 최신 JS 로드 가능

### 완료된 작업 (세션 14 — GFA 장바구니 매출 하락 분석 + 캠페인 리네임 발견, 2026-03-04)

36. **GFA 캠페인 예산 재배분 영향 분석 + 캠페인 리네임 매핑**

    **캠페인 리네임 매핑 (2/18→2/19 일괄 변경):**
    | 구 광고명 | 현재 광고명 | 상태 |
    |-----------|-------------|------|
    | [키크론] ADVoost | TOF 0530 - ADVoost | ✅ 운영중 (215K/일) |
    | 3단계 리타게팅 후킹(실12만) | BOF 웹사이트 전환 #1 | ✅ 운영중 (197K/일) |
    | 1단계 990만(실330만) | TOF 1127 - 웹사이트 전환 실330만 | ✅ 운영중 (78K/일) |
    | 1분기 신규 인지도#2601131038 | TOF 0113 - 인지도 및 트래픽 | ⚠️ 감액 (80K→35K) |
    | [1차] 모수확보 | TOF 1211 - 인지도 및 트래픽 | ⚠️ 감액 (58K→17K) |
    | [3차] 리타게팅 | BOF 웹사이트 전환 #2 | ⚠️ 거의 중단 (30K→1K) |
    | [키크론] 카탈로그 판매 | BOF 카탈로그 판매 | 📌 이전 분석에서 비효율 판정, 유지 최소 |
    | (신규) | TOF 0223 - 인지도 및 트래픽 | 🆕 2/23 신설 (122K/일) |
    | (신규) | MOF 0225 - 인지도 및 트래픽 - SE로변경 | 🆕 2/25 신설 (37K/일) |

    > ⚠️ 네이버 GFA 플랫폼 한계: 캠페인명 변경이 DB에서 별도 캠페인으로 기록됨. "중단→신규"처럼 보이지만 실제로는 리네임.

    **주요 발견사항:**
    - GFA 전체 광고비: 776K→718K/일 (▼8%), Cart Rev: 76.4M→74.5M/일 (▼3%)
    - 0223 증액(+97K/일) + ADVoost 소폭 증액(+18K/일) → 장바구니 매출 +7.05M/일 기여
    - 0113 감액(-45K) + 1211 감액(-41K) + BOF#2 거의 중단(-29K) + 기타 → 장바구니 매출 -8.99M/일 손실
    - 순감소 -1.94M/일 = 고ROAS 캠페인 감액 손실이 신규 캠페인 효과를 초과
    - 0223 파이프라인 효과 확인: lag 상관관계 0.55, ADVoost CartRate 14%→19% 개선
    - 단, 0223 스윗스팟(55K/일) 근거 불충분: 9일치 데이터, 초기 4일 학습기간 포함
    - BOF 카탈로그 판매: 이전 세션 분석에서 비효율 판정→끄기 권고한 캠페인, 현재 최소 운영(16K/일) 유지

    **현재 캠페인별 성과 (2/27-3/3 일평균):**
    | 광고명 | 타입 | 하루 광고비 | Cart ROAS | Cart/일 | 비고 |
    |--------|------|------------|-----------|---------|------|
    | TOF 0530 - ADVoost | 전환 | 214,905 | 137x | 138.6 | ★ 최고효율 |
    | BOF 웹사이트 전환 #1 | 전환 | 196,554 | 135x | 104.8 | =구 3단계 리타게팅 |
    | TOF 0223 - 인지도 및 트래픽 | 클릭 | 122,453 | 31x | 18.2 | 파이프라인, 한계효율↓ |
    | TOF 1127 - 웹사이트 전환 | 전환 | 77,871 | 89x | 27.6 | =구 1단계 990만 |
    | MOF 0225 - SE로변경 | 클릭 | 37,238 | 35x | 5.0 | 신규, 모니터링 |
    | TOF 0113 - 인지도 및 트래픽 | 클릭 | 35,439 | 103x | 14.0 | ⚠️ 감액됨 (80K→35K) |
    | TOF 1211 - 인지도 및 트래픽 | 클릭 | 17,106 | 98x | 7.2 | ⚠️ 감액됨 (58K→17K) |
    | BOF 카탈로그 판매 | 클릭 | 16,259 | 42x | 4.2 | 📌 이전 비효율 판정 |
    | BOF 웹사이트 전환 #2 | 전환 | 1,085 | 340x | 2.0 | ⚠️ 거의 중단 |
    | **합계** | | **718,911** | | | |

    **액션 아이템 (현재 광고명 기준, 원래 예산 대비):**

    | 광고명 | 원래 예산 | 현재 지출 | 조정안 | 근거 |
    |--------|----------|----------|--------|------|
    | TOF 0530 - ADVoost | 197K | 215K | 215K 유지 | ROAS 137x, 소폭 증액 상태 안정 |
    | BOF 웹사이트 전환 #1 | 197K | 197K | 197K 유지 | ROAS 135x 안정 |
    | TOF 0223 - 인지도 및 트래픽 | 0 (신규) | 122K | **60K** 감액 | 파이프라인 유효하나 한계효율 급감 |
    | TOF 1127 - 웹사이트 전환 | 78K | 78K | 78K 유지 | ROAS 89x 안정 |
    | MOF 0225 - SE로변경 | 0 (신규) | 37K | 37K 유지 | 신규, 모니터링 중 |
    | TOF 0113 - 인지도 및 트래픽 | **110-130K** | **29-59K** | **미정** | 원래 110K에서 급감, 하루 관찰 후 결정 |
    | TOF 1211 - 인지도 및 트래픽 | **65-68K** | **9-19K** | **미정** | 원래 65K에서 급감, 하루 관찰 후 결정 |
    | BOF 카탈로그 판매 | 16K | 16K | 16K 유지 | 이전 비효율 판정, 현 수준 유지 |
    | BOF 웹사이트 전환 #2 | **103K** | **0-1K** | **미정** | 사실상 중단, 재개 여부 검토 |

    > ⚠️ 0113/1211/BOF#2는 2/26에 급감. 원래 예산 대비 효율: 0113(54x@110K→103x@30K), 1211(35x@65K→98x@19K)
    > ⚠️ 예산 늘리면 ROAS 하락은 확실하지만, 총 장바구니 건수는 증가할 수 있음 (볼륨 vs 효율 트레이드오프)
    > ⚠️ 0223 절감분 62K의 재배분처를 하루 관찰 후 결정

### 완료된 작업 (세션 13 — GFA 피로도 + 예산 인사이트 복구 + 모바일, 2026-03-04)

33. **GFA 소재 피로도 자동 감지** (v2.29.0, 커밋 `41d4bfd`)
    - 단기 감지: 최근 7일 vs 이전 7일 CTR/CPM/CVR 비교
    - 장기 감지: 캠페인 초기 30일 baseline vs 현재 성과 비교 (LATERAL JOIN)
    - `_classify_fatigue(recent, prior, baseline)` + `_fatigue_queries()` 함수 추가
    - `/api/gfa/fatigue-summary` 엔드포인트, GFA 대시보드 인사이트+캠페인관리 배지 연동
    - 기준: 🔴높음(CTR -30%+CPM +20%), 🟡주의(CTR -15% or CVR -20%), 장기는 더 높은 임계값

34. **`/api/gfa/budget-insight` 엔드포인트 복구** (커밋 `e471bc3`)
    - v2.29.0 피로도 작업 중 실수로 삭제된 budget-insight 함수 복구 (~200줄)
    - 통합 대시보드 + GFA 대시보드 양쪽 예산 인사이트 정상 표시

35. **예산 인사이트 테이블 모바일 가로 스크롤** (커밋 `19ea5dc`)
    - 통합 대시보드: 예산 테이블 `overflow-x:auto` + `min-width:700px`, 시뮬레이션 `min-width:560px`
    - GFA 대시보드: 동일하게 예산 테이블 `min-width:700px`, 시뮬레이션 `min-width:560px`

### 다음 세션 작업 예정

- **광고비-매출 상관 분석 인사이트 추가**: GFA 예산 변동 감지, 매출 영향 상관, 주말 효과 분리, 검색트렌드 연동, 퍼널 효율 변화 — 5종 자동 인사이트
- 백엔드: `/api/unified/ad-revenue-correlation` 신규
- 프론트: `renderUnifiedInsights()` 인사이트 풀 확장
- **GFA UTM 연동 개선**: GFA 캠페인 → SmartStore 유입 시 UTM 파라미터 설정하여 BizAdviser 채널 추적 가능하게

### 완료된 작업 (세션 12 — SA vs GFA 비교 차트 개선, 2026-02-27)

32. **SA vs GFA 비교 차트 듀얼 축 콤보 + 레이아웃 정리** (v2.23.0~v2.23.1, 커밋 `b6d9226`~`29a6c90`)
    - 핵심 인사이트 자동 생성 8종 (ROAS 승자, 예산 편중, 최고/비효율 캠페인, 전환0 낭비, 장바구니 기회, CPC/CVR 비교)
    - 불필요 차트 4종(광고비/매출/ROAS/배분) 삭제, 캠페인 매트릭스+콤보 차트 2열 배치
    - ROAS TOP 10 듀얼 Y축 콤보: 막대(일별 광고비) + 꺾은선(캠페인별 ROAS)
    - `/api/compare/top-trend` 백엔드 API 신규 추가

### 완료된 작업 (세션 11 — 보고서 모바일 반응형, 2026-02-27)

31. **키크론 보고서 모바일 반응형 개선** (v2.21.2, 커밋 `ac1bead`)
    - @media(max-width:768px) + @media(max-width:480px) 반응형 CSS 추가
    - 넓은 테이블 12개에 `.table-scroll` 가로 스크롤 래퍼 적용
    - KPI 카드 태블릿 2x2 → 모바일 1열 배치
    - 차트, 인사이트 박스, 타임라인 등 전체 컴포넌트 모바일 대응

### 완료된 작업 (세션 10 — 키크론 보고서 탭, 2026-02-26~27)

29. **2026 키크론 마케팅 보고서 생성** (커밋 `be3587a`)
    - Google Sheets 매출 데이터 연동 + 7년간 종합 분석 보고서 HTML 생성
    - Executive Summary, 브랜드 밸류, 시장 잠재력, 투자 현황, 채널 분석, 진단, 로드맵, KPI 목표

30. **보고서 탭 추가 + 네비게이션 개선** (v2.20.0~v2.21.1, 커밋 `7668df8`~`e8937fd`)
    - 통합 대시보드에 '2026 키크론 보고서' iframe 탭 추가
    - 보고서 내부 목차 네비게이션 자동 숨김 + 마우스 호버 슬라이드
    - static 파일 인증 예외 처리, 보고서 탭 404 수정

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
| `naver_ad_gfa_campaign_hourly` | GFA 시간별 캠페인 누적 스냅샷 (TBNWS_AD) |

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

## 세션 기록

### 세션 20 — MCP 서버 GFA 실시간 도구 + REST API (2026-03-10)

**완료 작업**:
1. **MCP 도구 7개 추가** (`tbnws-ai/ai/mcp_server.py`)
   - GFA 실시간분석 4개: KPI, 시간별 추이, 캠페인별 성과, 자동 알림
   - GFA OPS AI 3개: 엔진 상태, 캠페인 스코어, 예산 재배분 제안
   - SQL 쿼리 파일: `sql/gfa_realtime_queries.py`
   - OPS AI 3개는 Flask API httpx 프록시 (`X-MCP-Internal` 인증)

2. **듀얼 DB 풀** (`tbnws-ai/database.py`)
   - TBNWS_ADMIN (`dev` 유저) + TBNWS_AD (`tbnws22` 유저) 분리
   - `config.py`에 `db_user_ad`/`db_pass_ad` 설정 추가

3. **REST API + Swagger docs** (`tbnws-ai/routers/gfa_realtime.py`)
   - 7개 MCP 도구를 REST GET 엔드포인트로 래핑
   - `tbe.kr/data/docs`에 "GFA 실시간분석" + "GFA OPS AI" 섹션 표시

4. **Flask 인증 우회** (`app/auth.py`)
   - `X-MCP-Internal` 헤더 체크로 MCP 서버 → Flask OPS AI API 접근 허용

**핵심 파일** (tbnws-ai 레포):
- `ai/mcp_server.py` — 7개 도구 + 핸들러 (+392줄)
- `sql/gfa_realtime_queries.py` — GFA 실시간 SQL 쿼리 (NEW)
- `routers/gfa_realtime.py` — REST API 라우터 (NEW)
- `config.py` — TBNWS_AD 별도 크레덴셜
- `database.py` — 듀얼 DB 풀 + Python 3.9 호환
- `main.py` — AD 풀 lifespan + gfa_realtime 라우터 등록

**핵심 파일** (tbnws-ad 레포):
- `app/auth.py` — MCP 내부 인증 우회

### 세션 19 — GFA OPS AI 엔진 통합 + 인텔리전스 리포트 (2026-03-09)

**커밋**: `9cc66db` (v2.35.0)

**완료 작업**:
1. **GFA OPS AI 8개 모듈 통합** (`app/gfa_ops/`)
   - ops_db, ops_config, ops_state, ops_scorer, ops_decision, ops_experiment, ops_runner, ops_learner
   - Flask Blueprint: `/api/gfa-ops-ai/*` (상태/실행/점수/제안/실험/학습)
   - 프론트엔드: `gfa-ops-ai.js` (상태카드/점수테이블/AI제안/실험현황)

2. **EC2 배포 + 버그 3건 수정**
   - `collect_all()` conn 파라미터 오류 수정
   - GFA 세션 파일 경로 심볼릭 링크
   - XSRF 토큰 warm-up 추가

3. **EC2 서비스 systemd 전환**
   - `sabang-dashboard.service` (port 5050) — 신규 등록, 자동 재시작
   - `tbnws-ad.service` (port 8087) — 수동 프로세스 → systemd 전환
   - flask_watchdog cron 비활성화 (systemd와 중복 방지)
   - **재시작 명령**: `sudo systemctl restart tbnws-ad.service`

4. **Keychron 종합 인텔리전스 리포트** (`app/gfa_ops/data/keychron_intelligence_2026Q1.md`)
   - 97일 매출 데이터 분석 (Dec 2025 ~ Mar 2026)
   - 요일별 패턴: 화/목 피크(₩29.1M), 토 최저(₩18.4M)
   - 시즈널: 설 연휴 60% 급감, 설 전 20% 상승, 1월 첫주 피크
   - 19개 외부 채널 + 22개 모델 분석
   - GFA Knowledge Base 이론 프레임워크 적용
   - OPS AI 엔진 캘리브레이션 파라미터

**핵심 파일**:
- `app/gfa_ops/` — 엔진 모듈 (8개)
- `app/routes/gfa_ops_ai.py` — API 라우트
- `app/static/js/tabs/gfa-ops-ai.js` — 프론트엔드
- `app/gfa_ops/data/keychron_intelligence_2026Q1.md` — 종합 인텔리전스
- `app/gfa_ops/data/keychron_weekly_2026W10.md` — 주간 리포트

**EC2 systemd 서비스 목록**:
| 서비스 | 포트 | 명령 |
|--------|------|------|
| tbnws-ad | 8087 | `sudo systemctl restart tbnws-ad.service` |
| sabang-dashboard | 5050 | `sudo systemctl restart sabang-dashboard.service` |
| tbnws-data-api | 8100 | `sudo systemctl restart tbnws-data-api.service` |
| tobe-ai-v2 | 3100 | `sudo systemctl restart tobe-ai-v2.service` |

---

## 현재 미커밋 변경

- 없음 (clean)

### tbnws-ai 레포 (`~/tobe-mcp/tbnws-ai/`)

MCP 서버 레포 — GFA 실시간 도구 추가 관련 변경은 세션 20에서 커밋 완료.
