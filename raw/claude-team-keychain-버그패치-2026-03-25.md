# Keychain 버그패치 (2026-03-25)

## 증상
bridge.py가 Slack 멘션 수신 → Claude CLI 호출 → "❌ 오류: [Errno 2] No such file or directory: 'claude'" 반복 발생

## 근본 원인
1. Claude Code 인증 정보가 macOS Keychain(`Claude Code-credentials`)에 저장됨
2. macOS가 슬립/시간 경과 시 자동으로 keychain 잠금
3. SSH에서 `security set-keychain-settings`(자동잠금 비활성화)가 `User interaction is not allowed`로 적용 불가
4. keychain 잠기면 Claude CLI가 토큰을 못 읽고 "Not logged in" → bridge.py가 Errno 2로 표시

## 패치 내용

### bridge.py — CLI 호출 전 자동 keychain unlock
```python
# 방안 3: macOS Keychain 자동 unlock
import platform
if platform.system() == "Darwin":
    kc_pass = os.environ.get("KEYCHAIN_PASSWORD", "")
    if kc_pass:
        kc_path = Path.home() / "Library" / "Keychains" / "login.keychain-db"
        subprocess.run(
            ["security", "unlock-keychain", "-p", kc_pass, str(kc_path)],
            capture_output=True, timeout=5
        )
```
위치: `call_claude_and_respond()` 함수, `env = os.environ.copy()` 직전

### .env.claude-team — 머신별 비밀번호 추가
```
# 맥미니
KEYCHAIN_PASSWORD=tobe

# 맥북
KEYCHAIN_PASSWORD=56terces
```

### bridge.py — Claude CLI 경로 자동 탐색
```python
import shutil
CLAUDE_BIN = shutil.which("claude")
if not CLAUDE_BIN:
    for p in [
        Path.home() / ".local" / "bin" / "claude",
        Path("/opt/homebrew/bin/claude"),
        Path("/usr/local/bin/claude"),
    ]:
        if p.exists():
            CLAUDE_BIN = str(p)
            break
if not CLAUDE_BIN:
    CLAUDE_BIN = "claude"
```
모든 `"claude"` 문자열을 `CLAUDE_BIN` 변수로 교체

## 결과
- SSH 세션 종료 후에도 bridge.py가 안정적으로 Claude CLI 호출 가능
- 매 호출마다 keychain unlock → 잠금 상태 무관하게 동작
- 맥미니(`/opt/homebrew/bin/claude`), 맥북(`~/.local/bin/claude`) 경로 자동 감지