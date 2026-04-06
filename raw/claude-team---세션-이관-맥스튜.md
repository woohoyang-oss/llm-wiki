# Claude Team 세션 이관 — 맥스튜에서 이어서 작업

## 현재 상황
- 브랜치: feat/claude-team (GitHub 푸시 완료)
- 프로젝트: /Users/woohoyang/auto-email (또는 ~/auto-email)
- 맥미니에서 coder/reviewer/tester 브릿지 실행 중
- 대시보드: claude-team/dashboard/server.py (localhost:5555)

## 즉시 해야 할 일: 배포 차단 사유 3건 수정

reviewer가 배포 불가(🔴) 판정. 아래 3건 수정 후 재리뷰 → main 머지 → 배포

### 1. Gmail 토큰 마이그레이션 스크립트 작성
- 파일: scripts/migrate_gmail_tokens.py (신규)
- 내용: agents 테이블의 기존 평문 gmail_access_token, gmail_refresh_token을 Fernet으로 암호화 변환
- .env.example에 ALLOW_PLAINTEXT_FALLBACK=true 추가

### 2. edit_and_send 발송 후 email.status 업데이트
- 파일: app/api/admin.py
- 위치: edit_and_send 액션 핸들러
- 수정: 발송 성공 후 email.status = "replied" 추가

### 3. 벡터 검색 SQL 인젝션 수정
- 파일: app/api/admin.py
- 위치: POST /knowledge-docs/search 엔드포인트
- 수정: f-string SQL → pgvector 바인드 파라미터 방식으로 변경

## 수정 후 절차
1. coder에게 위 3건 수정 지시
2. reviewer에게 재리뷰 요청
3. 🟢 배포 가능 판정 받으면
4. EC2 스냅샷 생성: aws ec2 create-snapshot --volume-id vol-0dff8e880ea6b9d45
5. main에 머지 (PR 또는 직접)
6. GitHub Actions 자동 배포 확인
7. mail.tbe.kr 접속 확인

## 브릿지 실행 (맥스튜에서)
```bash
cd ~/auto-email
git clone https://github.com/woohoyang-oss/auto-email.git  # 또는 git pull
git checkout feat/claude-team
pip3 install slack_bolt slack_sdk python-dotenv
# .env.claude-team 생성 (아카샤 'Claude Team - Slack App Credentials' 참조)
python3 claude-team/bridge.py team-lead &
python3 claude-team/bridge.py project-leader &
python3 claude-team/dashboard/server.py &
```

## Slack 봇 현황
| 봇 | 머신 | 상태 |
|-----|------|------|
| team-lead | 맥북→맥스튜로 이관 | 재시작 필요 |
| project-leader | 맥북→맥스튜로 이관 | 재시작 필요 |
| coder | 맥미니 | ✅ 실행 중 |
| reviewer | 맥미니 | ✅ 실행 중 |
| tester | 맥미니 | ✅ 실행 중 |

## 완료된 과제: 27건 (상세는 'Claude Team - 구축 진행상황' 참조)
## 미완료 과제: 배포 차단 3건 + 태그룰 + 견적분석 + 모바일UI + 모니터링