# Claude Team 머신별 역할 배치

| 머신 | 역할 | 모델 |
|------|------|------|
| 맥북 | team-lead (TL), project-leader (PL) | sonnet, opus |
| 맥미니 | coder, reviewer, tester | sonnet |
| 맥스튜 | 개발 작업 전용 (브릿지 안 돌림) | - |

## 맥북 실행 명령
```bash
cd ~/auto-email
python3 claude-team/bridge.py team-lead &
python3 claude-team/bridge.py project-leader &
```

## 맥미니 실행 명령
```bash
cd ~/auto-email
python3 claude-team/bridge.py coder &
python3 claude-team/bridge.py reviewer &
python3 claude-team/bridge.py tester &
```
