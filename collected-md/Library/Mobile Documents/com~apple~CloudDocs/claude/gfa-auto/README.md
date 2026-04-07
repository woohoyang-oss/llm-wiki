# GFA Auto — 네이버 성과형 디스플레이 광고 리포트 자동화

네이버 GFA(성과형 디스플레이 광고) 리포트를 자동 다운로드하고 데이터 서버에 업로드하는 자동화 시스템.

## 구성 요소

| 파일 | 역할 |
|------|------|
| `gfa_download.py` | **메인 스크립트** (v11). Playwright+CDP로 GFA 리포트 다운로드 → API 서버 업로드 → 텔레그램 알림 |
| `gfa_bot.py` | 텔레그램 봇 (`/gfa` 명령 수신). ⚠️ OpenClaw과 봇 토큰 충돌로 비활성화 — 아래 참고 |
| `run_gfa.sh` | cron wrapper. Chrome CDP 확인 → gfa_download.py 실행 → 로그 정리 |

## 동작 방식

```
[30분 간격 LaunchAgent]
    └─ run_gfa.sh
        ├─ Chrome CDP (port 9222) 확인/시작
        └─ gfa_download.py
            ├─ GFA 웹 (gfa.naver.com) 접속 (기존 로그인 세션 유지)
            ├─ Route interception으로 날짜 설정 (v11)
            ├─ 캠페인/광고그룹/애셋그룹/광고소재 4단위 리포트 다운로드
            ├─ ZIP → CSV 추출 → API 서버 업로드
            ├─ 다운로드 리스트 정리 (삭제)
            └─ 텔레그램 알림 (sendMessage)
```

## 설정

### GFA 계정
- Account ID: `26343`
- URL: `https://gfa.naver.com/adAccount/accounts/26343`

### 업로드 서버
- URL: `http://13.125.219.231:8087/api/gfa/upload`
- API Key: `tobe-gfa-upload-2026`

### 텔레그램 알림
- Bot: `@Kimbiseo33_bot`
- Chat ID: `8188602191` (DM)
- **sendMessage만 사용** (polling 안 함)

### Chrome CDP
- Port: `9222`
- Profile: `~/gfa-auto/chrome-profile`
- 네이버 로그인 세션이 유지되어야 함

## LaunchAgent

### `com.tobe.gfa-download.plist` — 30분 자동 실행 ✅ 활성
```xml
<key>StartInterval</key>
<integer>1800</integer>
```
```bash
# 상태 확인
launchctl list | grep gfa-download

# 중지/시작
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.tobe.gfa-download.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.tobe.gfa-download.plist
```

### `com.tobe.gfa-bot.plist` — 텔레그램 봇 ❌ 비활성화
OpenClaw과 동일 봇 토큰으로 `getUpdates` polling 충돌 (409 Conflict) 때문에 비활성화.
`/gfa` 명령은 OpenClaw 스킬로 대체.

## 리포트 단위

| 코드 | 이름 | 설명 |
|------|------|------|
| `CAMPAIGN` | 캠페인 | 최상위 |
| `AD_SET` | 광고 그룹 | 타겟팅 단위 |
| `ASSET_GROUP` | 애셋 그룹 | 소재 묶음 |
| `CREATIVE` | 광고 소재 | 개별 소재 |

## 개발 히스토리

| 버전 | 내용 |
|------|------|
| v9.2 | 초기 자동화. CDP 연결, 다운로드, 업로드 |
| v10 | 기본 날짜를 yesterday → today 변경 (매시간 갱신용) |
| v11 | Route interception으로 날짜 설정 문제 해결 (날짜 picker UI 불안정 우회) |

## 트러블슈팅

### OpenClaw 텔레그램 409 Conflict (2026-03-08)

**증상**: OpenClaw 게이트웨이의 텔레그램 polling이 `409: Conflict: terminated by other getUpdates request` 에러로 실패.

**원인**: `gfa_bot.py`와 OpenClaw이 **동일한 봇 토큰**으로 `getUpdates` long-polling을 동시에 실행.

**해결**:
1. `gfa_bot.py` LaunchAgent 비활성화 (`com.tobe.gfa-bot.plist` bootout)
2. Telegram `logOut` API 호출 → 전체 세션 초기화 (쿨다운 ~30분 소요)
3. OpenClaw 게이트웨이 재시작 → 텔레그램 정상 연결

**구조 변경**:
- `gfa_bot.py`의 `/gfa` 명령 기능 → OpenClaw 스킬로 통합
- `gfa_download.py`의 `sendMessage` 알림 → 충돌 없음 (polling 안 함)
- 30분 자동 다운로드 (`run_gfa.sh` + LaunchAgent) → 그대로 유지

## 로컬 환경

- 실행 머신: MacBook Pro 13" M1 (`wooho-macbookpro-13`)
- Python: venv (`~/gfa-auto/venv`)
- 의존성: `playwright`, `requests`
- 로그: `~/gfa-auto/logs/`
