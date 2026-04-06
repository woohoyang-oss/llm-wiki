# 맥미니 브릿지 운영 가이드

## 타임아웃 문제

bridge.py의 Claude CLI 호출 타임아웃이 300초(5분)입니다.
무거운 작업(CI 분석+수정+푸시, pytest+테스트 작성 등)은 5분을 초과할 수 있습니다.

### 수정 방법

bridge.py 153번째 줄:
```python
# 변경 전
timeout=300,

# 변경 후 (10분)
timeout=600,
```

파일: `claude-team/bridge.py` line 153

수정 후 coder, reviewer, tester 브릿지를 재시작하세요:
```bash
pkill -f "bridge.py coder"
pkill -f "bridge.py reviewer"
pkill -f "bridge.py tester"

cd ~/auto-email
python3 claude-team/bridge.py coder &
python3 claude-team/bridge.py reviewer &
python3 claude-team/bridge.py tester &
```

## PL 지시 규칙 (타임아웃 방지)

PL이 팀원에게 지시할 때 반드시 지켜야 할 규칙:
1. **한 지시당 1가지 작업만** — 여러 작업을 한 메시지에 넣지 않기
2. **5분 안에 끝나는 크기로** — 분석만, 수정만, 테스트만 분리
3. **나쁜 예**: "CI 분석 + 수정 + 커밋 + 푸시 + 보고" (5분 초과)
4. **좋은 예**:
   - 1차: "ruff check app/ 실행해서 에러 목록 보고"
   - 2차: "위 에러 수정해서 커밋+푸시"

## 현재 상태 확인 명령
```bash
# 프로세스 확인
ps aux | grep bridge.py

# 로그 확인 (포그라운드 실행)
python3 claude-team/bridge.py coder
```

## 브릿지 재시작 후 확인
Slack #claude-team에서 각 봇 멘션해서 응답 오는지 테스트:
- @Claude Coder 연결 확인
- @Claude Reviewer 연결 확인
- @Claude Tester 연결 확인
