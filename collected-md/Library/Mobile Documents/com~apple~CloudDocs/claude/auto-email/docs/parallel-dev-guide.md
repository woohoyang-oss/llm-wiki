# 병렬 개발 가이드 — Worktree + 로컬 LLM 활용

> 작성일: 2026-03-13
> 최종 테스트: 2026-03-13 23:10 KST
> 목적: Claude Code 워크트리 기반 병렬 개발 + 로컬 Ollama 엔진 활용 전략

---

## 1. 현재 리소스 현황

### 1.1 로컬 LLM 엔진

| 호스트 | 포트 | 모델 | 테스트 결과 | 용도 제안 |
|--------|------|------|------------|-----------|
| wooho-pc:11434 | 11434 | qwen2.5:14b | ✅ 38.1s | 이메일 분류/태깅 |
| wooho-pc:11434 | 11434 | qwen3:8b | ✅ 8.1s (최빠름) | 빠른 초안 (가벼운 문의) |
| wooho-pc:11434 | 11434 | qwen3:14b | ✅ | 이메일 답변 생성 |
| wooho-pc:11434 | 11434 | qwen3:8b | — | 빠른 초안 (가벼운 문의) |
| wooho-studio:11434 | 11434 | qwen3:14b | — | ❌ 연결 안됨 (2026-03-13 테스트) |
| dev-macstudio:11434 | 11434 | qwen3:14b | ✅ 26.1s | 이메일 답변 생성 |
| dev-macstudio:11434 | 11434 | deepseek-r1:14b | ✅ 24.7s | 복잡한 추론 (견적/발주) |
| tobe-macmini:11434 | 11434 | qwen3-nothink:14b | ✅ 32.5s | 빠른 답변 (thinking 없이) |
| tobe-macmini:11434 | 11434 | qwen3:14b | ✅ 63.5s | 이메일 답변 (대체) |

### 1.2 개발 인프라

- **Git 저장소**: GitHub (main 브랜치)
- **Claude Code**: 워크트리 기반 병렬 세션 가능
- **DB**: PostgreSQL 16 + pgvector (localhost:5432)
- **Redis**: localhost:6379
- **Celery**: Worker + Beat

---

## 2. 로컬 LLM 활용 방법

### 2.1 이미 구현된 인프라 (즉시 사용 가능)

시스템에 이미 `LocalLLMProcessor` + `AIRouter` + `LLMEngine` DB 모델이 구현되어 있음.

**Admin UI → LLM 엔진 페이지에서 등록:**

```
/engines 페이지에서 아래 엔진들을 등록:

이름: Qwen3-14B (Studio)
Type: openai_compat
Endpoint: http://wooho-studio:11434/v1
Model: qwen3:14b
Priority: 50
Active: true

이름: DeepSeek-R1 (Reasoning)
Type: openai_compat
Endpoint: http://dev-macstudio:11434/v1
Model: deepseek-r1:14b
Priority: 100
Active: true
Capabilities: ["generation", "classification"]
```

### 2.2 Ollama는 OpenAI 호환 API 제공

Ollama는 `/v1/chat/completions` 엔드포인트를 기본 제공하므로 `LocalLLMProcessor`가 바로 연결됨.

```bash
# 테스트: 각 호스트에서 동작 확인
curl http://wooho-studio:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3:14b","messages":[{"role":"user","content":"Hello"}]}'
```

### 2.3 엔진 선택 전략 (룰별 배정)

| 메일 유형 | 추천 엔진 | 이유 |
|-----------|-----------|------|
| 단순 배송 문의 | qwen3:8b (wooho-pc) | 빠른 응답, 패턴화된 답변 |
| 일반 CS (교환/반품) | qwen3:14b (wooho-studio) | 중간 복잡도, 안정적 |
| 견적/발주 문의 | deepseek-r1:14b (dev-macstudio) | 수치 계산, 추론 필요 |
| 복잡한 B2B/VIP | Claude CLI (API) | 최고 품질, tool_use(MCP) |
| 대량 분류 작업 | qwen3-nothink:14b (tobe-macmini) | thinking 없이 빠른 분류 |

**룰에서 engine_id 지정 방법:**
```json
{
  "name": "배송문의 자동답변",
  "conditions": {"category": "shipping"},
  "action_type": "auto_reply",
  "engine_id": "qwen3-8b-pc"
}
```

### 2.4 로드밸런싱 / 페일오버

현재 `AIRouter._get_priority_engine()`는 priority 기반 단일 선택.

**Phase 1 이후 개선안** (선택적):

```python
# 같은 모델의 여러 호스트를 등록하고 라운드로빈:
# qwen3-14b-studio  priority=50  endpoint=wooho-studio:11434
# qwen3-14b-mac     priority=50  endpoint=dev-macstudio:11434
# qwen3-14b-mini    priority=50  endpoint=tobe-macmini:11434
#
# 동일 priority → 랜덤 또는 라운드로빈 선택
# 한 호스트 fail → health_status='unhealthy' → 자동 스킵
```

이 기능은 `AIRouter._get_priority_engine()` 메서드에 간단한 로직 추가로 구현 가능.

### 2.5 Health Check 자동화

```python
# 이미 LLMEngine.health_status, last_health_at 컬럼 존재
# Celery Beat에 헬스체크 태스크 추가 (Phase 1 스케줄러에 포함 가능):

@celery_app.task
def check_engine_health():
    """5분마다 각 엔진 /v1/models 호출, 상태 업데이트"""
    for engine in active_engines:
        try:
            resp = requests.get(f"{engine.endpoint_url}/v1/models", timeout=5)
            engine.health_status = "healthy" if resp.ok else "unhealthy"
        except:
            engine.health_status = "unhealthy"
        engine.last_health_at = datetime.utcnow()
```

---

## 3. 워크트리 병렬 개발 구조

### 3.1 의존성 분석

```
Phase 0 (공유 인프라) ──── 모든 Phase의 선행 조건
    │
    ├── Phase 1 (자동 파이프라인) ─── 독립 실행 가능
    │
    ├── Phase 3 (RBAC) ─── 독립 실행 가능
    │
    └── Phase 4-A (DB 마이그레이션 + ScopeService) ─── 독립 실행 가능
            │
            ├── Phase 4-B (API scope 확장)
            └── Phase 4-C (Frontend scope 탭)

Phase 2 (알림 + 자동배정) ─── Phase 1 이후 (pipeline.py 의존)

Phase 5 (대시보드) ─── Phase 3 이후 (역할별 분기 의존)
```

### 3.2 병렬 실행 가능한 조합

**최대 3개 동시 워크트리:**

```
┌─────────────────────────────────────────────────────────┐
│ Week 1                                                  │
│                                                         │
│  Worktree A (main)     : Phase 0 — 공유 인프라 구현       │
│                          ScopeMixin, ScopeService,       │
│                          crud_helpers, auth 수정          │
│                          DB 마이그레이션 (전체)             │
│                                                         │
│  ── Phase 0 완료 후 main에 merge ──                       │
│                                                         │
│ Week 2                                                  │
│                                                         │
│  Worktree A (phase1)   : Phase 1 — 자동 파이프라인        │
│  Worktree B (phase3)   : Phase 3 — RBAC 권한             │
│  Worktree C (phase4-fe): Phase 4 Frontend — scope 탭 9개 │
│                                                         │
│ Week 3                                                  │
│                                                         │
│  Worktree A (phase2)   : Phase 2 — 알림 + 자동배정        │
│  Worktree B (phase4-be): Phase 4 Backend — API scope     │
│  Worktree C (phase5)   : Phase 5 — 대시보드 + 통계        │
│                                                         │
│ Week 4                                                  │
│                                                         │
│  main                  : 통합 테스트 + 버그픽스            │
│  로컬 LLM 엔진 등록 + 룰별 엔진 배정 설정                   │
└─────────────────────────────────────────────────────────┘
```

### 3.3 워크트리별 충돌 위험 분석

| 파일 | 수정하는 Phase | 충돌 위험 | 해결 전략 |
|------|---------------|-----------|-----------|
| `app/api/admin.py` | 1, 2, 3, 4, 5 | **높음** | 섹션별 분할, 각 Phase가 다른 섹션만 수정 |
| `app/services/pipeline.py` | 1, 2 | 중간 | Phase 1 먼저 → Phase 2는 Phase 1 merge 후 |
| `app/services/auth.py` | 3 | 낮음 | Phase 3 전용 |
| `app/services/notification.py` | 2 | 낮음 | Phase 2 전용 |
| `app/models/base.py` | 0, 4 | 중간 | Phase 0에서 ScopeMixin 추가 완료 후 |
| `admin/src/app/*/page.tsx` | 4, 5 | 낮음 | 각각 다른 페이지 수정 |
| `admin/src/components/app-sidebar.tsx` | 3, 5 | 낮음 | Phase 3: roles 필터, Phase 5: 대시보드 분기 |
| `admin/src/lib/queries.ts` | 5 | 낮음 | Phase 5 전용 |
| `admin/src/lib/mutations.ts` | 4 | 낮음 | Phase 4 전용 (또는 별도 훅 파일) |

### 3.4 admin.py 충돌 방지 전략

`admin.py`는 3668줄로 모든 Phase가 수정하는 핵심 파일. 충돌 방지를 위한 규칙:

```
admin.py 섹션 배정:

  Line 1~100      : imports, verify_admin     → Phase 3만 수정
  Line 100~500    : dashboard/statistics API  → Phase 5만 수정
  Line 500~1000   : email CRUD               → Phase 1만 수정 (폴링 관련)
  Line 1000~1500  : rules, templates, macros  → Phase 4만 수정 (scope 추가)
  Line 1500~2000  : mailboxes, signatures     → Phase 1, 4 (순차 실행)
  Line 2000~2500  : notification, agents      → Phase 2만 수정
  Line 2500~3000  : repositories, knowledge   → Phase 4만 수정
  Line 3000~3500  : engines, system           → 수정 없음
  Line 3500~3668  : personal email            → Phase 4만 수정

  신규 엔드포인트: 파일 맨 끝에 추가 (충돌 최소화)
```

**워크트리별 규칙**:
- 기존 엔드포인트 수정은 배정된 섹션만
- 신규 엔드포인트는 파일 끝에 추가 (merge 시 auto-resolve 가능)

---

## 4. Claude Code 워크트리 실행 방법

### 4.1 워크트리 생성

```bash
# 터미널 1: Phase 1 워크트리
cd /Users/woohoyang/Downloads/claude/auto-email
git worktree add .claude/worktrees/phase1 -b phase1

# 터미널 2: Phase 3 워크트리
git worktree add .claude/worktrees/phase3 -b phase3

# 터미널 3: Phase 4 Frontend 워크트리
git worktree add .claude/worktrees/phase4-fe -b phase4-fe
```

### 4.2 각 워크트리에서 Claude Code 실행

```bash
# 터미널 1
cd .claude/worktrees/phase1
claude

# 프롬프트:
# "Phase 1 자동 파이프라인을 구현해줘.
#  docs/phase1-auto-pipeline.md를 참조하고
#  docs/dev-framework.md §5의 컨텍스트 맵을 따라서."

# 터미널 2
cd .claude/worktrees/phase3
claude

# 터미널 3
cd .claude/worktrees/phase4-fe
claude
```

### 4.3 Merge 순서

```bash
# 1. Phase 0 (공유 인프라) → main
git checkout main
git merge phase0

# 2. Phase 1, 3은 독립이므로 순서 무관
git merge phase1
git merge phase3  # 충돌 시: admin.py 섹션 확인

# 3. Phase 4는 Phase 0 이후
git merge phase4-fe
git merge phase4-be

# 4. Phase 2는 Phase 1 이후
git merge phase2

# 5. Phase 5는 Phase 3 이후
git merge phase5
```

---

## 5. 로컬 LLM 개발 시 활용

### 5.1 개발 중 AI 테스트 (Claude API 비용 절약)

개발/테스트 시 Claude API 대신 로컬 LLM 사용:

```bash
# .env.development
DEFAULT_ENGINE=qwen3-14b-studio

# 또는 Admin UI에서:
# 1. /engines 페이지에서 로컬 엔진 등록
# 2. priority를 1로 설정 (최우선)
# 3. Claude CLI 엔진 priority를 999로 설정
# → 모든 AI 처리가 로컬 LLM으로 라우팅
```

### 5.2 용도별 엔진 배정 제안

```
┌──────────────────────────────────────────────────────┐
│ Production 엔진 배치 (제안)                            │
│                                                      │
│  ┌─── 1차 필터: 분류 ────────────────────────────┐   │
│  │ qwen3-nothink:14b (tobe-macmini)              │   │
│  │ 역할: 메일 카테고리 분류 + 태깅                  │   │
│  │ 속도: 가장 빠름 (thinking 없음)                 │   │
│  │ priority: 10 (분류 전용 룰에 배정)              │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─── 2차: 일반 답변 생성 ──────────────────────┐   │
│  │ qwen3:14b (wooho-studio / dev-macstudio)      │   │
│  │ 역할: 배송/교환/반품 등 일반 CS 답변             │   │
│  │ 로드밸런싱: 2대로 분산                          │   │
│  │ priority: 50                                   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─── 3차: 복잡한 추론 ────────────────────────┐   │
│  │ deepseek-r1:14b (dev-macstudio)               │   │
│  │ 역할: 견적 계산, 발주 판단, 복잡한 문의          │   │
│  │ priority: 70                                   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─── 4차: 최고 품질 (폴백) ──────────────────┐    │
│  │ Claude API (claude-cli)                       │   │
│  │ 역할: VIP/B2B, MCP tool_use 필요 케이스       │   │
│  │ priority: 100                                  │   │
│  │ ※ MCP tool_use는 Claude만 지원                 │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### 5.3 MCP tool_use 제한사항

**중요**: 로컬 LLM(Ollama)은 MCP tool_use를 지원하지 않음.

| 기능 | Claude CLI | 로컬 LLM (Ollama) |
|------|------------|-------------------|
| 답변 생성 | ✅ | ✅ |
| 지식 문서 참조 | ✅ (tool_use) | ✅ (시스템 프롬프트에 주입) |
| 실시간 DB 조회 (MCP) | ✅ | ❌ |
| 고객 이력 조회 | ✅ (tool_use) | ✅ (_fetch_customer_context 결과 주입) |
| confidence 산출 | ✅ | ✅ (JSON 파싱) |

**현재 구현 상태**: `AIRouter.process()` (ai_processor.py:1310-1393)에서 로컬 LLM 호출 시 이미 지식 문서, 유사 사례, 고객 컨텍스트를 시스템 프롬프트에 주입하도록 구현되어 있음. MCP tool_use만 로컬 LLM에서는 불가.

### 5.4 LLM 엔진 DB 등록 SQL

```sql
-- Admin UI (/engines)에서도 등록 가능하지만, 일괄 등록 시:

INSERT INTO llm_engines (engine_id, name, type, endpoint_url, model_name, priority, capabilities, max_tokens, timeout_sec, is_active) VALUES
('qwen25-14b-pc',       'Qwen2.5 14B (PC)',         'openai_compat', 'http://wooho-pc:11434/v1',       'qwen2.5:14b',        80, '["classification"]', 4096, 120, true),
('qwen3-8b-pc',         'Qwen3 8B Fast (PC)',       'openai_compat', 'http://wooho-pc:11434/v1',        'qwen3:8b',           60, '["generation"]', 4096, 60, true),
('qwen3-14b-studio',    'Qwen3 14B (Studio)',       'openai_compat', 'http://wooho-studio:11434/v1',    'qwen3:14b',          50, '["generation","classification"]', 8192, 180, true),
('qwen3-14b-mac',       'Qwen3 14B (MacStudio)',    'openai_compat', 'http://dev-macstudio:11434/v1',   'qwen3:14b',          50, '["generation","classification"]', 8192, 180, true),
('deepseek-r1-mac',     'DeepSeek R1 (MacStudio)',  'openai_compat', 'http://dev-macstudio:11434/v1',   'deepseek-r1:14b',    70, '["generation","reasoning"]', 8192, 300, true),
('qwen3-nothink-mini',  'Qwen3 NoThink (Mini)',     'openai_compat', 'http://tobe-macmini:11434/v1',    'qwen3-nothink:14b',  30, '["classification"]', 4096, 60, true),
('qwen3-14b-mini',      'Qwen3 14B (Mini)',         'openai_compat', 'http://tobe-macmini:11434/v1',    'qwen3:14b',          50, '["generation","classification"]', 8192, 180, true);
```

---

## 6. 태스크 단위 분해 (워크트리별)

### Phase 0: 공유 인프라 (워크트리 불필요 — main에서 직접)

| # | 태스크 | 수정 파일 | 예상 |
|---|--------|-----------|------|
| 0-1 | DB 마이그레이션 (scope/owner_id 9테이블 + 2신규) | alembic/versions/ | 30분 |
| 0-2 | ScopeMixin 모델 추가 | app/models/base.py | 15분 |
| 0-3 | ScopeService 구현 | app/services/scope_service.py (신규) | 20분 |
| 0-4 | crud_helpers 구현 | app/services/crud_helpers.py (신규) | 20분 |
| 0-5 | get_current_user_optional | app/services/auth.py | 10분 |
| 0-6 | require_role 수정 | app/services/auth.py | 10분 |
| 0-7 | Frontend 공용 컴포넌트 4개 | admin/src/components/, hooks/ | 30분 |

### Worktree A: Phase 1 (자동 파이프라인)

| # | 태스크 | 충돌 파일 없음 확인 |
|---|--------|-------------------|
| 1-1 | sync_session 추가 | app/core/database.py — Phase 1 전용 |
| 1-2 | celery_schedules 테이블 | alembic — 별도 migration |
| 1-3 | DatabaseScheduler | app/workers/scheduler.py (신규) |
| 1-4 | GmailService 멀티메일박스 | app/services/gmail_service.py |
| 1-5 | 폴링 태스크 리팩토링 | app/workers/tasks.py |
| 1-6 | 스케줄 Admin API | admin.py 맨 끝에 추가 |
| 1-7 | 스케줄 Admin UI | admin/src/app/mailboxes/page.tsx |
| 1-8 | dev.sh 시작 스크립트 | scripts/dev.sh (신규) |

### Worktree B: Phase 3 (RBAC)

| # | 태스크 | 충돌 파일 없음 확인 |
|---|--------|-------------------|
| 3-1 | verify_admin → require_role 교체 | admin.py 상단 (import + 데코레이터) |
| 3-2 | 65개 엔드포인트에 역할 적용 | admin.py 전체 (데코레이터만 추가) |
| 3-3 | Frontend RoleGuard | admin/src/components/role-guard.tsx (신규) |
| 3-4 | Sidebar 역할 필터 | admin/src/components/app-sidebar.tsx |

### Worktree C: Phase 4 Frontend

| # | 태스크 | 충돌 파일 없음 확인 |
|---|--------|-------------------|
| 4-F1 | ScopeTabs 적용: rules | admin/src/app/rules/page.tsx |
| 4-F2 | ScopeTabs 적용: repositories | admin/src/app/repositories/page.tsx |
| 4-F3 | ScopeTabs 적용: knowledge-docs | admin/src/app/knowledge-docs/page.tsx |
| 4-F4 | ScopeTabs 적용: email-templates | admin/src/app/email-templates/page.tsx |
| 4-F5 | ScopeTabs 적용: macros | admin/src/app/macros/page.tsx |
| 4-F6 | ScopeTabs 적용: signatures | admin/src/app/signatures/page.tsx |
| 4-F7 | ScopeTabs 적용: disclaimers | admin/src/app/disclaimers/page.tsx |
| 4-F8 | ScopeTabs 적용: notifications | admin/src/app/notifications/page.tsx |
| 4-F9 | 개인설정 UI | admin/src/app/my-inbox/settings/ (신규) |
| 4-F10 | 지식 저장 모달 | admin/src/components/knowledge-save-modal.tsx (신규) |

---

## 7. Claude Code 세션 프롬프트 템플릿

### 7.1 Worktree A (Phase 1) 시작 프롬프트

```
이 워크트리에서 Phase 1 (자동 파이프라인)을 구현합니다.

참조 문서:
- docs/phase1-auto-pipeline.md
- docs/dev-framework.md §5 (Phase 1 컨텍스트 맵)

구현 순서: 1-1 → 1-2 → 1-3 → 1-4 → 1-5 → 1-6 → 1-7 → 1-8

규칙:
- admin.py 수정 시 맨 끝에 신규 엔드포인트 추가
- 기존 엔드포인트는 폴링 관련 섹션(Line 500~1000)만 수정
- 다른 Phase와 겹치는 파일(auth.py, models/base.py) 수정 금지

1-1 태스크부터 시작해줘.
```

### 7.2 Worktree B (Phase 3) 시작 프롬프트

```
이 워크트리에서 Phase 3 (RBAC)을 구현합니다.

참조 문서:
- docs/phase3-rbac-permissions.md
- docs/shared-infrastructure.md §1.5 (require_role)

구현 순서: 3-1 → 3-2 → 3-3 → 3-4

규칙:
- admin.py에서는 데코레이터(require_role) 추가만
- 엔드포인트 로직 변경 금지
- auth.py: require_role 함수만 수정 (Phase 0에서 get_current_user_optional 이미 추가됨)

3-1 태스크부터 시작해줘.
```

### 7.3 Worktree C (Phase 4 Frontend) 시작 프롬프트

```
이 워크트리에서 Phase 4 Frontend (scope 탭 9개 페이지)를 구현합니다.

참조 문서:
- docs/dev-framework.md §4 (프론트엔드 템플릿)
- docs/shared-infrastructure.md §2 (ScopeTabs, hooks)

전략: "첫 번째 구현 → 나머지 복사"
1. rules/page.tsx에 ScopeTabs + useScopedQuery 적용 (첫 번째)
2. 나머지 8개 페이지에 동일 패턴 복사

규칙:
- Backend API는 수정하지 않음 (별도 워크트리)
- 공용 컴포넌트(scope-tabs, hooks)는 Phase 0에서 이미 생성됨
- queries.ts / mutations.ts 수정 최소화 (별도 훅 파일 사용)

4-F1 (rules 페이지)부터 시작해줘.
```

---

## 8. 로컬 LLM으로 개발 비용 절감 요약

### 8.1 개발 비용 시뮬레이션

| 용도 | Claude API | 로컬 LLM | 절감 |
|------|------------|----------|------|
| 개발 중 AI 답변 테스트 | ~$0.05/건 | $0 | 100% |
| 일반 CS 답변 (운영) | ~$0.03/건 | $0 (전기료만) | ~100% |
| 분류/태깅 (운영) | ~$0.01/건 | $0 | 100% |
| 복잡한 B2B/VIP (운영) | ~$0.05/건 | Claude 유지 | 0% |
| MCP tool_use (운영) | ~$0.10/건 | Claude 유지 | 0% |

### 8.2 하이브리드 전략

```
일반 CS (80% 볼륨) → 로컬 LLM (비용 $0)
    ↓ confidence < 0.70이면
Claude API 폴백 (20% 볼륨)

예상 Claude API 비용 절감: 70~80%
```

### 8.3 즉시 실행 가능

1. 각 호스트에서 Ollama 실행 확인 (`ollama serve`)
2. Admin UI `/engines`에서 7개 엔진 등록 (§5.4 SQL 또는 UI)
3. 룰별 `engine_id` 설정
4. 테스트: 메일 수신 → 로컬 LLM으로 답변 생성 확인

---

## 9. CLAUDE.md 생성 (프로젝트 설정)

워크트리 병렬 개발 시 각 Claude 세션이 프로젝트 규칙을 알 수 있도록:

```markdown
# CLAUDE.md (프로젝트 루트에 생성)

## 프로젝트
Email Support Bot — FastAPI + Next.js 14 + PostgreSQL

## 기술 스택
- Backend: Python 3.12, FastAPI, Celery, SQLAlchemy (async)
- Frontend: Next.js 14, shadcn/ui, TanStack Query
- DB: PostgreSQL 16 + pgvector, Redis 7
- AI: Claude CLI + Ollama (OpenAI compat)

## 개발 규칙
- admin.py: 3668줄, 분할하지 않음. 섹션 주석으로 구분.
- 신규 엔드포인트는 admin.py 맨 끝에 추가
- scope 관련: ScopeService + crud_helpers 사용 (직접 구현 금지)
- Frontend 훅: useScopedQuery, useScopedCRUD 사용
- 문서: docs/ 디렉토리의 Phase별 문서 참조

## 주요 파일
- app/api/admin.py — 모든 API 엔드포인트
- app/services/pipeline.py — 메일 처리 파이프라인
- app/services/ai_processor.py — AI 엔진 라우팅
- app/services/auth.py — 인증/권한
- app/models/base.py — DB 모델
- admin/src/components/app-sidebar.tsx — 사이드바 네비게이션

## 문서
- docs/implementation-plan.md — 전체 개요
- docs/dev-framework.md — 토큰 절약 개발 프레임워크
- docs/shared-infrastructure.md — 공유 인프라 패턴
```
