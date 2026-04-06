# 로컬 경로 및 환경 정보

## 머신별 경로

### 맥북 (커맨드센터)
- 홈: /Users/yangwooho
- 레포 클론: /tmp/auto-email-fix (git clone으로 생성, 세션마다 확인 필요)
- SSH 키: ~/.ssh/auto-email-poc.pem (EC2 접속용)
- Claude 메모리: ~/.claude/projects/-Users-yangwooho/memory/
- Claude 스킬: ~/.claude/skills/akaxa/SKILL.md
- sshpass: /opt/homebrew/bin/sshpass

### 맥미니 (팀멤버 브릿지)
- 호스트: yangwooho@tobe-macmini (pw: tobe)
- 레포: /Users/yangwooho/auto-email
- 브릿지: /Users/yangwooho/claude-team/bridge.py
- 브릿지 환경변수: /Users/yangwooho/claude-team/.env.claude-team
- 브릿지 로그: /tmp/bridge_coder.log, /tmp/bridge_reviewer.log, /tmp/bridge_tester.log
- 에이전트 프롬프트: /Users/yangwooho/auto-email/.claude/agents/coder.md 등
- CLAUDE.md: /Users/yangwooho/.claude/CLAUDE.md (EC2 접속 금지 규칙)

### 맥북 (TL/PL 브릿지 — 현재 미사용)
- 호스트: woohoyang@wooho-macbookpro-14 (pw: 56terces)

### EC2 (프로덕션 서버)
- 호스트: ec2-user@3.34.66.166
- SSH: ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166
- 레포: /home/ec2-user/auto-email
- Docker: esbot-admin, esbot-api, esbot-beat, esbot-worker, esbot-db, esbot-redis
- nginx: /etc/nginx/conf.d/mail.tbe.kr.conf
- uploads: /app/uploads/chat/ (API 컨테이너 내부)

## GitHub
- URL: https://github.com/woohoyang-oss/auto-email
- PAT: ghp_***MASKED***
- CI/CD: .github/workflows/ci.yml + deploy.yml

## 노션
- DB ID: d3f5daa7d3a848398543da524a120d54
- Data Source: collection://a0b8418e-81fa-430c-842d-c30fd06eefd4
- Status: "시작 전" / "진행 중" / "완료" (띄어쓰기 주의)
- Priority: Critical / High / Medium / Low

## Slack
- 채널: C0AN03UNL0M (#claude-team)
- PL 토큰: xoxb-***MASKED***
- 사용자 ID: U0AN1CSS8DC
- TL: U0AN3FEMS6N / Coder: U0ANXPNJWAC / Reviewer: U0AMN2KKML7 / Tester: U0AN73VFGSY