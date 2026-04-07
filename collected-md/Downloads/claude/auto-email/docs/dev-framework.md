# 개발 프레임워크 — Claude 토큰 절약형 구조

> 작성일: 2026-03-13
> 목적: 각 Phase/태스크 구현 시 Claude 컨텍스트를 최소화하면서 일관된 품질 유지

---

## 1. 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Single Source of Truth** | shared-infrastructure.md의 패턴을 모든 엔티티에 동일 적용 |
| **Template-First** | 새 파일 생성 시 아래 템플릿 복사 → 엔티티명만 교체 |
| **참조 최소화** | 각 태스크는 필요한 파일 2~3개만 읽으면 구현 가능하도록 설계 |
| **독립 태스크** | 태스크 간 의존성 최소화, 병렬 구현 가능 |

---

## 2. 태스크 실행 프로토콜

### 2.1 Claude에게 태스크 지시 시 포맷

```
태스크: [Phase X - Task Y] {태스크 제목}

참조 파일:
- docs/{해당 phase 문서}.md → 태스크 Y 섹션만
- docs/shared-infrastructure.md → 섹션 {N}만

수정 대상:
- {파일 경로 1}: {무엇을 추가/수정}
- {파일 경로 2}: {무엇을 추가/수정}

기존 패턴 참조:
- {이미 구현된 유사 파일}: {이 파일의 패턴을 따라서}
```

### 2.2 예시: "서명 페이지에 scope 탭 추가"

```
태스크: [Phase 4 - Task 4-8] 서명 페이지 scope 탭 추가

참조 파일:
- docs/shared-infrastructure.md → 섹션 2.1 (ScopeTabs)

수정 대상:
- admin/src/app/signatures/page.tsx: ScopeTabs 래핑 + scope 쿼리 파라미터

기존 패턴 참조:
- admin/src/app/rules/page.tsx: 이미 scope 탭이 적용된 첫 번째 페이지
```

---

## 3. 백엔드 코드 템플릿

### 3.1 Scope 적용 API 엔드포인트 (9개 엔티티 공통)

> 이 템플릿은 rules, repositories, knowledge_docs, email_templates,
> canned_responses, signatures, disclaimers, notification_channels,
> notification_routing 모두에 동일 적용

```python
# ── {entity_name} scope 엔드포인트 ──────────────────────────
# 파일: app/api/admin.py 내 해당 섹션에 추가

from app.services.scope_service import ScopeService
from app.services.crud_helpers import list_with_scope, create_with_scope, update_with_scope, delete_with_scope
from app.services.auth import get_current_user_optional

@router.get("/api/{entity_plural}")
async def list_{entity_plural}(
    scope: str = Query("all", regex="^(all|shared|personal)$"),
    current_user=Depends(get_current_user_optional),
    db: AsyncSession = Depends(get_db),
):
    return await list_with_scope(
        model={ModelClass},
        db=db,
        user_id=current_user.id if current_user else None,
        scope=scope,
    )

@router.post("/api/{entity_plural}")
async def create_{entity_name}(
    data: {CreateSchema},
    scope: str = Query("shared", regex="^(shared|personal)$"),
    current_user=Depends(get_current_user_optional),
    db: AsyncSession = Depends(get_db),
):
    return await create_with_scope(
        model={ModelClass},
        db=db,
        data=data.dict(),
        user_id=current_user.id if current_user else None,
        user_role=current_user.role if current_user else "admin",
        scope=scope,
    )

@router.put("/api/{entity_plural}/{id}")
async def update_{entity_name}(
    id: int,
    data: {UpdateSchema},
    current_user=Depends(get_current_user_optional),
    db: AsyncSession = Depends(get_db),
):
    return await update_with_scope(
        model={ModelClass},
        db=db,
        item_id=id,
        data=data.dict(exclude_unset=True),
        user_id=current_user.id if current_user else None,
        user_role=current_user.role if current_user else "admin",
    )

@router.delete("/api/{entity_plural}/{id}")
async def delete_{entity_name}(
    id: int,
    current_user=Depends(get_current_user_optional),
    db: AsyncSession = Depends(get_db),
):
    return await delete_with_scope(
        model={ModelClass},
        db=db,
        item_id=id,
        user_id=current_user.id if current_user else None,
        user_role=current_user.role if current_user else "admin",
    )
```

**적용 방법**: 위 템플릿에서 `{entity_plural}`, `{entity_name}`, `{ModelClass}`, `{CreateSchema}`, `{UpdateSchema}`만 교체.

### 3.2 엔티티별 치환표

| 엔티티 | entity_plural | entity_name | ModelClass | 비고 |
|--------|---------------|-------------|------------|------|
| 룰 | rules | rule | Rule | 기존 CRUD 있음, scope 파라미터만 추가 |
| 레포지토리 | repositories | repository | KnowledgeRepository | 기존 CRUD 있음 |
| 지식 문서 | knowledge-docs | knowledge_doc | KnowledgeDocument | 기존 CRUD 있음 |
| 템플릿 | email-templates | email_template | EmailTemplate | 기존 CRUD 있음 |
| 매크로 | macros | macro | CannedResponse | 모델명 주의 |
| 서명 | signatures | signature | EmailSignature | 기존 CRUD 있음 |
| 고지문 | disclaimers | disclaimer | AIDisclaimer | 기존 CRUD 있음 |
| 알림 채널 | notification-channels | notification_channel | NotificationChannel | 기존 CRUD 있음 |
| 알림 라우팅 | notification-routing | notification_routing | NotificationRouting | 기존 CRUD 있음 |

---

## 4. 프론트엔드 코드 템플릿

### 4.1 Scope 적용 페이지 (9개 엔티티 공통)

```tsx
// admin/src/app/{entity-plural}/page.tsx
// ── scope 탭 적용 패턴 ──────────────────────────

"use client";

import { ScopeTabs } from "@/components/scope-tabs";
import { useScopedQuery } from "@/hooks/use-scoped-query";
import { useEntityPermissions } from "@/hooks/use-entity-permissions";

export default function {Entity}Page() {
  const [scope, setScope] = useState<"all" | "shared" | "personal">("all");
  const { data, isLoading } = useScopedQuery("{entity-plural}", scope);
  const permissions = useEntityPermissions(scope);

  return (
    <div className="flex flex-col gap-4 p-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">{페이지 제목}</h1>
        {permissions.canCreate && (
          <Button onClick={() => setOpen(true)}>
            <Plus className="mr-2 h-4 w-4" /> 추가
          </Button>
        )}
      </div>

      <ScopeTabs value={scope} onChange={setScope} />

      {/* 기존 리스트 테이블 — data 소스만 변경 */}
      <DataTable
        data={data ?? []}
        loading={isLoading}
        permissions={permissions}
        // ... columns, actions
      />
    </div>
  );
}
```

### 4.2 공통 훅 시그니처 (1회 구현, 9곳 재사용)

```tsx
// hooks/use-scoped-query.ts
export function useScopedQuery(entityKey: string, scope: string) {
  return useQuery({
    queryKey: [entityKey, { scope }],
    queryFn: () => apiClient.get(`/api/${entityKey}?scope=${scope}`),
  });
}

// hooks/use-scoped-crud.ts
export function useScopedCRUD(entityKey: string, scope: string) {
  const qc = useQueryClient();
  return {
    create: useMutation({
      mutationFn: (data) => apiClient.post(`/api/${entityKey}?scope=${scope}`, data),
      onSuccess: () => qc.invalidateQueries({ queryKey: [entityKey] }),
    }),
    update: useMutation({
      mutationFn: ({ id, ...data }) => apiClient.put(`/api/${entityKey}/${id}`, data),
      onSuccess: () => qc.invalidateQueries({ queryKey: [entityKey] }),
    }),
    remove: useMutation({
      mutationFn: (id) => apiClient.delete(`/api/${entityKey}/${id}`),
      onSuccess: () => qc.invalidateQueries({ queryKey: [entityKey] }),
    }),
  };
}

// hooks/use-entity-permissions.ts
export function useEntityPermissions(scope: string) {
  const { user } = useAuth();
  return {
    canCreate: user?.role === "admin" || scope === "personal",
    canEdit: (item) => user?.role === "admin" || (scope === "personal" && item.owner_id === user?.id),
    canDelete: (item) => user?.role === "admin" || (scope === "personal" && item.owner_id === user?.id),
  };
}

// components/scope-tabs.tsx
export function ScopeTabs({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return (
    <Tabs value={value} onValueChange={onChange}>
      <TabsList>
        <TabsTrigger value="all">전체</TabsTrigger>
        <TabsTrigger value="shared">공용</TabsTrigger>
        <TabsTrigger value="personal">내 설정</TabsTrigger>
      </TabsList>
    </Tabs>
  );
}
```

---

## 5. 태스크별 필요 컨텍스트 맵

> Claude가 각 태스크를 실행할 때 읽어야 할 파일 목록.
> 이 맵을 따르면 불필요한 파일 읽기를 최소화함.

### Phase 1: 자동 파이프라인

| 태스크 | 읽을 파일 | 수정할 파일 |
|--------|-----------|-------------|
| 1-1. DB 스키마 | phase1 §1-1, migration-guide.md | alembic/versions/새파일 |
| 1-2. DatabaseScheduler | phase1 §1-2, app/workers/celery_app.py | app/workers/scheduler.py (신규) |
| 1-3. 폴링 태스크 | phase1 §1-3, app/workers/tasks.py | app/workers/tasks.py |
| 1-4. Admin API | phase1 §1-4, app/api/admin.py (스케줄 섹션만) | app/api/admin.py |
| 1-5. Admin UI | phase1 §1-5, admin/src/app/mailboxes/page.tsx | admin/src/app/mailboxes/page.tsx |

### Phase 2: 알림 + 자동배정

| 태스크 | 읽을 파일 | 수정할 파일 |
|--------|-----------|-------------|
| 2-1. DB 스키마 | phase2 §2-1 | alembic/versions/새파일 |
| 2-2. AssignmentService | phase2 §2-2 | app/services/assignment.py (신규) |
| 2-3. 알림 확장 | phase2 §2-3, app/services/notification.py | app/services/notification.py |
| 2-4. pipeline 통합 | phase2 §2-4, app/services/pipeline.py | app/services/pipeline.py |
| 2-5. Admin API | phase2 §2-5 | app/api/admin.py |
| 2-6. Admin UI | phase2 §2-6 | admin/src/app/notifications/page.tsx |

### Phase 3: RBAC

| 태스크 | 읽을 파일 | 수정할 파일 |
|--------|-----------|-------------|
| 3-1. require_role 수정 | phase3 §3-1, shared-infra §1.5 | app/services/auth.py |
| 3-2. 엔드포인트 적용 | phase3 §3-2 (엔드포인트 목록) | app/api/admin.py |
| 3-3. Frontend RoleGuard | phase3 §3-3, shared-infra §2.2-2.3 | admin/src/components/ (3개 신규) |

### Phase 4: 개인 워크스페이스

| 태스크 | 읽을 파일 | 수정할 파일 |
|--------|-----------|-------------|
| 4-1. DB 마이그레이션 | shared-infra §3 | alembic/versions/새파일 |
| 4-2. ScopeService | shared-infra §1.2 | app/services/scope_service.py (신규) |
| 4-3. crud_helpers | shared-infra §1.3 | app/services/crud_helpers.py (신규) |
| 4-4. API scope 확장 (9개) | 이 문서 §3.1-3.2, app/api/admin.py (해당 섹션) | app/api/admin.py |
| 4-5. 개인설정 API | phase4 §4-4 | app/api/admin.py |
| 4-6. KnowledgeAccumulator | phase4 §4-5 | app/services/knowledge_accumulator.py (신규) |
| 4-7. Frontend scope 탭 (9개) | 이 문서 §4.1, 이미 완성된 첫 페이지 참조 | admin/src/app/{9 pages} |
| 4-8. 개인설정 UI | phase4 §4-9 | admin/src/app/my-inbox/settings/page.tsx (신규) |

### Phase 5: 대시보드 + 통계

| 태스크 | 읽을 파일 | 수정할 파일 |
|--------|-----------|-------------|
| 5-1. 대시보드 API 분기 | phase5 §5-1 | app/api/admin.py (dashboard 섹션) |
| 5-2. 통계 API 확장 | phase5 §5-2 | app/api/admin.py (statistics 섹션) |
| 5-3. Frontend 대시보드 | phase5 §5-3, admin/src/app/dashboard/page.tsx | admin/src/app/dashboard/page.tsx |
| 5-4. Frontend 통계 | phase5 §5-4, admin/src/app/statistics/page.tsx | admin/src/app/statistics/page.tsx |

---

## 6. "첫 번째 구현 → 나머지 복사" 전략

### 6.1 Backend scope 적용

1. **첫 번째: rules** — scope_service.py + crud_helpers.py + rules 엔드포인트 수정 (가장 로직 많음)
2. **나머지 8개**: rules 패턴 복사 → 치환표(§3.2) 기반 엔티티명만 변경

### 6.2 Frontend scope 탭 적용

1. **첫 번째: rules/page.tsx** — ScopeTabs + useScopedQuery + useEntityPermissions 통합
2. **나머지 8개**: rules/page.tsx에서 import 패턴 복사 → 엔티티명만 변경

### 6.3 효과

| 항목 | 첫 번째 | 나머지 8개 (각각) |
|------|---------|-------------------|
| 읽을 파일 수 | 4~5개 | 2개 (이 문서 + 첫 번째 구현 파일) |
| 예상 토큰 | ~15K | ~3K |
| 구현 시간 | 15분 | 3분 |

---

## 7. 코드 생성 체크리스트

### Backend 체크리스트
- [ ] ScopeMixin 적용 (모델에 scope, owner_id 추가)
- [ ] ScopeService.filtered_query 사용
- [ ] crud_helpers 4개 함수 사용
- [ ] get_current_user_optional 의존성 주입
- [ ] scope Query 파라미터 추가 (list/create)
- [ ] 에러 응답: 403 Forbidden (권한 없음), 404 Not Found (존재하지 않음)

### Frontend 체크리스트
- [ ] ScopeTabs 컴포넌트 사용
- [ ] useScopedQuery 훅 사용
- [ ] useEntityPermissions로 버튼 표시/숨김
- [ ] useScopedCRUD로 생성/수정/삭제
- [ ] RoleGuard로 admin 전용 UI 감싸기
- [ ] 쿼리 키에 scope 포함 (캐시 분리)

---

## 8. 파일 구조 (Phase 4 완료 후 예상)

```
app/
  services/
    scope_service.py      ← 신규: 1회 구현, 9곳 사용
    crud_helpers.py        ← 신규: 1회 구현, 9곳 사용
    knowledge_accumulator.py ← 신규
    assignment.py          ← 신규 (Phase 2)
    pipeline.py            ← 수정
    notification.py        ← 수정
    auth.py                ← 수정 (require_role)
  api/
    admin.py               ← 수정 (scope 파라미터 추가)
  models/
    *.py                   ← 수정 (ScopeMixin 추가)

admin/src/
  components/
    scope-tabs.tsx         ← 신규: 1회 구현, 9곳 사용
    role-guard.tsx         ← 신규: 1회 구현
  hooks/
    use-scoped-query.ts    ← 신규: 1회 구현, 9곳 사용
    use-scoped-crud.ts     ← 신규: 1회 구현, 9곳 사용
    use-entity-permissions.ts ← 신규: 1회 구현, 9곳 사용
  app/
    rules/page.tsx         ← 수정 (scope 탭 패턴 첫 번째 적용)
    repositories/page.tsx  ← 수정 (rules 패턴 복사)
    knowledge-docs/page.tsx ← 수정
    email-templates/page.tsx ← 수정
    macros/page.tsx        ← 수정
    signatures/page.tsx    ← 수정
    disclaimers/page.tsx   ← 수정
    notifications/page.tsx ← 수정
```

---

## 9. 주의사항

1. **admin.py 분할 금지**: 현재 3668줄이지만 기존 import/라우터 구조 유지. 섹션 주석으로 구분.
2. **기존 테스트 깨뜨리지 않기**: scope 파라미터는 기본값 `"all"`이므로 기존 호출 호환.
3. **마이그레이션 순서**: shared-infra §3 마이그레이션 먼저 → 그 다음 Phase별 마이그레이션.
4. **프론트엔드 훅 파일 분리**: queries.ts/mutations.ts에 추가하지 말고 별도 훅 파일로. 기존 338줄/960줄 파일 비대화 방지.
