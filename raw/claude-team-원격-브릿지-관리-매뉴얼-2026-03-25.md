# Claude Team 원격 브릿지 관리 매뉴얼

## 머신 접속 정보

| 머신 | 호스트 | 계정 | 비밀번호 | 역할 |
|------|--------|------|---------|------|
| 맥미니 | tobe-macmini | yangwooho | tobe | coder, tester, reviewer |
| 맥북 | wooho-macbookpro-14 | woohoyang | 56terces | team-lead, project-leader |
| EC2 | 3.34.66.166 | ec2-user | SSH key (~/.ssh/auto-email-poc.pem) | API/Worker/Beat |

## SSH 접속
```bash
# 맥미니
ssh yangwooho@tobe-macmini  # pass: tobe

# 맥북
ssh woohoyang@wooho-macbookpro-14  # pass: 56terces

# EC2
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166
```

## Claude CLI 경로 (머신별 다름)
- 맥미니: `/opt/homebrew/bin/claude`
- 맥북: `/Users/woohoyang/.local/bin/claude`
- bridge.py에 CLAUDE_BIN 자동 탐색 로직 추가됨 (shutil.which + fallback)

## 브릿지 시작 절차 (중요: 순서 반드시 지켜야 함)

### 1단계: Keychain Unlock (필수!)
SSH 세션에서는 macOS Keychain이 잠겨 있어서 Claude CLI가 "Not logged in" 에러 발생.
반드시 bridge 실행 전에 keychain을 unlock해야 함.

```bash
# 맥미니
security unlock-keychain -p 'tobe' ~/Library/Keychains/login.keychain-db

# 맥북
security unlock-keychain -p '56terces' ~/Library/Keychains/login.keychain-db
```

### 2단계: 기존 브릿지 종료
```bash
pkill -f bridge.py
sleep 2
```

### 3단계: 브릿지 시작
```bash
# 맥미니 (coder, tester, reviewer)
cd ~/auto-email
nohup /opt/homebrew/bin/python3 claude-team/bridge.py coder > /tmp/bridge-coder.log 2>&1 &
nohup /opt/homebrew/bin/python3 claude-team/bridge.py tester > /tmp/bridge-tester.log 2>&1 &
nohup /opt/homebrew/bin/python3 claude-team/bridge.py reviewer > /tmp/bridge-reviewer.log 2>&1 &

# 맥북 (team-lead, project-leader)
cd ~/auto-email
nohup python3 claude-team/bridge.py team-lead > /tmp/bridge-tl.log 2>&1 &
nohup python3 claude-team/bridge.py project-leader > /tmp/bridge-pl.log 2>&1 &
```

### 4단계: 확인
```bash
tail -5 /tmp/bridge-*.log
# "⚡️ Bolt app is running!" 이 보이면 정상
```

## expect 스크립트로 원격 실행 시 주의사항
- `$2` 같은 변수가 expect와 충돌 → 별도 .sh 파일로 분리
- `awk '{print $2}'` 대신 `pkill -f bridge.py` 사용

## Claude CLI 인증

### 인증 정보 저장 위치
- macOS Keychain: `Claude Code-credentials` (service name)
- account: OS 사용자명 (yangwooho, woohoyang)

### 계정 배치
- 맥미니: ai.login@tbnws.com (Claude Max)
- 맥북: wooho.yang@tbnws.com (Claude Max)

### 토큰 만료 시 재로그인
1. 해당 머신에서 직접 `claude` → `/login` (브라우저 필요)
2. 또는 Screen Sharing으로 원격 접속 후 로그인

### Keychain에서 credential 확인
```bash
security unlock-keychain -p 'PASSWORD' ~/Library/Keychains/login.keychain-db
security find-generic-password -s 'Claude Code-credentials' -a 'USERNAME' -w
```
→ JSON 형태로 accessToken, refreshToken, expiresAt 반환

## 코드 배포 (git pull)
각 머신에서 GitHub 인증이 안 될 수 있음. PAT 사용:
```bash
git remote set-url origin https://ghp_***MASKED***@github.com/woohoyang-oss/auto-email.git
git pull origin main
```

## 트러블슈팅

### "Not logged in" 에러
→ keychain unlock 안 됨 또는 토큰 만료. 위 절차 참조.

### "No module named 'dotenv'" (맥미니)
→ python3.14로 업그레이드되면서 패키지 소실
```bash
/opt/homebrew/bin/pip3 install --break-system-packages python-dotenv slack-bolt slack-sdk
```

### "[Errno 2] No such file or directory: 'claude'"
→ bridge.py가 claude CLI 경로를 못 찾음. CLAUDE_BIN 자동 탐색 로직으로 해결됨.

### git pull 실패 (local changes)
```bash
git stash && git pull origin main
```

## Slack 핑 테스트
```bash
# PL 토큰으로 다른 봇에게 (자기 토큰으로 자기한테 멘션하면 무시됨)
PL_TOKEN="xoxb-***MASKED***"
curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer $PL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channel":"C0AN03UNL0M","text":"<@BOT_USER_ID> 핑!"}'
```

Bot User IDs:
- TL: U0AN3FEMS6N
- PL: U0AN0EWHXJ9
- Coder: U0ANXPNJWAC
- Reviewer: U0AMN2KKML7
- Tester: U0AN73VFGSY