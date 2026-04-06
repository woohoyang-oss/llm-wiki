# bridge.py 상세 가이드

Claude Team Bible 부록. Slack ↔ Claude CLI 연결 프로그램.

## 개요
bridge.py는 각 Slack 봇이 멘션되면 Claude CLI를 호출하고 결과를 Slack에 전송하는 프로그램.
스레드 단위로 세션을 유지하여 대화 맥락이 이어짐.

## 사용법
```bash
python3 claude-team/bridge.py team-lead
python3 claude-team/bridge.py coder
python3 claude-team/bridge.py reviewer
python3 claude-team/bridge.py tester
python3 claude-team/bridge.py project-leader
```

## 에이전트 설정 (AGENT_CONFIG)
| 에이전트 | 모델 | 에이전트 파일 |
|----------|------|-------------|
| team-lead | sonnet | team-lead.md |
| project-leader | opus | project-leader.md |
| coder | sonnet | coder.md |
| reviewer | sonnet | reviewer.md |
| tester | sonnet | tester.md |

## 핵심 로직
1. 메시지 수신 (Slack Socket Mode)
2. Claude CLI 호출 (subprocess, --resume for session continuity)
3. 세션 관리 (thread_ts → session_id 매핑)
4. 응답 전송 (4000자 초과 시 분할)
5. 대시보드 연동 (옵션)

## Slack App 생성 가이드
- Bot Token Scopes: chat:write, channels:history, channels:read, users:read, app_mentions:read
- Socket Mode 활성화 + App-Level Token (connections:write)

## 아카샤 테스터 폴링 (대안 통신)
Slack 없이 아카샤만으로 운영하는 에이전트 패턴.
