# 커맨드센터 이전 명령어 가이드

## 새 세션 시작 시 실행 순서

### 1. 아카샤에서 이관 패키지 로드
```
/akaxa load Claude Team - 커맨드센터 이관 패키지
```
- 최신 날짜 것을 로드 (예: 2026-03-27 08:00)
- 현재 상태, 인프라 정보, 미완료 티켓, 규칙 등 전부 포함

### 2. 로컬 메모리 확인
- 경로: `~/.claude/projects/-Users-yangwooho/memory/`
- `MEMORY.md` — 메모리 인덱스 (피드백, 규칙 목록)
- `feedback_no_ask.md` — 판단 가능한 건 묻지 않고 바로 처리
- `feedback_github_only.md` — EC2 직접 수정 절대 금지
- `feedback_deploy_github_only.md` — 배포는 GitHub CI/CD만

### 3. 소스 동기 확인
```bash
cd /tmp/auto-email-fix && git fetch origin
# LOCAL, GITHUB(origin/main), EC2 커밋 해시 비교
# EC2 dirty 파일 0 확인
```

### 4. 폴링 시작
```
/loop 5m 슬랙 폴링 (PL_TOKEN, CHANNEL 포함)
/loop 3m 노션 폴링 (DB ID 포함, 1건씩)
```

### 5. 브릿지 확인 (맥미니)
```bash
sshpass -p 'tobe' ssh yangwooho@tobe-macmini "pgrep -f bridge.py | wc -l"
```

## 핵심 규칙
1. EC2 직접 수정 절대 금지 (git pull, docker build 포함)
2. 배포: 로컬 커밋 → GitHub push → CI/CD 자동 배포
3. 노션 티켓 한 번에 하나씩 (리뷰→코딩→테스트→배포→배포후테스트→완료→다음)
4. 판단 가능한 건 묻지 않고 바로 처리

## 아카샤 인증 (자동)
- email: wooho.yang@tbnws.com
- star_name: 포넥스
- PIN: 1245 (백업)

## 주요 아카샤 키
- `Claude Team - 커맨드센터 이관 패키지 (날짜)` — 세션 인수인계
- `Claude Team - Slack 전체 토큰` — 봇 토큰 + 채널
- `Claude Team - 원격 브릿지 관리 매뉴얼` — SSH 접속 절차
- `Claude Team Bible v2.0` — 클론 구축 매뉴얼

## Claude Code 스킬
- `/akaxa` — 아카샤 볼트 접근 (저장/로드/검색)