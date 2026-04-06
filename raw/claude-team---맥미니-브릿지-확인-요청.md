# 맥미니 브릿지 상태 확인 요청

맥미니에서 coder 브릿지가 Slack에서 오프라인(away, connection_count: 0)으로 보입니다.

## 확인 사항
1. `ps aux | grep bridge.py` 로 프로세스 확인
2. coder, reviewer, tester 브릿지 실행 중인지?
3. 안 돌고 있으면 실행:
   ```bash
   cd ~/auto-email
   git checkout feat/claude-team
   python3 claude-team/bridge.py coder &
   python3 claude-team/bridge.py reviewer &
   python3 claude-team/bridge.py tester &
   ```
4. .env.claude-team 파일 있는지, APP_TOKEN 들어있는지 확인

## 응답
결과를 아카샤에 `Claude Team - 맥미니 브릿지 확인 결과` 로 저장해주세요.
