# 통합 마이그레이션 + 병렬 개발 가이드 v2

> 갱신: 2026-03-13 — 코드 재점검 + 병렬 개발 구조 추가

---

## 1. 전체 DB 변경 요약

### 새 테이블

| 테이블 | Phase | 설명 |
|--------|-------|------|
| `celery_schedules` | 1 | Celery Beat 동적 스케줄 |
| `assignment_rules` | 2 | 자동배정 규칙 |

### 기존 테이블 변경

| 테이블 | Phase | 추가 컬럼 |
|--------|-------|----------|
| `mailbox_configs` | 1 | `poll_interval_minutes`, `last_polled_at`, `last_history_id` |
| `notification_routing` (단수!) | 2 | `agent_id`, `brand`, `category`, `priority_filter` |
| `agents` | 2 | `notification_channel_id`, `notification_events` |

---

## 2. 통합 SQL 마이그레이션

```sql
-- ================================================================
-- Phase 1: 자동 파이프라인
-- ================================================================

CREATE TABLE IF NOT EXISTS celery_schedules (
    id              SERIAL PRIMARY KEY,
    schedule_id     VARCHAR UNIQUE NOT NULL,
    name            VARCHAR NOT NULL,
    description     VARCHAR,
    task_name       VARCHAR NOT NULL,
    schedule_type   VARCHAR NOT NULL DEFAULT 'crontab',
    cron_expression VARCHAR,
    interval_seconds INTEGER,
    task_args       JSONB DEFAULT '[]'::jsonb,
    task_kwargs     JSONB DEFAULT '{}'::jsonb,
    queue           VARCHAR DEFAULT 'email_polling',
    is_active       BOOLEAN DEFAULT true,
    last_run_at     TIMESTAMPTZ,
    total_run_count INTEGER DEFAULT 0,
    last_result     VARCHAR,
    last_error      TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_celery_schedules_active ON celery_schedules (is_active);

ALTER TABLE mailbox_configs ADD COLUMN IF NOT EXISTS poll_interval_minutes INTEGER DEFAULT 5;
ALTER TABLE mailbox_configs ADD COLUMN IF NOT EXISTS last_polled_at TIMESTAMPTZ;
ALTER TABLE mailbox_configs ADD COLUMN IF NOT EXISTS last_history_id VARCHAR;


-- ================================================================
-- Phase 2: 알림 라우팅 + 자동배정
-- ================================================================

CREATE TABLE IF NOT EXISTS assignment_rules (
    id                      SERIAL PRIMARY KEY,
    rule_id                 VARCHAR UNIQUE NOT NULL,
    name                    VARCHAR NOT NULL,
    description             VARCHAR,
    brand                   VARCHAR,
    category                VARCHAR,
    priority_match          VARCHAR,
    agent_id                INTEGER REFERENCES agents(id),
    group_id                INTEGER REFERENCES agent_groups(id),
    strategy                VARCHAR DEFAULT 'round_robin',
    priority                INTEGER DEFAULT 100,
    is_active               BOOLEAN DEFAULT true,
    last_assigned_agent_id  INTEGER,
    created_at              TIMESTAMPTZ DEFAULT now(),
    updated_at              TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_assignment_rules_brand ON assignment_rules (brand);
CREATE INDEX IF NOT EXISTS idx_assignment_rules_priority ON assignment_rules (priority);

-- ※ 테이블명 주의: notification_routing (단수)
ALTER TABLE notification_routing ADD COLUMN IF NOT EXISTS agent_id INTEGER REFERENCES agents(id);
ALTER TABLE notification_routing ADD COLUMN IF NOT EXISTS brand VARCHAR;
ALTER TABLE notification_routing ADD COLUMN IF NOT EXISTS category VARCHAR;
ALTER TABLE notification_routing ADD COLUMN IF NOT EXISTS priority_filter VARCHAR;

ALTER TABLE agents ADD COLUMN IF NOT EXISTS notification_channel_id VARCHAR
    REFERENCES notification_channels(channel_id);
ALTER TABLE agents ADD COLUMN IF NOT EXISTS notification_events JSONB
    DEFAULT '["approval_needed","escalation"]'::jsonb;


-- ================================================================
-- 초기 데이터
-- ================================================================

INSERT INTO celery_schedules (schedule_id, name, task_name, schedule_type, cron_expression, queue, is_active)
VALUES ('poll-all-mailboxes', '전체 메일박스 폴링', 'app.workers.tasks.check_new_emails', 'crontab', '*/5 * * * *', 'email_polling', true)
ON CONFLICT (schedule_id) DO NOTHING;
```

---

## 3. 전체 파일 변경 목록

### 신규 파일 (10개)

| 파일 | Phase | 설명 |
|------|-------|------|
| `app/workers/db_scheduler.py` | 1 | DB 기반 Celery Beat 스케줄러 |
| `app/services/schedule_service.py` | 1 | 스케줄 CRUD + 메일박스 동기화 |
| `app/services/assignment_service.py` | 2 | 자동배정 엔진 (fixed/RR/LL) |
| `admin/src/app/schedules/page.tsx` | 1 | 스케줄 관리 UI |
| `admin/src/app/assignment-rules/page.tsx` | 2 | 배정 규칙 관리 UI |
| `admin/src/components/role-guard.tsx` | 3 | 페이지 역할 가드 |
| `scripts/dev.sh` | 1 | 개발환경 원클릭 시작 |
| `seed-data/celery_schedules.json` | 1 | 스케줄 시드 데이터 |
| `seed-data/assignment_rules.json` | 2 | 배정 규칙 시드 데이터 |
| `alembic/versions/*_phase1_2.py` | 1+2 | 통합 마이그레이션 |

### 수정 파일 (13개)

| 파일 | Phase | 변경 내용 |
|------|-------|----------|
| `app/models/base.py` | 1,2 | +CelerySchedule, +AssignmentRule, MailboxConfig 확장, NotificationRouting 확장, Agent 확장 |
| `app/core/database.py` | 1 | +sync_engine, +SyncSessionLocal (Beat용) |
| `app/services/gmail_service.py` | 1 | 생성자에 delegated_user 파라미터 추가, list_unread_messages에 query 파라미터 |
| `app/services/notification.py` | 2 | +EmailAdapter, notify() 시그니처 확장 (brand/category/assignee_id kwargs), 컨텍스트 기반 라우팅 |
| `app/services/pipeline.py` | 2 | +AssignmentService.auto_assign() 호출, notify()에 컨텍스트 전달 (3곳) |
| `app/services/auth.py` | 3 | require_role() 수정 (X-Admin-Key 지원), +require_admin/agent_or_above/any_authenticated, +get_current_user_optional, +require_email_access |
| `app/workers/celery_app.py` | 1 | beat_schedule 제거, +check_mailbox_emails 라우팅 |
| `app/workers/tasks.py` | 1 | +check_mailbox_emails 태스크, +_check_single_mailbox |
| `app/api/admin.py` | 1,2,3 | +스케줄 CRUD (7개), +배정 규칙 CRUD (4개), +라우팅 CRUD (4개), +GET /me, verify_admin→require_role 교체 (65+ 엔드포인트) |
| `admin/src/components/app-sidebar.tsx` | 1,2,3 | +스케줄/배정 메뉴, NavItem에 roles 추가, 역할별 필터링 |
| `admin/src/app/notifications/page.tsx` | 2 | +라우팅 규칙 탭, +에이전트 알림 탭 |
| `admin/src/app/dashboard/page.tsx` | 3 | 역할별 대시보드 분기 (admin/agent/viewer) |
| `scripts/seed_data.py` | 1,2 | SEED_MAP에 celery_schedules, assignment_rules 추가 |
| `docker-compose.yml` | 1 | beat command에 -S DatabaseScheduler 추가 |

---

## 4. 병렬 개발 구조 (Claude Code 워크트리 / 클로드 팀)

### 4.1 개발 스트림 분리

전체 태스크를 **4개 독립 스트림**으로 분리:

```
Stream A: Backend Core (모델 + 서비스)
Stream B: Backend API (admin.py 엔드포인트)
Stream C: Frontend (Admin UI 페이지/컴포넌트)
Stream D: Infrastructure (Celery, Docker, Scripts)
```

### 4.2 Phase 1 병렬 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Phase 1                               │
├──────────┬──────────┬──────────┬──────────────────────────────┤
│ Stream A │ Stream B │ Stream C │ Stream D                     │
│ (Backend)│ (API)    │ (Frontend)│ (Infra)                     │
├──────────┼──────────┼──────────┼──────────────────────────────┤
│ 1-1 모델 │          │          │ 1-10 docker+dev.sh          │
│ 1-2 sync │          │          │                              │
│ 1-3 Gmail│          │          │                              │
│   ↓      │          │          │                              │
│ 1-4 Sched│          │          │                              │
│ 1-5 Task │          │          │                              │
│ 1-6 Svc  │          │          │                              │
│   ↓      │   ↓      │          │                              │
│          │ 1-7 API  │          │                              │
│          │   ↓      │   ↓      │                              │
│          │ 1-8 celery│ 1-9 UI  │                              │
└──────────┴──────────┴──────────┴──────────────────────────────┘

병렬 실행 가능:
  T=0: Stream A (1-1,1-2,1-3) + Stream D (1-10)
  T=1: Stream A (1-4,1-5,1-6)
  T=2: Stream B (1-7,1-8) + Stream C (1-9, Mock API)
```

### 4.3 Phase 2 병렬 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Phase 2                               │
├──────────┬──────────┬──────────────────────────────────────────┤
│ Stream A │ Stream B │ Stream C                                │
├──────────┼──────────┼──────────────────────────────────────────┤
│ 2-1 모델 │          │                                         │
│   ↓      │          │                                         │
│ 2-2 배정 │ 2-5 API  │ 2-6a 배정규칙 UI │ 2-6b 알림라우팅 UI  │
│ 2-3 알림 │          │                  │                     │
│   ↓      │          │                  │                     │
│ 2-4 파이프│          │                  │                     │
└──────────┴──────────┴──────────────────┴─────────────────────┘

병렬 실행 가능:
  T=0: Stream A (2-1)
  T=1: Stream A (2-2,2-3) + Stream B (2-5) + Stream C (2-6a,2-6b)
  T=2: Stream A (2-4) + 통합 테스트
```

### 4.4 Phase 3 병렬 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Phase 3                               │
├──────────┬──────────────────────────────────────────────────┤
│ Stream A │ Stream C                                         │
├──────────┼──────────────────────────────────────────────────┤
│ 3-1 auth │ 3-3a 사이드바 필터링                              │
│   ↓      │ 3-3b 에이전트 대시보드                            │
│ 3-2 적용 │ 3-3c RoleGuard 컴포넌트                          │
└──────────┴──────────────────────────────────────────────────┘

병렬 실행 가능:
  T=0: Stream A (3-1) + Stream C (3-3a,3-3b,3-3c)
  T=1: Stream A (3-2)
  T=2: 통합 테스트
```

### 4.5 Claude Code 워크트리 사용법

```bash
# Stream A: Backend Core
claude --worktree "phase1-backend"
# 프롬프트: "Phase 1 docs/phase1-auto-pipeline.md 참조하여 Task 1-1~1-6 구현"

# Stream B: Backend API (Stream A 완료 후)
claude --worktree "phase1-api"
# 프롬프트: "Phase 1 Task 1-7, 1-8 구현"

# Stream C: Frontend (Mock API로 병렬 가능)
claude --worktree "phase1-frontend"
# 프롬프트: "Phase 1 Task 1-9 구현 — API는 /api/admin/schedules 형식"

# Stream D: Infrastructure (독립)
claude --worktree "phase1-infra"
# 프롬프트: "Phase 1 Task 1-10 구현"
```

### 4.6 전체 Phase 간 의존성

```
Phase 1 ──→ Phase 2 ──→ Phase 3
  ↓             ↓           ↓
모델/DB       서비스       API 레벨
스키마 추가   로직 추가   권한 적용

※ Phase 3은 Phase 1,2와 코드 충돌이 거의 없음
  (admin.py dependencies 변경만)
  → Phase 2와 Phase 3 Frontend를 병렬 진행 가능
```

---

## 5. 구현 시 주의사항 (코드 재점검에서 발견)

### 5.1 테이블명 불일치 주의

```python
# notification_routing (단수!) — base.py:432
class NotificationRouting(Base):
    __tablename__ = "notification_routing"

# 문서에서 "notification_routings" (복수)로 오기하지 않도록 주의
# SQL ALTER 시: ALTER TABLE notification_routing ADD COLUMN ...
```

### 5.2 ADAPTER_MAP이 lambda 팩토리

```python
# 현재 (notification.py:91-95):
ADAPTER_MAP = {
    "slack": lambda cfg: SlackWebhookAdapter(cfg["webhook_url"]),
    ...
}

# email 추가 시 동일 패턴으로:
"email": lambda cfg: EmailAdapter(cfg["to"], cfg.get("from", "ai@tbe.kr")),
```

### 5.3 기존 require_role() 수정 필수

```python
# 현재 auth.py:159-167 — get_current_user 의존 (X-Admin-Key 미지원)
# 수정 필요: get_current_user_optional 의존으로 변경 (None=슈퍼어드민 통과)
```

### 5.4 verify_admin → require_role 마이그레이션 전략

**위험**: admin.py에서 65+ 엔드포인트를 한꺼번에 변경하면 regression 위험

**안전한 방법**:
1. 먼저 require_role을 정의하되, verify_admin도 유지
2. 한 그룹씩 변경 (dashboard → emails → rules → ...)
3. 각 그룹 변경 후 테스트
4. 모든 그룹 완료 후 verify_admin 삭제

### 5.5 GmailService delegation 제약

Service Account delegation이 각 메일박스 주소에 설정되어 있지 않으면:
- **폴백 전략**: 공용 계정(ai@tbe.kr)으로 전체 메일 조회 + `to:` 필터
- check_mailbox_emails에서 try/except로 delegation 실패 시 폴백 구현

### 5.6 notify() 하위 호환성

```python
# 기존 호출 (pipeline.py에서 4곳):
await self.notification.notify("auto_sent", message, db, metadata)

# 수정 후에도 기존 호출이 동작해야 함
# → 새 파라미터를 keyword-only (*)로 정의
async def notify(self, event_type, message, db, metadata=None, *, brand=None, ...):
```

---

## 6. Seed Data 업데이트

seed_data.py SEED_MAP 최종:

```python
SEED_MAP = [
    ("knowledge_repositories.json", KnowledgeRepository, "repo_id"),
    ("knowledge_documents.json", KnowledgeDocument, "slug"),
    ("rules.json", Rule, "rule_id"),
    ("mailbox_config.json", MailboxConfig, "address"),
    ("llm_engines.json", LLMEngine, "engine_id"),
    ("notification_channels.json", NotificationChannel, "channel_id"),
    ("ai_disclaimers.json", AIDisclaimer, "disclaimer_id"),
    ("agents.json", Agent, "email"),
    ("celery_schedules.json", CelerySchedule, "schedule_id"),       # Phase 1
    ("assignment_rules.json", AssignmentRule, "rule_id"),            # Phase 2
]
```

---

## 7. 문서 인덱스

| 파일 | 버전 | 내용 |
|------|------|------|
| `docs/implementation-plan.md` | v1 | 전체 계획 개요 |
| `docs/phase1-auto-pipeline.md` | **v2** | Phase 1 상세 (10개 태스크, 의존성 그래프, 워크플로우) |
| `docs/phase2-notification-assignment.md` | **v2** | Phase 2 상세 (6개 태스크, 배정 시나리오, 테이블명 주의) |
| `docs/phase3-rbac-permissions.md` | **v2** | Phase 3 상세 (기존 코드 활용 전략, 65+ 엔드포인트 매트릭스) |
| `docs/migration-guide.md` | **v2** | 통합 SQL, 전체 파일 목록, 병렬 개발 구조, 주의사항 |
