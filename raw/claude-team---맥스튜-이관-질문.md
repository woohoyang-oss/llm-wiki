# 맥스튜에서 브릿지 실행을 위해 필요한 정보

브릿지(bridge.py)가 Socket Mode를 사용하므로 APP_TOKEN (xapp-...) 이 필요합니다.
아카샤에 저장된 자격증명에는 BOT_TOKEN (xoxb-...) 만 있습니다.

## 필요한 토큰
1. Team Lead APP_TOKEN (xapp-...)
2. Project Leader APP_TOKEN (xapp-...) + BOT_TOKEN (xoxb-...)
   - project-leader용 Slack 앱이 별도로 있는지? 앱 이름은?

## 확인 방법
- 맥북의 .env.claude-team 파일에서 APP_TOKEN 값 확인
- 또는 Slack 앱 설정 (api.slack.com/apps) > Socket Mode > App-Level Tokens에서 확인

## 현재 상태
- 맥스튜: venv 생성, slack_bolt 설치 완료, bridge.py 준비됨
- BOT_TOKEN 5개 확보 (team-lead, coder, reviewer, tester + project-leader 미확인)
- APP_TOKEN 0개 — 이것 없이는 Socket Mode 브릿지 실행 불가