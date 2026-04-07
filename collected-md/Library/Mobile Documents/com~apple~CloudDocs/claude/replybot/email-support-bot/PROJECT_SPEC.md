# Email Support Bot — 프로젝트 기획서 및 구현 현황

> AI 기반 이메일 자동 응대 시스템
> 최종 업데이트: 2026-03-13

---

## 1. 프로젝트 개요

### 1.1 목적
고객 지원 이메일을 AI(Claude)가 자동 분석하여 답변 초안을 생성하고, 신뢰도에 따라 자동 발송하거나 상담원의 승인 후 발송하는 시스템.

### 1.2 핵심 기능
- Gmail 수신 이메일 실시간 감지 (Pub/Sub + Polling)
- Claude CLI를 통한 AI 답변 생성 (MCP 도구 연동)
- 신뢰도 기반 자동 라우팅 (자동발송 / 승인대기 / 수동처리)
- 멀티 브랜드 지원 (keychron.kr, gtgear.co.kr 등)
- 규칙 엔진 기반 자동 태깅 및 분류
- 어드민 대시보드 (Next.js)

### 1.3 기술 스택

| 계층 | 기술 |
|------|------|
| **백엔드** | Python 3.12, FastAPI, SQLAlchemy (async), Celery |
| **DB** | PostgreSQL 16 + pgvector |
| **캐시/큐** | Redis 7 |
| **AI/LLM** | Claude CLI (Claude Code) + MCP |
| **프론트엔드** | Next.js 14+ (App Router), React 19, TypeScript |
| **UI** | Tailwind CSS, shadcn/ui, Lucide Icons, Recharts |
| **이메일** | Gmail API (OAuth 2.0 + Service Account) |
| **인프라** | Docker, Alembic, Structlog |

---

## 2. 아키텍처

### 2.1 시스템 구성도

```
[Gmail] ──Pub/Sub──▶ [Webhook /webhook/gmail/push]
                         │
                         ▼
                   [Celery Worker]
                         │
                    ┌─────┴─────┐
                    ▼           ▼
            [Rule Engine]  [MailboxResolver]
                    │           │
                    ▼           ▼
            [AI Processor]  (brand/category)
            (Claude CLI + MCP)
                    │
              ┌─────┼─────┐
              ▼     ▼     ▼
         ≥0.85   0.70   <0.70
        자동발송  승인대기  수동처리
              │     │     │
              ▼     ▼     ▼
          [Gmail Send] [Approval Queue] [Agent Dashboard]
```

### 2.2 서버 구성

| 서비스 | 포트 | 설명 |
|--------|------|------|
| FastAPI | 3000 | 백엔드 API |
| Next.js Admin | 3001 | 어드민 프론트엔드 |
| PostgreSQL | 5432 | 메인 DB |
| Redis | 6379 | Celery 브로커 + 캐시 |

### 2.3 인증 체계

- **Google SSO (OAuth 2.0)** — 어드민 로그인
- **JWT 토큰** — API 인증 (`Authorization: Bearer {jwt}`)
- **Service Account** — 공용 메일함 Gmail API 접근
- **개인 OAuth** — 개인 메일함 Gmail API 접근

---

## 3. 데이터 모델 (23개 테이블)

### 3.1 핵심 테이블

#### Agent (상담원)
```
id, email, name, role(admin/agent/viewer)
gmail_access_token, gmail_refresh_token, gmail_token_expiry
google_id, avatar_url, is_active
```

#### Email (이메일)
```
id, message_id, thread_id, mailbox_id
from_address, to_address, original_to, subject, body_text, body_html
status(new/open/pending/solved/closed/error)
priority(low/normal/high/urgent)
brand, category, tags[], assigned_to
ai_response_id, rule_id, sla_breached
```

#### AIResponse (AI 응답 초안)
```
id, email_id, engine_id
draft_subject, draft_body, confidence(0.0~1.0)
tools_used, reasoning, processing_time_ms
```

#### ApprovalQueue (승인 대기열)
```
id, email_id, ai_response_id
status(pending/approved/rejected/expired)
edited_body, approved_by, approved_at, rejection_reason
```

#### Rule (자동화 규칙)
```
id, name, description, priority
conditions[{field, match, value, operator}]
actions{action_type, auto_reply_engine_id, tags[], ...}
is_active
```

### 3.2 설정 테이블

| 테이블 | 용도 |
|--------|------|
| MailboxConfig | 브랜드별 메일함 설정 (주소, 서명, CC 등) |
| LLMEngine | AI 모델 레지스트리 (Claude CLI, 로컬 LLM) |
| EmailSignature | 이메일 서명 템플릿 |
| AIDisclaimer | AI 답변 면책 문구 |
| EmailTemplate | 카테고리별 답변 템플릿 |
| CannedResponse | 매크로 (단축 응답) |
| SLAPolicy | 우선순위별 응답/해결 목표 시간 |
| BusinessHours | 업무 시간 설정 |
| Automation | 시간/이벤트 기반 자동화 |
| KnowledgeRepository | 지식 데이터 소스 설정 |
| KnowledgeDocument | FAQ, 절차서, 정책 문서 |
| KnowledgeEmbedding | pgvector 임베딩 (1024차원) |
| NotificationChannel | 알림 채널 (Slack/Telegram/Webhook) |
| NotificationRouting | 이벤트→채널 라우팅 |
| SavedView | 커스텀 이메일 필터 뷰 |

---

## 4. AI 처리 파이프라인

### 4.1 처리 흐름
1. **이메일 수신** — Gmail Pub/Sub 또는 주기적 폴링
2. **파싱** — MIME 디코딩, 헤더 추출, 첨부파일 처리
3. **메일함 매칭** — `original_to` 주소로 브랜드/카테고리 결정
4. **자동 태깅** — 키워드 기반 (배송, 교환, 반품, AS, 주문, 불량 등)
5. **규칙 매칭** — 우선순위 기반, 첫 번째 매칭 규칙 적용
6. **AI 추론** — Claude CLI + MCP 도구로 답변 초안 생성
7. **신뢰도 라우팅**:
   - **≥ 0.85**: 자동 발송
   - **0.70 ~ 0.84**: 승인 대기열
   - **< 0.70**: 수동 처리 (에스컬레이션)
8. **알림 발송** — 설정된 채널로 이벤트 알림
9. **활동 로그** — 모든 액션 감사 추적

### 4.2 Claude CLI 통합
- subprocess로 `claude` CLI 바이너리 호출
- 동적 MCP 설정 빌더 (규칙 태그 기반 지식 저장소 선택)
- JSON 출력: `{draft_body, draft_subject, confidence, tools_used, reasoning}`
- 지원 MCP 저장소 유형: MySQL, PostgreSQL, Filesystem, Custom

### 4.3 규칙 엔진
- **조건 필드**: subject, body, from, original_to, subject_or_body
- **매칭 연산자**: contains, exact, regex, starts_with, domain, not_contains
- **조건 결합**: AND / OR
- **액션 유형**:
  - `auto_reply` — AI 답변 자동 발송
  - `draft_and_wait` — AI 답변 초안 후 승인 대기
  - `tag_only` — 태깅만 (답변 없음)
  - `forward` — 다른 상담원에게 전달
  - `escalate` — 수동 처리 에스컬레이션

---

## 5. 어드민 대시보드 (23개 페이지)

### 5.1 주요 페이지

#### 대시보드 (`/dashboard`)
- 실시간 KPI 카드 (총 이메일, 미답변, 답변완료, 미배정)
- 상태 분포 바 (new/open/pending/solved/closed)
- 7일 평균 응답시간
- SLA 준수율
- 상담원 워크로드 Top 5
- 14일 볼륨 트렌드 차트

#### 이메일 목록 (`/emails`)
- 검색, 필터 (상태/브랜드/태그/우선순위/담당자), 정렬
- 벌크 액션 (상태 변경, 배정, 태그)
- 저장된 뷰 관리
- 스레드 병합, 스누즈
- 매크로 빠른 삽입

#### 이메일 상세 (`/emails/[id]`)
- 스레드 대화 뷰
- AI 답변 초안 + 신뢰도 표시
- 승인/거절 워크플로우
- 고객 정보 패널 (TBNWS CRM 조회)
- 내부 노트
- 활동 히스토리
- 수동 답장 에디터
- 전달 기능

#### 승인 대기열 (`/approvals`)
- 대기/승인/거절 탭
- 신뢰도 필터 (높음/중간/낮음)
- 벌크 승인/거절
- 발송 전 초안 편집

### 5.2 설정 페이지

| 페이지 | 경로 | 기능 |
|--------|------|------|
| 규칙 관리 | `/rules` | 자동화 규칙 CRUD, 조건 빌더, 우선순위 |
| 상담원 관리 | `/agents` | 팀원 CRUD, 역할, 그룹 |
| 메일함 설정 | `/mailboxes` | 브랜드별 메일함, 서명, CC/BCC |
| AI 엔진 | `/engines` | LLM 모델 레지스트리, 상태 모니터링 |
| 서명 관리 | `/signatures` | HTML/텍스트 서명 템플릿 |
| 면책 문구 | `/disclaimers` | AI 답변 면책 문구 |
| 답변 템플릿 | `/email-templates` | 카테고리/브랜드별 템플릿 |
| 매크로 | `/macros` | 단축 응답 (캔드 리스폰스) |
| 지식 문서 | `/knowledge-docs` | FAQ, 절차서, 정책 문서 |
| 저장소 설정 | `/repositories` | 지식 데이터 소스 (DB, 파일, MCP) |
| SLA 정책 | `/sla-policies` | 응답/해결 시간 목표 |
| 자동화 | `/automations` | 시간/이벤트 트리거 워크플로우 |
| 업무 시간 | `/business-hours` | 주간 스케줄, 공휴일 |
| 알림 설정 | `/notifications` | Slack/Telegram/Webhook 채널 |
| 통계 | `/statistics` | 분석 대시보드 |
| 시스템 | `/system` | 환경 상태, DB 연결 |
| 내 메일함 | `/my-inbox` | 개인 Gmail 연동 |

---

## 6. API 엔드포인트 목록

### 6.1 인증 (`/auth`)
| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/auth/google/login` | Google OAuth 리다이렉트 |
| GET | `/auth/google/callback` | OAuth 콜백, JWT 발급 |
| GET | `/auth/me` | 현재 사용자 정보 |

### 6.2 웹훅 (`/webhook`)
| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/webhook/gmail/push` | Gmail Pub/Sub 알림 수신 |
| POST | `/webhook/gmail/push/test` | 수동 이메일 체크 |

### 6.3 어드민 (`/admin`)
| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/admin/dashboard` | 대시보드 통계 |
| GET | `/admin/emails` | 이메일 목록 (필터/정렬/페이지네이션) |
| GET | `/admin/emails/{id}` | 이메일 상세 |
| POST | `/admin/emails/{id}/send-reply` | 답장 발송 |
| POST | `/admin/emails/{id}/assign` | 담당자 배정 |
| POST | `/admin/emails/{id}/status` | 상태 변경 |
| POST | `/admin/emails/{id}/priority` | 우선순위 변경 |
| POST | `/admin/emails/{id}/tags` | 태그 변경 |
| POST | `/admin/emails/{id}/forward` | 전달 |
| GET | `/admin/approvals` | 승인 대기열 |
| POST | `/admin/approvals/{id}/action` | 승인/거절 |
| CRUD | `/admin/rules` | 규칙 관리 |
| CRUD | `/admin/mailboxes` | 메일함 설정 |
| CRUD | `/admin/engines` | AI 엔진 관리 |
| CRUD | `/admin/agents` | 상담원 관리 |
| CRUD | `/admin/signatures` | 서명 관리 |
| CRUD | `/admin/disclaimers` | 면책 문구 관리 |
| CRUD | `/admin/templates` | 답변 템플릿 |
| CRUD | `/admin/macros` | 매크로 관리 |
| CRUD | `/admin/repositories` | 지식 저장소 |
| CRUD | `/admin/knowledge-docs` | 지식 문서 |
| CRUD | `/admin/sla-policies` | SLA 정책 |
| CRUD | `/admin/automations` | 자동화 워크플로우 |
| CRUD | `/admin/business-hours` | 업무 시간 |
| CRUD | `/admin/notifications` | 알림 설정 |
| GET | `/admin/personal-emails` | 개인 메일함 목록 |
| GET | `/admin/statistics` | 통계 데이터 |

---

## 7. 구현 완료 현황

### Phase 1 — POC v0.1 ✅

| 기능 | 상태 | 비고 |
|------|------|------|
| Gmail Pub/Sub + 주기적 폴링 | ✅ 완료 | gmail_service.py |
| Claude CLI + MCP 통합 | ✅ 완료 | ai_processor.py |
| 신뢰도 기반 라우팅 | ✅ 완료 | pipeline.py |
| 규칙 엔진 + 자동 태깅 | ✅ 완료 | rule_engine.py |
| 멀티 브랜드 지원 | ✅ 완료 | mailbox_resolver.py |
| 승인 워크플로우 | ✅ 완료 | admin.py |
| 어드민 대시보드 (23페이지) | ✅ 완료 | Next.js admin/ |
| Google SSO 인증 | ✅ 완료 | auth.py |
| 개인 메일함 | ✅ 완료 | personal_gmail.py |
| 고객 정보 조회 (TBNWS CRM) | ✅ 완료 | customer_info_panel.tsx |
| 알림 시스템 (Slack/TG/Webhook) | ✅ 완료 | notification.py |
| SLA 정책 관리 | ✅ 완료 | admin.py |
| 매크로/캔드 리스폰스 | ✅ 완료 | admin.py |
| 지식 문서 관리 | ✅ 완료 | admin.py |
| Celery 비동기 태스크 | ✅ 완료 | workers/ |
| Docker 배포 설정 | ✅ 완료 | docker-compose.yml |

### Phase 2 — 계획 중

| 기능 | 상태 | 비고 |
|------|------|------|
| 로컬 LLM 통합 (vLLM/Ollama) | 📋 계획 | LocalLLMRouter 스켈레톤 있음 |
| RAG 파이프라인 (pgvector) | 📋 계획 | KnowledgeEmbedding 모델 있음 |
| GitHub/Notion 저장소 커넥터 | 📋 계획 | Repository 모델 지원 |
| 다국어 지원 | 📋 계획 | |
| 고급 분석/리포트 | 📋 계획 | |

---

## 8. 고객 매칭 규칙

> **중요**: 동명이인 오매칭 방지

- 전화번호 기반 매칭 **최우선**
- 이름(name) 단독 매칭 **금지**
- 전화번호 없으면 이메일 등 다른 식별자 사용

---

## 9. 환경 변수 (.env)

```env
# Database
DATABASE_URL=postgresql+asyncpg://bot:bot_password@localhost:5432/email_support_bot

# Redis
REDIS_URL=redis://localhost:6379/0

# Gmail (Service Account)
GMAIL_SERVICE_ACCOUNT_FILE=credentials/service-account.json
GMAIL_DELEGATED_USER=support@keychron.kr

# Google OAuth (Admin SSO)
GOOGLE_OAUTH_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=xxx
GOOGLE_OAUTH_REDIRECT_URI=http://localhost:3000/api/auth/google/callback

# JWT
JWT_SECRET_KEY=random-secret-key
JWT_ALGORITHM=HS256
JWT_EXPIRE_HOURS=24

# Claude CLI
CLAUDE_CLI_PATH=/usr/local/bin/claude

# TBNWS MySQL (Legacy CRM)
TBNWS_MYSQL_HOST=xxx
TBNWS_MYSQL_USER=xxx
TBNWS_MYSQL_PASSWORD=xxx
TBNWS_MYSQL_DB=xxx

# Admin
ADMIN_ALLOWED_ORIGINS=http://localhost:3001,http://localhost:3000
```

---

## 10. 실행 방법

```bash
# 1. 백엔드 서버
cd email-support-bot
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 3000 --reload

# 2. Celery 워커
celery -A app.workers.celery_app worker -l info

# 3. 프론트엔드
cd admin
npm install
npm run dev -- -p 3001

# 4. Docker (전체)
docker-compose up -d
```

---

## 11. 디렉토리 구조

```
email-support-bot/
├── app/
│   ├── main.py                 # FastAPI 앱 진입점
│   ├── core/
│   │   ├── config.py           # 환경 설정
│   │   ├── database.py         # SQLAlchemy 엔진
│   │   └── tbnws_mysql.py      # TBNWS CRM 연동
│   ├── models/
│   │   └── base.py             # ORM 모델 (23개 테이블)
│   ├── api/
│   │   ├── admin.py            # 어드민 API (136KB)
│   │   ├── auth.py             # 인증 API
│   │   ├── webhook.py          # Gmail 웹훅
│   │   └── health.py           # 헬스체크
│   ├── services/
│   │   ├── pipeline.py         # 이메일 처리 파이프라인
│   │   ├── ai_processor.py     # Claude CLI + LLM 추론
│   │   ├── gmail_service.py    # Gmail API 래퍼
│   │   ├── personal_gmail.py   # 개인 Gmail 서비스
│   │   ├── mailbox_resolver.py # 브랜드 감지
│   │   ├── rule_engine.py      # 규칙 매칭 엔진
│   │   ├── notification.py     # 알림 발송
│   │   └── auth.py             # JWT/OAuth 유틸
│   └── workers/
│       ├── celery_app.py       # Celery 설정
│       └── tasks.py            # 비동기 태스크
├── admin/                      # Next.js 프론트엔드
│   ├── src/
│   │   ├── app/                # 23개 페이지 라우트
│   │   ├── components/         # 19개 커스텀 컴포넌트 + UI
│   │   ├── hooks/              # 7개 커스텀 훅
│   │   └── lib/                # API 클라이언트, 쿼리, 타입
│   ├── next.config.ts          # 프록시 설정
│   └── package.json
├── docker-compose.yml
├── requirements.txt
├── alembic/                    # DB 마이그레이션
└── credentials/                # 서비스 계정 키
```
