# Keychain 잠금 문제 — Bridge Errno 2 근본 원인

## 증상
- bridge.py가 Slack 멘션을 수신하면 Claude CLI를 subprocess로 호출
- Claude CLI가 macOS Keychain에서 OAuth 토큰을 읽으려 함
- Keychain이 잠겨 있으면 토큰을 못 읽고 "Not logged in" 반환
- bridge.py는 이를 `❌ CLI 오류:` 또는 `❌ 오류: [Errno 2] No such file or directory: 'claude'`로 표시

## 근본 원인
1. Claude Code 인증 정보는 macOS Keychain `Claude Code-credentials`에 저장됨
2. SSH 세션에서는 Keychain이 기본적으로 잠겨 있음
3. `security unlock-keychain`으로 풀어도 **SSH 세션이 끊기면 다시 잠김**
4. bridge.py는 백그라운드(nohup)로 실행되므로, SSH 세션 종료 후 keychain 접근 불가

## 현재 임시 해결 (불안정)
```bash
security unlock-keychain -p 'PASSWORD' ~/Library/Keychains/login.keychain-db
nohup python3 bridge.py coder > /tmp/bridge-coder.log 2>&1 &
```
→ SSH 세션이 열려 있는 동안만 작동. 세션 닫으면 다음 CLI 호출부터 실패.

## 영구 해결 방안 (3가지)

### 방안 1: ANTHROPIC_API_KEY 환경변수 (권장)
- Keychain 의존 없이 API 키로 직접 인증
- `.env.claude-team`에 추가:
```
ANTHROPIC_API_KEY=sk-ant-***MASKED***
```
- bridge.py의 env에 이미 os.environ.copy()를 사용하므로 자동 적용
- **장점**: Keychain 불필요, 가장 안정적
- **단점**: API 키 발급 필요, Max 플랜 기능 사용 불가할 수 있음

### 방안 2: Keychain 자동 잠금 비활성화
- macOS에서 Keychain Access 앱 → 설정 → "자동 잠금" 해제
- 또는 CLI로: `security set-keychain-settings ~/Library/Keychains/login.keychain-db`
  (타임아웃 없이 설정하면 자동잠금 비활성화)
- **장점**: 기존 OAuth 인증 유지
- **단점**: 보안 약화, 머신별 수동 설정 필요

### 방안 3: bridge.py에서 Keychain unlock 자동화
- Claude CLI 호출 전에 매번 keychain unlock 실행
- bridge.py의 call_claude_and_respond() 함수에 추가:
```python
subprocess.run(
    ["security", "unlock-keychain", "-p", KEYCHAIN_PASSWORD,
     os.path.expanduser("~/Library/Keychains/login.keychain-db")],
    capture_output=True
)
```
- **장점**: 코드 수준 해결, SSH 세션 무관
- **단점**: 비밀번호가 코드/환경변수에 저장됨

## 머신별 계정/비밀번호 정보
- 맥미니: yangwooho / tobe (ai.login@tbnws.com)
- 맥북: woohoyang / 56terces (wooho.yang@tbnws.com)

## Credential 위치 및 포맷
```
서비스명: Claude Code-credentials
계정명: OS 사용자명 (yangwooho, woohoyang)
값: JSON {"claudeAiOauth":{"accessToken":"sk-ant-oat01-...","refreshToken":"sk-ant-ort01-...","expiresAt":...}}
```

## 검증 방법
```bash
# 1. Keychain unlock
security unlock-keychain -p 'PASSWORD' ~/Library/Keychains/login.keychain-db

# 2. Credential 읽기
security find-generic-password -s 'Claude Code-credentials' -a 'USERNAME' -w

# 3. CLI 테스트
claude -p 'pong' --output-format json --dangerously-skip-permissions
```

## 결론
방안 3 (bridge.py에 unlock 자동화)을 먼저 적용하고, 장기적으로 방안 1 (API 키)로 전환 권장.