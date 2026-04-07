# Email Support Bot — 운영 가이드

## 서비스 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│  클라이언트 (브라우저)                                   │
│  http://localhost:3000  (Next.js Admin)                  │
└───────────────┬─────────────────────────────────────────┘
                │ API 호출 (/api → proxy → :8000)
┌───────────────▼─────────────────────────────────────────┐
│  FastAPI Backend (:8000)                                 │
│  - REST API (admin.py 단일 파일)                        │
│  - Gmail Webhook 수신                                    │
│  - 텔레그램 봇 Long Polling (자동 시작)                  │
└──┬──────────────┬──────────────┬────────────────────────┘
   │              │              │
   ▼              ▼              ▼
┌──────┐   ┌──────────┐   ┌──────────────┐
│ PG16 │   │ Redis 7  │   │ Celery       │
│:5432 │   │ :6379    │   │ Worker+Beat  │
└──────┘   └──────────┘   └──────────────┘
                                │
                          ┌─────▼─────┐
                          │ Gmail API │
                          │ Pub/Sub   │
                          └───────────┘
```

## 필수 서비스 목록

| 서비스 | 포트 | 역할 | 필수여부 |
|--------|------|------|----------|
| PostgreSQL 16 + pgvector | 5432 | 메인 DB | 필수 |
| Redis 7 | 6379 | Celery 브로커, 캐시 | 필수 |
| FastAPI (uvicorn) | 8000 | REST API + 텔레그램 polling | 필수 |
| Next.js (admin) | 3000 | 관리자 웹 UI | 필수 |
| Celery Worker | - | 비동기 이메일 처리 | 필수 |
| Celery Beat | - | 스케줄러 (Gmail 폴링 등) | 선택 (자동화 시) |
| Ollama (Mac Studio) | 11434 | 챗봇 LLM (Qwen3 14B) | 선택 (챗봇용) |

## 로컬 개발 환경 시작 순서

### 1단계: 인프라 (Docker)

```bash
# Colima 시작 (macOS)
colima start

# PostgreSQL + Redis 시작
docker compose up -d db redis

# 상태 확인
docker compose ps
docker exec esbot-db pg_isready -U bot -d email_support_bot
docker exec esbot-redis redis-cli ping
```

### 2단계: Backend

```bash
cd /Users/woohoyang/Downloads/claude/auto-email

# 가상환경 활성화
source .venv/bin/activate

# DB 마이그레이션 (테이블 자동 생성은 앱 시작 시 수행됨)
# 수동 마이그레이션 필요 시:
# alembic upgrade head

# FastAPI 서버 시작
uvicorn app.main:app --reload --port 8000
```

> 서버 시작 시 자동으로 수행되는 작업:
> - DB 테이블 자동 생성 (`Base.metadata.create_all`)
> - 텔레그램 봇 Long Polling 시작 (token 설정 시)

### 3단계: Celery (비동기 처리)

```bash
# Worker (이메일 처리 큐)
celery -A app.workers.celery_app worker \
  --loglevel=info \
  -Q email_processing,email_polling

# Beat (스케줄러) - 별도 터미널
celery -A app.workers.celery_app beat --loglevel=info
```

### 4단계: Frontend

```bash
cd admin
npm install   # 최초 또는 의존성 변경 시
npm run dev
```

### 5단계: (선택) Ollama — 챗봇 LLM

```bash
# Mac Studio에서 실행
ollama serve
ollama run qwen3:14b   # 최초 다운로드
```

## 환경변수 (.env) 필수 항목

```bash
# ── 필수 ──
DATABASE_URL=postgresql+asyncpg://bot:bot_password@localhost:5432/email_support_bot
DATABASE_URL_SYNC=postgresql://bot:bot_password@localhost:5432/email_support_bot
REDIS_URL=redis://localhost:6379/0
ADMIN_SECRET_KEY=<변경필수>
JWT_SECRET_KEY=<변경필수-32자이상>

# ── Gmail API ──
GMAIL_AUTH_MODE=oauth
GMAIL_OAUTH_TOKEN_PATH=./credentials/gmail_token.json
GMAIL_DELEGATED_USER=ai.login@tbnws.com
GOOGLE_CLOUD_PROJECT_ID=tbe-email-bot

# ── Google OAuth (SSO) ──
GOOGLE_OAUTH_CLIENT_ID=<Google Cloud Console에서 발급>
GOOGLE_OAUTH_CLIENT_SECRET=<Google Cloud Console에서 발급>
GOOGLE_OAUTH_REDIRECT_URI=http://localhost:3000/api/auth/google/callback

# ── 텔레그램 ──
TELEGRAM_BOT_TOKEN=<BotFather에서 발급>
TELEGRAM_BOT_USERNAME=<봇 username>
TELEGRAM_CHAT_ID=<관리자 chat_id>

# ── 챗봇 LLM ──
OLLAMA_BASE_URL=http://dev-macstudio:11434
CHATBOT_MODEL=qwen3:14b

# ── Pipeline ──
DRY_RUN=true                              # POC: 자동발송 차단
DRY_RUN_ALLOWED_RECIPIENTS=wooho.yang@tbnws.com  # 테스트 수신자

# ── CORS ──
ADMIN_ALLOWED_ORIGINS=http://localhost:3000
```

## 운영 서버 마이그레이션 체크리스트

### 환경 설정

- [ ] `.env` 파일 생성 (위 필수 항목 모두 설정)
- [ ] `ADMIN_SECRET_KEY` 강력한 랜덤 문자열로 변경
- [ ] `JWT_SECRET_KEY` 32자 이상 랜덤 문자열로 변경
- [ ] `APP_ENV=production`, `APP_DEBUG=false`, `LOG_LEVEL=INFO`
- [ ] `DRY_RUN=true` 유지 (POC 완료 전까지 절대 false 금지)
- [ ] `ADMIN_ALLOWED_ORIGINS`를 운영 도메인으로 변경
- [ ] `GOOGLE_OAUTH_REDIRECT_URI`를 운영 도메인으로 변경

### 인프라

- [ ] PostgreSQL 16 + pgvector 확장 설치 확인
- [ ] Redis 7 설치 확인
- [ ] Docker Compose로 DB/Redis 시작 또는 별도 서버 사용
- [ ] DB 볼륨 백업 설정

### Gmail API

- [ ] `credentials/gmail_token.json` 배포 (OAuth 토큰)
- [ ] 또는 Service Account 방식으로 전환 (`GMAIL_AUTH_MODE=service_account`)
- [ ] Gmail Pub/Sub 설정 (실시간 수신 필요 시)

### 텔레그램

- [ ] `TELEGRAM_BOT_TOKEN` 설정
- [ ] `TELEGRAM_BOT_USERNAME` 설정
- [ ] 텔레그램 봇에 `/start` 연결 확인

### Celery

- [ ] Worker 프로세스 데몬화 (systemd 또는 supervisor)
- [ ] Beat 프로세스 데몬화
- [ ] 동시성 설정: `--concurrency=4` (CPU 코어 수에 맞게)

### 보안

- [ ] HTTPS 설정 (nginx/caddy 리버스 프록시)
- [ ] `.env` 파일 권한 제한 (`chmod 600 .env`)
- [ ] `credentials/` 디렉터리 권한 제한
- [ ] 방화벽: 8000 포트 직접 노출 금지 (리버스 프록시만 허용)
- [ ] CORS origin 운영 도메인만 허용

### Frontend

- [ ] `admin/.env.production` 생성: `NEXT_PUBLIC_API_URL=https://<운영도메인>/api`
- [ ] `npm run build` 후 `npm start` (또는 PM2)
- [ ] 또는 nginx에서 정적 파일 서빙

## Docker Compose 배포 (전체)

```bash
# 전체 서비스 시작
docker compose up -d

# 특정 서비스만 재시작
docker compose restart api
docker compose restart worker

# 로그 확인
docker compose logs -f api
docker compose logs -f worker
```

## 프로세스 관리 (비-Docker)

### systemd 서비스 예시

`/etc/systemd/system/esbot-api.service`:
```ini
[Unit]
Description=Email Support Bot API
After=postgresql.service redis.service
Requires=postgresql.service redis.service

[Service]
Type=simple
User=esbot
WorkingDirectory=/opt/email-support-bot
EnvironmentFile=/opt/email-support-bot/.env
ExecStart=/opt/email-support-bot/.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/esbot-worker.service`:
```ini
[Unit]
Description=Email Support Bot Celery Worker
After=postgresql.service redis.service

[Service]
Type=simple
User=esbot
WorkingDirectory=/opt/email-support-bot
EnvironmentFile=/opt/email-support-bot/.env
ExecStart=/opt/email-support-bot/.venv/bin/celery -A app.workers.celery_app worker --loglevel=info --concurrency=4 -Q email_processing,email_polling
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/esbot-beat.service`:
```ini
[Unit]
Description=Email Support Bot Celery Beat
After=postgresql.service redis.service

[Service]
Type=simple
User=esbot
WorkingDirectory=/opt/email-support-bot
EnvironmentFile=/opt/email-support-bot/.env
ExecStart=/opt/email-support-bot/.venv/bin/celery -A app.workers.celery_app beat --loglevel=info
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable esbot-api esbot-worker esbot-beat
sudo systemctl start esbot-api esbot-worker esbot-beat
```

## 헬스체크

```bash
# API 서버
curl http://localhost:8000/health

# DB 연결
docker exec esbot-db pg_isready -U bot -d email_support_bot

# Redis
docker exec esbot-redis redis-cli ping

# Celery Worker 상태
curl http://localhost:8000/admin/batch/workers -H "X-Admin-Key: admin"

# 텔레그램 봇
curl "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe"
```

## 트러블슈팅

### 포트 충돌
```bash
# 포트 사용 중인 프로세스 찾기
lsof -i :8000
lsof -i :3000
# 강제 종료
kill -9 $(lsof -ti:8000)
```

### DB 연결 실패
```bash
# Docker 상태 확인
docker ps
# Colima 상태 (macOS)
colima status
# Colima 재시작
colima start
docker compose up -d db redis
```

### Celery Worker 연결 안 됨
```bash
# Redis 연결 확인
redis-cli -h localhost -p 6379 ping
# Worker 수동 시작
celery -A app.workers.celery_app worker --loglevel=debug
```

### 텔레그램 봇 응답 없음
- FastAPI 서버가 실행 중인지 확인 (polling은 서버 시작 시 자동)
- `TELEGRAM_BOT_TOKEN` 설정 확인
- 로그 확인: `telegram_polling_started` 메시지 있는지

### 챗봇 LLM 응답 없음
- Ollama 서버 상태: `curl http://dev-macstudio:11434/api/tags`
- 모델 설치 확인: `ollama list` (qwen3:14b 있어야 함)
- 네트워크: Mac Studio에 접근 가능한지 ping 확인

## 중요 주의사항

1. **DRY_RUN=true 유지**: POC 단계에서 `DRY_RUN=false`로 변경하면 실제 고객에게 AI 답변이 자동 발송됩니다. **절대 변경 금지**.
2. **admin.py 단일 파일**: `app/api/admin.py`는 분할하지 않습니다. 신규 엔드포인트는 파일 끝에 추가.
3. **credentials 보안**: `credentials/` 폴더의 Gmail OAuth 토큰, 서비스 계정 키는 절대 git에 커밋하지 않습니다.
4. **DB 백업**: 운영 전 PostgreSQL 자동 백업 설정 필수.
