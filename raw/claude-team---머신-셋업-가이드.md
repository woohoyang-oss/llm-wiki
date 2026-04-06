# 클로드 팀 머신 셋업 가이드

## 이 가이드를 사용하는 법
각 머신(맥미니, 스튜디오)에서 Claude Code를 열고 이 가이드를 불러와서 실행하면 됩니다.

## 사전 조건
- Claude CLI 설치 + 엔터프라이즈 계정 로그인 완료
- Python 3.9+ 설치
- Git 설치

## 1. 레포 클론
```bash
cd ~
git clone https://github.com/woohoyang-oss/auto-email.git
cd auto-email
git checkout feat/claude-team
```

## 2. 파이썬 패키지 설치
```bash
pip3 install slack_bolt slack_sdk python-dotenv
```

## 3. 환경변수 파일 생성
`.env.claude-team` 파일을 아래 내용으로 생성:

```
# Claude Team - Slack Configuration

# === Claude Team Lead ===
SLACK_TEAM_LEAD_BOT_TOKEN=xoxb-***MASKED***
SLACK_TEAM_LEAD_APP_TOKEN=xapp-***MASKED***

# === Claude Coder ===
SLACK_CODER_BOT_TOKEN=xoxb-***MASKED***
SLACK_CODER_APP_TOKEN=xapp-***MASKED***

# === Claude Reviewer ===
SLACK_REVIEWER_BOT_TOKEN=xoxb-***MASKED***
SLACK_REVIEWER_APP_TOKEN=xapp-***MASKED***

# === Claude Tester ===
SLACK_TESTER_BOT_TOKEN=xoxb-***MASKED***
SLACK_TESTER_APP_TOKEN=xapp-***MASKED***

# === Channel ===
SLACK_CHANNEL_TEAM=claude-team
```

## 4. 역할별 실행

### 맥미니 (coder)
```bash
cd ~/auto-email
python3 claude-team/bridge.py coder
```

### 스튜디오 (reviewer + tester)
터미널 2개에서:
```bash
cd ~/auto-email
python3 claude-team/bridge.py reviewer
```
```bash
cd ~/auto-email
python3 claude-team/bridge.py tester
```

## 5. 확인
Slack #claude-team 채널에서 해당 봇을 멘션하면 응답이 오면 성공.

## 머신별 역할
| 머신 | 역할 | 계정 |
|------|------|------|
| 맥북 | team-lead (opus) | Max |
| 맥미니 | coder (sonnet) | 엔터프라이즈 B |
| 스튜디오 | reviewer + tester (sonnet) | 엔터프라이즈 |