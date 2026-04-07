# GFA 자동 다운로드/업로드 자동화 (2026-03-05~06)

> 3개 세션에 걸쳐 진행 (컨텍스트 롤오버 2회)

## 목표

GFA(네이버 성과형 디스플레이) 리포트를 **매시간 자동으로 다운로드 → 서버 업로드**하는 자동화 파이프라인 구축.
GFA는 API가 없어 수동 CSV 다운로드만 가능했음 → Playwright 브라우저 자동화로 해결.

## 최종 결과

| 항목 | 상태 |
|------|------|
| 4개 유닛 다운로드 (캠페인/광고그룹/애셋그룹/소재) | ✅ 성공 |
| EC2 서버 업로드 (API Key 인증) | ✅ 성공 |
| DB 적재 확인 | ✅ 성공 |
| 텔레그램 알림 (@Kimbiseo33_bot) | ✅ 성공 |
| 매시간 자동 실행 (LaunchAgent) | ✅ 정상 동작 확인 |
| 동시 실행 방지 (Lock 파일) | ✅ 구현 |
| 다운로드 목록 60개 제한 대응 | ✅ 자동 삭제 |
| TargetClosedError 자동 복구 | ✅ CDP 재연결 |
| 김비서 봇 수동 명령어 | ✅ /gfa 명령어 |

## 아키텍처

```
[openclaw MacBook M1] 매시간
  ├── LaunchAgent: com.tobe.gfa-download (StartInterval=3600)
  ├── LaunchAgent: com.tobe.gfa-bot (KeepAlive, 텔레그램 봇)
  ├── run_gfa.sh (wrapper)
  │   └── Chrome CDP (:9222) 확인/실행
  │   └── gfa_download.py (v9.2)
  │       ├── [0] 다운로드 목록 cleanup (60개 제한 대응)
  │       ├── Playwright → Chrome (CDP) → GFA 페이지
  │       │   ├── 기존 탭 재사용 + 여분 탭 정리
  │       │   ├── 4개 유닛별 URL 네비게이션
  │       │   ├── 날짜 설정 (어제)
  │       │   ├── 열추가 (모든 컬럼 체크)
  │       │   ├── 다운로드 요청 → 확인 → 3초 대기
  │       │   └── TargetClosedError → CDP 재연결 + 재시도
  │       ├── [2] 45초 대기 → 파일 다운로드 (시간 필터링)
  │       ├── ZIP 해제 → CSV 추출
  │       ├── HTTP 업로드 → tbe.kr:8087/api/gfa/upload
  │       │   └── X-API-Key: tobe-gfa-upload-2026
  │       ├── [3] 다운로드 목록에서 처리된 항목 삭제
  │       └── 텔레그램 알림 → @Kimbiseo33_bot
  │
  ├── gfa_bot.py (텔레그램 봇 핸들러)
  │   └── /gfa, /gfa 3-6, /gfa today, /gfa status, /gfa help
  │
  └── ~/gfa-auto/
      ├── gfa_download.py        # 메인 스크립트 (v9.2)
      ├── gfa_bot.py             # 텔레그램 봇 명령어 핸들러
      ├── run_gfa.sh             # cron wrapper
      ├── venv/                  # Python 3.12 + playwright
      ├── chrome-profile/        # Chrome 프로필 (네이버 로그인 유지)
      ├── downloads/             # 임시 다운로드
      │   └── uploaded/          # 업로드 완료 아카이브
      └── logs/                  # 실행 로그 + 스크린샷
```

## 주요 파일 위치

### openclaw (MacBook)
| 파일 | 경로 | 설명 |
|------|------|------|
| 메인 스크립트 | `/Users/wooho/gfa-auto/gfa_download.py` | v9.2 (최종) |
| 봇 핸들러 | `/Users/wooho/gfa-auto/gfa_bot.py` | 텔레그램 /gfa 명령어 |
| Wrapper | `/Users/wooho/gfa-auto/run_gfa.sh` | Chrome 확인 + venv 활성화 |
| LaunchAgent (배치) | `~/Library/LaunchAgents/com.tobe.gfa-download.plist` | 매시간 실행 |
| LaunchAgent (봇) | `~/Library/LaunchAgents/com.tobe.gfa-bot.plist` | KeepAlive 봇 |
| Chrome 프로필 | `/Users/wooho/gfa-auto/chrome-profile/` | 네이버 로그인 세션 |
| Venv | `/Users/wooho/gfa-auto/venv/` | Python 3.12 |

### EC2 서버
| 파일 | 경로 | 설명 |
|------|------|------|
| auth.py (패치) | `/home/ec2-user/naver-ad/app/auth.py` | API Key 인증 추가 |
| auth.py 백업 | `/home/ec2-user/naver-ad/app/auth.py.bak` | 패치 전 원본 |

### 로컬 (개발)
| 파일 | 경로 | 설명 |
|------|------|------|
| v9 소스 | `/tmp/gfa_download_v9.py` | 최신 로컬 사본 |
| 봇 소스 | `/tmp/gfa_bot.py` | 최신 로컬 사본 |
| 봇 plist | `/tmp/com.tobe.gfa-bot.plist` | 최신 로컬 사본 |

## 설정 정보

### Chrome CDP
```
Port: 9222
Profile: ~/gfa-auto/chrome-profile/
Launch: open -na "Google Chrome" --args --user-data-dir=... --remote-debugging-port=9222
```

### GFA
- 계정 ID: 26343 (키크론)
- URL: `https://gfa.naver.com/adAccount/accounts/26343/report/performance?adUnit={UNIT}`
- 유닛: CAMPAIGN, AD_SET, ASSET_GROUP, CREATIVE
- 네이버 로그인: demigodsj

### 업로드 API
- URL: `http://tbe.kr:8087/api/gfa/upload`
- 인증: `X-API-Key: tobe-gfa-upload-2026`
- Method: POST multipart/form-data (file + account=keychron)

### 텔레그램
- Bot: @Kimbiseo33_bot (김비서)
- Token: `8201826147:AAGksnINT3JkaaL3IAbcuPOdCmFC7Iv3VnM`
- Chat ID: `8188602191` (W 개인)
- ※ 이전 설정: 투비클로드봇(`8238829073:...`) + 투비클로드 그룹(`-5280844598`) → 변경됨

### EC2 SSH
```bash
SSH_AUTH_SOCK="" ssh -i /Users/woohoyang/Desktop/claude.pem \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  ec2-user@tbe.kr
```

### openclaw SSH
```bash
SSH_AUTH_SOCK="" sshpass -p 'wooho' ssh \
  -o StrictHostKeyChecking=no -o PubkeyAuthentication=no \
  -o NumberOfPasswordPrompts=1 wooho@openclaw
```

### DB
- Host: 222.122.42.221:3306
- User: tbnws22 / dnpfzja!
- DB: TBNWS_ADMIN

## 개발 과정 & 해결한 문제들

### 1. GFA 페이지 구조 파악
- Ant Design 컴포넌트 기반
- 모달: `.ant-modal-body` (NOT `.ant-modal-content`)
- 체크박스: `.ant-checkbox-wrapper`
- 다운로드 버튼: `<button class="ant-btn-link">다운로드</button>`
- 캘린더: `td[title="YYYY-MM-DD"]`
- 다운로드 목록: "전체 선택" 체크박스 + "삭제" 버튼 (상단)

### 2. 라디오 버튼 유닛 전환 실패
**문제**: `input[type="radio"][value="AD_SET"]` click(force=True)가 실제로 유닛 변경 안 됨

**해결**: URL 네비게이션으로 변경
```python
url = f'{GFA_BASE}/report/performance?adUnit={unit_value}'
await page.goto(url, ...)
```

### 3. 다운로드 파일 필터링
**문제**: 이전 요청의 파일까지 모두 다운로드 (37개)

**해결**: 실행 시작 시간 기준 datetime 필터링
```python
run_start_str = run_start_dt.strftime('%Y.%m.%d. %H:%M')
if row_dt_str >= run_start_str:
    # 이번 실행 파일만 다운로드
```

### 4. 업로드 401 인증 실패
**문제**: EC2 Flask 앱이 Google OAuth 세션 인증 사용 → API Key 없음

**해결**: `auth.py`에 API Key bypass 추가
```python
api_key = request.headers.get('X-API-Key')
if api_key and api_key == UPLOAD_API_KEY:
    if path.startswith('/api/gfa/'):
        return None  # 인증 통과
```

### 5. Playwright async await 누락
**문제**: `'coroutine' object has no attribute 'suggested_filename'`
**해결**: `download = await dl_info.value` (await 추가)

### 6. ~/Downloads 권한 에러
**해결**: `~/gfa-auto/downloads/`로 경로 변경

### 7. EC2 SSH 사용자 오류
**문제**: `ubuntu@` → Permission denied
**해결**: `ec2-user@` + claude.pem + IdentitiesOnly=yes

### 8. 열추가 후 다운로드 버튼 미발견
**문제**: 열추가 팝업 닫힌 후 overlay 잔존 → 버튼 순회 시 stale element
**해결**: body.click() (overlay 해제) + 2초 대기 + try/except

### 9. 동시 실행 충돌
**문제**: LaunchAgent RunAtLoad가 수동 테스트 직후 실행 → 같은 페이지 충돌
**해결**: Lock 파일 메커니즘 (10분 타임아웃)

### 10. 다운로드 목록 60개 제한 (v9 해결)
**문제**: GFA 다운로드 요청 목록이 최대 60개 → 꽉 차면 새 요청이 등록 안 됨
- v8.2 이후 매시간 4개씩 누적 → 03:32 이후 모든 자동실행 0/4 실패
- 로그에는 "Requested!" + "Confirmed: 확인"이 나오지만 실제 등록 안 됨

**해결**: 매 실행 시작 시 "전체 선택" → "삭제"로 기존 항목 모두 제거
```python
async def cleanup_download_list(page):
    select_all = modal.locator('.ant-checkbox-wrapper:has-text("전체 선택")').first
    await select_all.click()
    del_btn = page.locator('button:has-text("삭제")').first
    await del_btn.click()
    # 확인 다이얼로그 (~60초 소요)
```

### 11. Chrome 탭 누적 → TargetClosedError (v9.2 해결)
**문제**: `ctx.new_page()` 호출마다 새 탭 생성, 에러 시 탭이 안 닫힘 → Chrome 리소스 압박 → 탭 자동 종료
- v9에서 4개 GFA 탭 누적 발견
- `BrowserContext.new_page: Target page, context or browser has been closed` (context까지 죽음)

**해결**: 기존 첫 번째 탭 재사용 + 여분 탭 정리 + CDP 재연결
```python
# 기존 탭 재사용
page = ctx.pages[0]
# 여분 탭 정리
for extra_page in ctx.pages[1:]:
    await extra_page.close()
# 에러 시 CDP 재연결
browser = await p.chromium.connect_over_cdp(f'http://localhost:{CDP_PORT}')
ctx = browser.contexts[0]
page = ctx.pages[0]
```

### 12. 텔레그램 봇/채팅 변경
- 초기: 투비클로드봇 → 투비클로드 그룹 (`-5280844598`)
- 변경: @Kimbiseo33_bot → W 개인 (`8188602191`)
- 사유: 유저가 김비서 봇 + 개인 채팅으로 변경 요청

## DB 데이터 현황 (2026-03-06 확인)

| 테이블 | 총 행수 | 3/5 데이터 |
|--------|---------|-----------|
| naver_ad_gfa_campaign_daily | 6,494 | 10행 |
| naver_ad_gfa_adgroup_daily | 1,502 | 15행 |
| naver_ad_gfa_creative_daily | 29,614 | 74행 |

> 애셋 그룹은 별도 테이블 없음 → campaign_daily에 통합 (서버 CSV 파싱 로직에 의해)

## 테스트 결과

| 버전 | 결과 | 문제 |
|------|------|------|
| v5 | 미테스트 | — |
| v6 | 다운로드 37개 | 모달 셀렉터 오류, await 누락 |
| v7 | 4/4 다운로드 성공 | 업로드 401, 시간 필터 부정확 |
| v8 | 4/4 다운로드+업로드 성공 | 첫 성공 (02:27) |
| v8.1 | 0/4 실패 | text-is 너무 strict, overlay 잔존 |
| v8.2 | 4/4 다운로드+업로드+텔레그램 성공 | 안정 버전 |
| v9 | cleanup 성공, 4/4 요청 성공 | 2nd unit TargetClosedError (new_page 누적) |
| v9.1 | cleanup+recovery 추가 | ctx.new_page()도 실패 (context dead) |
| v9.2 | **4/4 완벽 성공 (LaunchAgent)** | 기존 탭 재사용 + CDP 재연결 = 최종 안정 |

### v9.2 주요 개선사항
1. **다운로드 목록 60개 제한 해결**: 매 실행 시 기존 항목 전체 삭제
2. **Chrome 탭 누적 방지**: 기존 첫 번째 탭 재사용, 여분 탭 자동 정리
3. **CDP 재연결**: TargetClosedError → Chrome CDP 재연결 후 재시도
4. **다운로드 후 목록 삭제**: 성공한 항목 체크 → 삭제
5. **확인 모달 개선**: force=True 제거, 3초 대기 후 Escape

### v9.2 성공 로그 (08:25 LaunchAgent 자동실행)
```
[0] cleanup: 1 entry 삭제
[1/2] 4개 유닛 요청 (AD_SET 재연결 후 성공)
[2/2] 4/4 다운로드+업로드: 소재 465건, 애셋 7건, 광고그룹 95건, 캠페인 67건
[3] 4개 항목 목록 삭제, 텔레그램 전송
```

## 김비서 봇 명령어

| 명령어 | 설명 |
|--------|------|
| /gfa | 어제 데이터 다운로드 (기본) |
| /gfa 3-6 | 3월 6일 데이터 |
| /gfa today | 오늘 데이터 |
| /gfa 2026-03-05 | 특정 날짜 (YYYY-MM-DD) |
| /gfa status | 마지막 실행 상태 |
| /gfa help | 도움말 |

봇 서비스: `com.tobe.gfa-bot` LaunchAgent (KeepAlive)
파일: `/Users/wooho/gfa-auto/gfa_bot.py`

## 알려진 제한사항

1. **애셋 그룹 테이블 없음**: CSV 파싱 시 campaign으로 분류
2. **네이버 로그인 세션**: 쿠키 만료 시 수동 재로그인 필요
3. **Chrome 프로세스**: 재부팅 시 run_gfa.sh가 자동 재실행
4. **다운로드 대기 시간**: 45초 (최대 3회 재시도)
5. **삭제 확인 지연**: GFA 삭제 확인 다이얼로그가 ~60초 소요 (정상)
6. **TargetClosedError**: 간헐적 발생 → CDP 재연결로 자동 복구
7. **전체 선택 실패**: 항목이 1개일 때 간헐적으로 전체 선택 체크박스 못 찾음 (cleanup skip, 영향 없음)

## LaunchAgent 관리

```bash
# 상태 확인
launchctl list | grep gfa

# 다운로드 배치 수동 실행
cd ~/gfa-auto && source venv/bin/activate && python3 gfa_download.py
cd ~/gfa-auto && source venv/bin/activate && python3 gfa_download.py 2026-03-06  # 특정 날짜

# 배치 중지/시작
launchctl unload ~/Library/LaunchAgents/com.tobe.gfa-download.plist
launchctl load ~/Library/LaunchAgents/com.tobe.gfa-download.plist

# 봇 중지/시작
launchctl unload ~/Library/LaunchAgents/com.tobe.gfa-bot.plist
launchctl load ~/Library/LaunchAgents/com.tobe.gfa-bot.plist

# 로그 확인
tail -f ~/gfa-auto/logs/gfa_download.log
tail -f ~/gfa-auto/logs/bot_stdout.log
cat ~/gfa-auto/logs/cron_*.log | tail -50
```

## 타임라인 (3번째 세션 / 2026-03-06 08:00~)

| 시간 | 작업 |
|------|------|
| 07:54 | SSH 재연결, `python3 gfa_download.py 2026-03-06` 테스트 실행 |
| 07:57 | 0/4 실패 확인 — 다운로드 목록 60개 꽉 참 발견 |
| 08:00 | v9 작성 (cleanup + deletion 추가) |
| 08:03 | v9 배포, 테스트 → cleanup 50개 삭제 성공, 캠페인 요청 성공 |
| 08:05 | AD_SET에서 TargetClosedError → v9.1 에러복구 추가 |
| 08:07 | v9.1 배포, 테스트 → 4/4 완벽 성공 (수동 실행) |
| 08:12 | LaunchAgent 자동실행 → 3rd unit에서 context dead + new_page 실패 |
| 08:15 | Chrome 탭 4개 누적 발견 → v9.2 작성 (기존 탭 재사용) |
| 08:19 | v9.2 수동 테스트 → 4/4 성공 (여분 탭 3개 정리) |
| 08:25 | LaunchAgent 자동실행 → **4/4 성공** (AD_SET 재연결 후) |
| 08:25 | 김비서 봇 서비스 시작 (`com.tobe.gfa-bot` LaunchAgent) |
| 08:32 | 텔레그램 성공 보고 전송 |
| 08:35 | 세션 MD 저장 |
