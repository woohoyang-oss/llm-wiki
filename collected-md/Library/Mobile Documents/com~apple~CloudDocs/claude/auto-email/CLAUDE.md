# CLAUDE.md — Email Support Bot

## 프로젝트
Email Support Bot — 회사 이메일 자동/반자동 처리 시스템

## 기술 스택
- Backend: Python 3.12, FastAPI (port 8000), Celery, SQLAlchemy (async)
- Frontend: Next.js 14 (port 3000), shadcn/ui, TanStack Query
- DB: PostgreSQL 16 + pgvector (localhost:5432), Redis 7 (localhost:6379)
- AI: Claude CLI (MCP tool_use) + Ollama 로컬 LLM (OpenAI compat)

## 실행
```bash
# API 서버
uvicorn app.main:app --reload --port 8000

# Frontend
cd admin && npm run dev

# Celery Worker
celery -A app.workers.celery_app worker --loglevel=info -Q email_processing,email_polling

# Celery Beat
celery -A app.workers.celery_app beat --loglevel=info
```

## 개발 규칙
- `app/api/admin.py`: 3668줄 단일 파일. **분할하지 않음**. 섹션 주석으로 구분.
- 신규 엔드포인트는 `admin.py` 맨 끝에 추가 (워크트리 충돌 방지)
- scope 관련 로직: `ScopeService` + `crud_helpers` 사용 (직접 쿼리 금지)
- Frontend 공용 훅: `useScopedQuery`, `useScopedCRUD`, `useEntityPermissions` 사용
- Frontend 공용 컴포넌트: `ScopeTabs`, `RoleGuard` 사용
- `queries.ts`/`mutations.ts` 비대화 방지: 신규 기능은 별도 훅 파일로 분리

## 핵심 파일
| 파일 | 역할 |
|------|------|
| `app/api/admin.py` | 모든 API 엔드포인트 (65+개) |
| `app/services/pipeline.py` | 메일 수신 → AI → 발송 파이프라인 |
| `app/services/ai_processor.py` | AI 엔진 라우팅 (Claude + 로컬 LLM) |
| `app/services/auth.py` | JWT + X-Admin-Key 인증, require_role |
| `app/models/base.py` | SQLAlchemy 모델 전체 |
| `app/workers/tasks.py` | Celery 비동기 태스크 |
| `admin/src/components/app-sidebar.tsx` | 사이드바 네비게이션 |
| `admin/src/lib/queries.ts` | TanStack Query 훅 |
| `admin/src/lib/mutations.ts` | TanStack Mutation 훅 |

## 문서 (docs/)
| 문서 | 내용 |
|------|------|
| `implementation-plan.md` | 전체 5 Phase 개요 |
| `phase1-auto-pipeline.md` | Phase 1: 자동 파이프라인 |
| `phase2-notification-assignment.md` | Phase 2: 알림 + 자동배정 |
| `phase3-rbac-permissions.md` | Phase 3: RBAC 권한 |
| `phase4-personal-workspace.md` | Phase 4: 개인 워크스페이스 |
| `phase5-dashboard-analytics.md` | Phase 5: 대시보드 + 통계 |
| `shared-infrastructure.md` | 공유 인프라 패턴 (ScopeMixin 등) |
| `dev-framework.md` | 토큰 절약 개발 프레임워크 |
| `ux-ui-guidelines.md` | Gmail 친숙 UX/UI 가이드라인 |
| `parallel-dev-guide.md` | 병렬 개발 + 로컬 LLM 가이드 |
| `workflow-specs.md` | 9개 워크플로우 상세 |
| `migration-guide.md` | DB 마이그레이션 가이드 |

## AI 엔진 아키텍처
- `AIRouter` (ai_processor.py): 룰별 engine_id → 해당 엔진 디스패치
- `ClaudeCLIProcessor`: Claude API + MCP tool_use (실시간 DB 조회)
- `LocalLLMProcessor`: Ollama/OpenAI compat (지식+컨텍스트를 프롬프트 주입)
- DB 모델: `LLMEngine` (llm_engines 테이블) — Admin UI에서 등록/관리
- Confidence 라우팅: ≥0.85 자동발송, 0.70~0.84 승인큐, <0.70 에스컬레이션

## 워크트리 병렬 개발
- Phase 0 (공유 인프라) → main에서 직접
- Phase 1, 3, 4-FE → 동시 워크트리 가능
- Phase 2 → Phase 1 merge 후
- Phase 5 → Phase 3 merge 후
- 상세: `docs/parallel-dev-guide.md` 참조
