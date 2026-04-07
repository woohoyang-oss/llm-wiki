# Email Support Bot — POC v0.1

AI 기반 이메일 자동 응대 시스템 (1차 상담원)

## 아키텍처

```
수신 메일                         AI 처리                        발송
───────────────────────────────────────────────────────────────────────
support@keychron.kr  ─┐
support@gtgear.co.kr ─┼→ ai@tbe.kr ─→ 룰엔진 ─→ Claude CLI ─→ 자동발송
b2b@keychron.kr      ─┘     │          (MCP 도구)    │        or 승인큐
                         Gmail API      │         confidence
                         Pub/Sub    패턴매칭       ≥0.85 → auto
                                                  0.70~0.84 → approval
                                                  <0.70 → escalation
```

## 기술 스택

- **Backend**: Python 3.12, FastAPI, SQLAlchemy (async), Celery
- **AI**: Claude CLI (Claude Code) + MCP 도구, 로컬 LLM (OpenAI-compat, Phase 2)
- **DB**: PostgreSQL 16 + pgvector
- **Cache/Queue**: Redis 7
- **Mail**: Gmail API (Google Workspace, Service Account)

## 프로젝트 구조

```
email-support-bot/
├── app/
│   ├── api/                # FastAPI 라우터
│   │   ├── admin.py        # Admin CRUD (룰, 메일박스, 엔진, 고지문, 승인)
│   │   ├── health.py       # 헬스 체크
│   │   └── webhook.py      # Gmail Pub/Sub 웹훅
│   ├── core/
│   │   ├── config.py       # 환경 설정 (.env)
│   │   └── database.py     # SQLAlchemy async engine
│   ├── models/
│   │   └── base.py         # 전체 ORM 모델 (11 테이블)
│   ├── services/
│   │   ├── ai_processor.py      # Claude CLI / 로컬 LLM 호출
│   │   ├── gmail_service.py     # Gmail API 연동 (수신/파싱/발송)
│   │   ├── mailbox_resolver.py  # 수신주소 → 브랜드 매핑
│   │   ├── notification.py      # 알림 서비스 (Slack/Telegram/Webhook)
│   │   ├── pipeline.py          # 메인 오케스트레이터
│   │   └── rule_engine.py       # 룰 매칭 엔진
│   ├── workers/
│   │   ├── celery_app.py   # Celery 설정
│   │   └── tasks.py        # 비동기 태스크
│   └── main.py             # FastAPI 앱 엔트리
├── alembic/                # DB 마이그레이션
├── scripts/
│   └── seed_data.py        # 초기 데이터
├── claude_mcp_config.json  # Claude CLI MCP 도구 설정
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

## 빠른 시작

### 1. 환경 준비

```bash
# 프로젝트 클론
git clone https://github.com/woohoyang-oss/auto-email.git
cd auto-email

# .env 설정
cp .env.example .env
# .env 파일 편집 — DB, Redis, Gmail, Google OAuth, TBNWS MySQL 등

# Google 인증 파일 배치
mkdir -p credentials
# credentials/service-account.json  — Google Workspace 서비스 계정 키
# credentials/gmail_token.json      — Gmail OAuth 토큰
# credentials/oauth_client.json     — Google OAuth 클라이언트 설정
```

### 2. 인프라 실행 (Docker)

```bash
# PostgreSQL + Redis 실행
docker compose up -d

# Python 의존성 설치
python -m venv .venv && source .venv/bin/activate
pip install -e .
pip install aiomysql  # TBNWS MySQL 연동 시 필요
```

### 3. DB 초기화 + Seed Data 임포트

```bash
# 테이블 생성 + seed-data/*.json 임포트
python -m scripts.seed_data

# 기존 데이터 초기화 후 재입력 (주의: 데이터 삭제됨)
python -m scripts.seed_data --reset
```

**seed-data/** 디렉토리에 포함된 초기 데이터:

| 파일 | 내용 |
|------|------|
| `knowledge_repositories.json` | 지식 레포지토리 (8개) |
| `knowledge_documents.json` | 지식 문서 (11개) |
| `rules.json` | 룰 엔진 규칙 (7개) |
| `mailbox_config.json` | 메일박스 설정 (4개) |
| `llm_engines.json` | LLM 엔진 (2개) |
| `notification_channels.json` | 알림 채널 (2개) |
| `ai_disclaimers.json` | AI 고지문 (1개) |
| `agents.json` | 상담원 (2개, 토큰 제거) |

> **참고**: `knowledge_repositories`의 filesystem 경로는 `knowledge/` 상대경로로 되어있습니다.
> 원본 환경에서 절대경로를 사용하는 경우 Admin UI 또는 DB에서 직접 수정하세요.

### 4. 서버 실행 (개발)

```bash
# API 서버 (포트 8000)
uvicorn app.main:app --reload --port 8000

# Admin 프론트엔드 (포트 3000, 별도 터미널)
cd admin && npm install && npm run dev

# Celery 워커 (별도 터미널)
celery -A app.workers.celery_app worker --loglevel=info -Q email_processing,email_polling

# Celery Beat (별도 터미널, 스케줄링)
celery -A app.workers.celery_app beat --loglevel=info
```

### 5. 접속

- **Admin 대시보드**: http://localhost:3000
- **API 서버**: http://localhost:8000
- **API 문서**: http://localhost:8000/docs
- Next.js가 `/api/*` 요청을 FastAPI(8000)로 프록시합니다

### 6. 전체 Docker 실행 (프로덕션)

```bash
docker compose up -d
```

## 환경별 차이점

| 항목 | 원본 (iCloud) | 로컬 (clone) |
|------|---------------|--------------|
| knowledge 경로 | iCloud 절대경로 | `knowledge/` 상대경로 |
| claude_mcp_config.json | iCloud 경로 | 프로젝트 내 경로 |
| tbnws-data MCP | `tbnws-ai` 프로젝트 필요 | 별도 설치 필요 |
| qwen LLM 엔진 | `wooho-pc:18789` | LLM 서버 별도 실행 필요 |

`claude_mcp_config.json`과 DB의 레포지토리 config 경로를 로컬 환경에 맞게 수정하세요.

## Gmail 설정

### Google Workspace 서비스 계정

1. Google Cloud Console → 서비스 계정 생성
2. Domain-wide delegation 활성화
3. Google Admin → API 제어 → 도메인 전체 위임에 서비스 계정 추가
4. 필요 스코프:
   - `https://www.googleapis.com/auth/gmail.readonly`
   - `https://www.googleapis.com/auth/gmail.send`
   - `https://www.googleapis.com/auth/gmail.modify`

### Gmail "Send As" 설정

각 브랜드 주소(support@keychron.kr 등)에서 발송하려면:
1. ai@tbe.kr Gmail 설정 → 계정 → "다른 주소에서 메일 보내기" 추가
2. 또는 Service Account에 해당 주소의 Send As 권한 부여

### Pub/Sub 설정 (실시간 알림)

1. Google Cloud Pub/Sub → 토픽 생성
2. Push 구독 생성 → 엔드포인트: `https://your-domain/webhook/gmail/push`
3. Gmail API `watch()` 호출로 모니터링 시작

## API 엔드포인트

### 헬스 체크
- `GET /health` — 서비스 상태

### 웹훅
- `POST /webhook/gmail/push` — Gmail Pub/Sub 알림 수신
- `POST /webhook/gmail/push/test` — 수동 메일 확인 트리거

### Admin API (Header: `X-Admin-Key: {secret}`)
- `GET /admin/dashboard` — 대시보드 요약
- `GET /admin/emails` — 수신 메일 목록
- `GET /admin/emails/{id}` — 메일 상세 (AI 응답 포함)
- CRUD: `/admin/rules`, `/admin/mailboxes`, `/admin/disclaimers`, `/admin/engines`
- `GET /admin/approvals` — 승인 대기 목록
- `POST /admin/approvals/{id}/action` — 승인/거부/수정

## Claude CLI 설정

### claude_mcp_config.json

Claude CLI가 사용할 MCP 도구를 정의합니다. 기본 설정:

- **email-support-db**: PostgreSQL 직접 조회 (주문, 고객 등)
- **knowledge-base**: MD 문서 검색 (FAQ, 정책)
- **keychron-ops / gtgear-ops**: 브랜드별 운영 데이터 조회

### Claude CLI 동작 방식

```
수신 메일 → 시스템 프롬프트 + 메일 내용 →
    claude -p "메시지" --model sonnet --output-format json --system-prompt "..." --mcp-config ./claude_mcp_config.json
→ Claude가 MCP 도구로 DB 조회, 지식 검색 → JSON 응답 반환
```

## 룰 조건 형식

```json
{
  "operator": "AND|OR",
  "rules": [
    {"field": "subject|body|from|original_to|subject_or_body",
     "match": "contains|exact|regex|starts_with|domain|not_contains",
     "value": ["값1", "값2"]}
  ]
}
```

## 룰 액션 형식

```json
{
  "action": "auto_reply|draft_and_wait|tag_only|forward|escalate",
  "category": "cs_delivery|cs_exchange|b2b|as_repair|spam",
  "engine_id": "claude-cli",
  "forward_to": "someone@tbe.kr",
  "tags": ["tag1"]
}
```

## Phase 2 (예정)

- 로컬 LLM 통합 (vLLM/Ollama, `http://x.x.x.x:18787/v1`)
- RAG 파이프라인 (pgvector + MD 문서 임베딩)
- Next.js Admin 대시보드
- Slack/Telegram 알림 연동
- 다국어 고지문 관리
- 메일 템플릿 엔진
