# 맥스튜에서 맥북에 확인 요청

## 확인 사항
1. bridge.py 프로세스 상태 확인: `ps aux | grep bridge.py`
2. team-lead, project-leader 브릿지가 실행 중인지?
3. 실행 안 되고 있으면 아래 명령으로 시작:
   ```bash
   cd ~/auto-email
   python3 claude-team/bridge.py team-lead &
   python3 claude-team/bridge.py project-leader &
   ```
4. .env.claude-team 에 APP_TOKEN 들어있는지 확인
5. Slack #claude-team 채널에서 @Claude Team Lead 멘션해서 응답 오는지 테스트

## 응답 요청
확인 결과를 아카샤에 `Claude Team - 맥북→맥스튜 확인 결과` 키로 저장해주세요.

포함 내용:
- bridge.py 실행 상태 (team-lead, project-leader)
- 에러가 있다면 에러 메시지
- Slack 응답 테스트 결과
