# 서버 배포 가이드

## 서버 정보

| 항목 | 값 |
|------|------|
| 서버 | AWS EC2 t3.small (ap-northeast-2, 서울) |
| Elastic IP | 3.34.66.166 |
| 도메인 | mail.tbe.kr → 3.34.66.166 |
| OS | Amazon Linux 2023 |
| 디스크 | 50GB (gp3) |
| 메모리 | 2GB |
| PEM 키 | `~/.ssh/auto-email-poc.pem` |

## SSH 접속

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166
```

## 아키텍처

```
                    ┌─────────────────────────────────────┐
                    │        EC2 (3.34.66.166)            │
                    │                                     │
  :80 ─────────────▶│  esbot-admin  (Next.js standalone)  │
  :8000 ───────────▶│  esbot-api    (FastAPI + uvicorn)   │
                    │  esbot-worker (Celery Worker)       │
                    │  esbot-beat   (Celery Beat)         │
                    │  esbot-db     (PostgreSQL 16)       │
                    │  esbot-redis  (Redis 7)             │
                    └─────────────────────────────────────┘
```

## 컨테이너 구성 (Docker Compose)

| 컨테이너 | 이미지 | 포트 | 역할 |
|-----------|--------|------|------|
| esbot-db | pgvector/pgvector:pg16 | 5432 | PostgreSQL + pgvector |
| esbot-redis | redis:7-alpine | 6379 | Redis 캐시/큐 |
| esbot-api | auto-email-api (빌드) | 8000 | FastAPI 백엔드 |
| esbot-worker | auto-email-worker (빌드) | - | Celery 비동기 작업 |
| esbot-beat | auto-email-beat (빌드) | - | Celery 스케줄러 |
| esbot-admin | auto-email-admin (빌드) | 80→3000 | Next.js 프론트엔드 |

## 배포 명령어

### 백엔드만 배포 (코드 변경)

`app/` 디렉토리가 볼륨 마운트 + `--reload` 모드이므로 rsync만 하면 자동 반영됨.

```bash
rsync -avz --exclude '__pycache__' \
  -e "ssh -i ~/.ssh/auto-email-poc.pem" \
  app/ ec2-user@3.34.66.166:~/auto-email/app/
```

### 백엔드 재시작 (설정 변경, .env 수정 등)

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose restart api"
```

### 프론트엔드 배포 (빌드 필요)

Next.js는 standalone 빌드이므로 소스 전송 → 빌드 → 재시작 필요.

```bash
# 1. 소스 전송
rsync -avz --exclude 'node_modules' --exclude '.next' \
  -e "ssh -i ~/.ssh/auto-email-poc.pem" \
  admin/ ec2-user@3.34.66.166:~/auto-email/admin/

# 2. 빌드 + 재시작
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose build admin && docker-compose up -d admin"
```

### 전체 재시작

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose restart"
```

### 전체 재빌드 + 재시작

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose build && docker-compose up -d"
```

### Worker/Beat만 재시작

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose restart worker beat"
```

## 주요 경로 (서버)

| 경로 | 용도 |
|------|------|
| `~/auto-email/` | 프로젝트 루트 |
| `~/auto-email/.env` | 환경변수 (DB, API 키 등) |
| `~/auto-email/app/` | 백엔드 소스 (볼륨 마운트) |
| `~/auto-email/admin/` | 프론트엔드 소스 |
| `~/auto-email/credentials/` | OAuth 토큰 (읽기/쓰기 마운트) |
| `~/auto-email/knowledge/` | 지식베이스 파일 |

## 로그 확인

```bash
# API 서버 로그
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker logs esbot-api --tail 50"

# Worker 로그
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker logs esbot-worker --tail 50"

# 실시간 로그 (follow)
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker logs -f esbot-api"

# 전체 컨테이너 상태
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"
```

## DB 접속

```bash
# psql 직접 접속
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker exec -it esbot-db psql -U bot -d email_support_bot"

# 쿼리 실행
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker exec -i esbot-db psql -U bot -d email_support_bot -c 'SELECT count(*) FROM emails;'"
```

## .env 수정

```bash
# 편집
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "nano ~/auto-email/.env"

# 수정 후 반영 (API 재시작)
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && docker-compose restart api worker beat"
```

## 트러블슈팅

### 디스크 부족
```bash
# Docker 미사용 이미지/캐시 정리
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker system prune -f"
```

### 메모리 부족
```bash
# 메모리 확인
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "free -h && docker stats --no-stream"
```

### 컨테이너 크래시
```bash
# 상태 + 최근 재시작 횟수 확인
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker ps -a --format 'table {{.Names}}\t{{.Status}}'"

# 크래시 로그
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "docker logs esbot-api --tail 100"
```
