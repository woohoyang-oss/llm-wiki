# 공용 인프라 — 중복 없는 단일 소스 개발 가이드

> 작성일: 2026-03-13
> 목적: 모든 Phase에서 공용으로 사용하는 패턴, 서비스, 컴포넌트를 한 곳에 정의.
> **이 문서에 정의된 코드를 먼저 구현하면, 각 Phase는 이를 import하여 사용만 하면 됨.**

---

## 0. 핵심 원칙

```
1. 같은 패턴은 한 번만 구현, import하여 재사용
2. 9개 엔티티(룰/레포/지식/템플릿/매크로/서명/고지문/알림채널/알림라우팅)의
   CRUD + scope + 권한은 동일 패턴 → 공용 서비스/컴포넌트로 처리
3. 수정 시 한 곳만 고치면 전체 반영
```

---

## 1. 백엔드 공용 서비스

### 1.1 ScopeMixin — DB 모델 공용 컬럼

**파일:** `app/models/mixins.py` (신규)

```python
"""
9개 엔티티에 공통 적용되는 scope + owner_id 컬럼.
각 모델에서 이 Mixin을 상속하면 됨.
"""
from sqlalchemy import Column, String, Integer, ForeignKey, CheckConstraint


class ScopeMixin:
    """
    scope = 'shared': 공용 (owner_id=NULL). admin만 CRUD.
    scope = 'personal': 개인 (owner_id=로그인유저). 본인만 CRUD.
    """
    scope = Column(
        String(10),
        nullable=False,
        default='shared',
        server_default='shared',
        index=True,
    )
    owner_id = Column(
        Integer,
        ForeignKey('agents.id', ondelete='CASCADE'),
        nullable=True,
        index=True,
    )

    @classmethod
    def scope_constraints(cls, table_name: str):
        """__table_args__에 추가할 제약조건 반환"""
        return (
            CheckConstraint(
                "(scope = 'shared' AND owner_id IS NULL) OR "
                "(scope = 'personal' AND owner_id IS NOT NULL)",
                name=f'chk_{table_name}_scope_owner',
            ),
        )
```

**적용 방법 (각 모델에서):**
```python
# app/models/base.py — Rule 모델 예시
class Rule(Base, ScopeMixin):
    __tablename__ = 'rules'
    __table_args__ = (
        *ScopeMixin.scope_constraints('rules'),
    )
    # ... 기존 컬럼 유지
```

**적용 대상 9개 모델:**
| 모델 | base.py 라인 | __tablename__ |
|------|-------------|---------------|
| Rule | 148 | rules |
| KnowledgeRepository | 384 | knowledge_repositories |
| KnowledgeDocument | 401 | knowledge_documents |
| EmailTemplate | 464 | email_templates |
| CannedResponse | 265 | canned_responses |
| EmailSignature | 284 | email_signatures |
| AIDisclaimer | 340 | ai_disclaimers |
| NotificationChannel | 420 | notification_channels |
| NotificationRouting | 432 | notification_routing |

---

### 1.2 ScopeService — 조회/권한 필터 공용 서비스

**파일:** `app/services/scope_service.py` (신규)

```python
"""
모든 scope 대상 엔티티의 조회/수정 권한을 통합 관리.
각 API 엔드포인트에서 이 서비스를 호출하면 됨.
"""
from fastapi import HTTPException
from sqlalchemy import select, and_, or_
from sqlalchemy.ext.asyncio import AsyncSession


class ScopeService:

    @staticmethod
    def filtered_query(query, model, user_id: int | None, scope: str = 'all'):
        """
        GET 조회 시 scope 기반 필터 적용.

        Args:
            query: SQLAlchemy select() 쿼리
            model: ScopeMixin 상속 모델 클래스
            user_id: 로그인 유저 ID (None이면 미인증)
            scope: 'all' | 'shared' | 'personal'

        Returns:
            필터 적용된 쿼리

        사용 예:
            query = select(Rule).filter(Rule.is_active == True)
            query = ScopeService.filtered_query(query, Rule, user.id, scope='all')
        """
        if scope == 'shared':
            return query.filter(model.scope == 'shared')
        elif scope == 'personal':
            if user_id is None:
                raise HTTPException(401, "로그인이 필요합니다.")
            return query.filter(model.scope == 'personal', model.owner_id == user_id)
        else:  # 'all' — 공용 + 내 개인
            if user_id is None:
                return query.filter(model.scope == 'shared')
            return query.filter(
                or_(
                    model.scope == 'shared',
                    and_(model.scope == 'personal', model.owner_id == user_id)
                )
            )

    @staticmethod
    def check_create_permission(scope: str, user_role: str):
        """
        POST 생성 권한 확인.

        규칙:
          - shared 생성: admin만
          - personal 생성: admin, agent

        Raises:
            HTTPException 403 if not allowed
        """
        if scope == 'shared' and user_role != 'admin':
            raise HTTPException(403, "공용 항목 생성은 admin만 가능합니다.")
        if scope == 'personal' and user_role not in ('admin', 'agent'):
            raise HTTPException(403, "viewer는 항목을 생성할 수 없습니다.")

    @staticmethod
    def check_modify_permission(entity, user_id: int, user_role: str):
        """
        PUT/DELETE 수정/삭제 권한 확인.

        규칙:
          - shared 수정: admin만
          - personal 수정: owner 본인만 (admin 포함)

        Args:
            entity: DB에서 조회한 엔티티 객체 (scope, owner_id 필드 필요)

        Raises:
            HTTPException 403/404
        """
        if entity is None:
            raise HTTPException(404, "항목을 찾을 수 없습니다.")

        if entity.scope == 'shared':
            if user_role != 'admin':
                raise HTTPException(403, "공용 항목 수정은 admin만 가능합니다.")
        else:  # personal
            if entity.owner_id != user_id:
                raise HTTPException(403, "다른 사용자의 항목은 수정할 수 없습니다.")

    @staticmethod
    def set_scope_fields(data: dict, user_id: int) -> dict:
        """
        POST 생성 시 scope/owner_id 자동 설정.

        data에 scope가 없으면 'shared' 기본값.
        scope='personal'이면 owner_id를 로그인 유저로 설정.
        scope='shared'이면 owner_id=None.

        Returns:
            scope/owner_id가 설정된 data dict
        """
        scope = data.get('scope', 'shared')
        data['scope'] = scope
        data['owner_id'] = user_id if scope == 'personal' else None
        return data
```

---

### 1.3 GenericCRUDRouter — API 엔드포인트 공용 패턴

**파일:** `app/api/crud_helpers.py` (신규)

```python
"""
9개 엔티티의 CRUD API 패턴을 함수로 추출.
각 엔드포인트에서 이 함수들을 호출하면 동일한 로직이 적용됨.
"""
from fastapi import Query, Body, HTTPException
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from app.services.scope_service import ScopeService


async def list_with_scope(
    model,
    db: AsyncSession,
    user_id: int | None,
    user_role: str,
    scope: str = 'all',
    filters: list = None,
    order_by=None,
    limit: int = 100,
    offset: int = 0,
) -> dict:
    """
    공용 LIST 패턴. 모든 엔티티에 동일하게 적용.

    Returns:
        {"items": [...], "total": int}
    """
    query = select(model)

    # scope 필터
    query = ScopeService.filtered_query(query, model, user_id, scope)

    # 추가 필터
    if filters:
        for f in filters:
            query = query.filter(f)

    # 카운트
    count_q = select(func.count()).select_from(query.subquery())
    total = (await db.execute(count_q)).scalar() or 0

    # 정렬
    if order_by is not None:
        query = query.order_by(order_by)

    # 페이지네이션
    query = query.limit(limit).offset(offset)

    result = await db.execute(query)
    items = result.scalars().all()

    return {"items": items, "total": total}


async def create_with_scope(
    model,
    data: dict,
    db: AsyncSession,
    user_id: int,
    user_role: str,
) -> object:
    """공용 CREATE 패턴."""
    scope = data.get('scope', 'shared')

    # 권한 확인
    ScopeService.check_create_permission(scope, user_role)

    # scope 필드 설정
    data = ScopeService.set_scope_fields(data, user_id)

    entity = model(**data)
    db.add(entity)
    await db.flush()
    await db.refresh(entity)
    return entity


async def update_with_scope(
    model,
    entity_id: int,
    data: dict,
    db: AsyncSession,
    user_id: int,
    user_role: str,
) -> object:
    """공용 UPDATE 패턴."""
    entity = await db.get(model, entity_id)

    # 권한 확인
    ScopeService.check_modify_permission(entity, user_id, user_role)

    # scope/owner_id는 변경 불가
    data.pop('scope', None)
    data.pop('owner_id', None)

    for key, value in data.items():
        if hasattr(entity, key):
            setattr(entity, key, value)

    await db.flush()
    await db.refresh(entity)
    return entity


async def delete_with_scope(
    model,
    entity_id: int,
    db: AsyncSession,
    user_id: int,
    user_role: str,
) -> dict:
    """공용 DELETE 패턴."""
    entity = await db.get(model, entity_id)

    # 권한 확인
    ScopeService.check_modify_permission(entity, user_id, user_role)

    await db.delete(entity)
    return {"ok": True}
```

**API 엔드포인트에서 사용 예시:**
```python
# app/api/admin.py — 룰 목록 (기존 코드 대체)

from app.api.crud_helpers import list_with_scope, create_with_scope, update_with_scope, delete_with_scope
from app.services.auth import get_current_user_optional

@router.get("/rules")
async def list_rules(
    scope: str = Query('all', pattern='^(all|shared|personal)$'),
    limit: int = Query(100, ge=1, le=500),
    offset: int = Query(0, ge=0),
    db: AsyncSession = Depends(get_db),
    user = Depends(get_current_user_optional),
):
    return await list_with_scope(
        model=Rule,
        db=db,
        user_id=user.id if user else None,
        user_role=user.role if user else 'viewer',
        scope=scope,
        filters=[Rule.is_active == True],
        order_by=Rule.priority.asc(),
        limit=limit,
        offset=offset,
    )

@router.post("/rules")
async def create_rule(
    data: dict = Body(...),
    db: AsyncSession = Depends(get_db),
    user: Agent = Depends(get_current_user),
):
    return await create_with_scope(Rule, data, db, user.id, user.role)

@router.put("/rules/{rule_id}")
async def update_rule(
    rule_id: int,
    data: dict = Body(...),
    db: AsyncSession = Depends(get_db),
    user: Agent = Depends(get_current_user),
):
    return await update_with_scope(Rule, rule_id, data, db, user.id, user.role)

@router.delete("/rules/{rule_id}")
async def delete_rule(
    rule_id: int,
    db: AsyncSession = Depends(get_db),
    user: Agent = Depends(get_current_user),
):
    return await delete_with_scope(Rule, rule_id, db, user.id, user.role)
```

**9개 엔티티 모두 위 패턴을 동일하게 적용.** 엔티티별로 다른 것은 `filters`와 `order_by`뿐.

---

### 1.4 get_current_user_optional — 인증 선택적 의존성

**파일:** `app/services/auth.py` — 기존 파일에 추가

```python
# 기존 get_current_user (Line 139) 바로 아래에 추가

async def get_current_user_optional(
    credentials: HTTPAuthorizationCredentials | None = Depends(security),
    x_admin_key: str | None = Header(None, alias="X-Admin-Key"),
    db: AsyncSession = Depends(get_db),
) -> Agent | None:
    """
    인증 선택적. 토큰 있으면 Agent 반환, 없으면 None.
    X-Admin-Key만 있는 경우 → admin 역할의 가상 Agent 반환.

    사용처: GET 엔드포인트에서 scope 필터링 시
    - 인증됨 → shared + 내 personal
    - 미인증 → shared만
    """
    # JWT 토큰 우선
    if credentials:
        try:
            payload = verify_jwt(credentials.credentials)
            agent_id = int(payload["sub"])
            result = await db.execute(select(Agent).where(Agent.id == agent_id))
            agent = result.scalar_one_or_none()
            if agent and agent.is_active:
                return agent
        except Exception:
            pass

    # X-Admin-Key 폴백
    if x_admin_key and x_admin_key == settings.admin_secret_key:
        # API 키 인증은 admin 권한으로 처리
        # 가상 Agent 객체 생성 (DB 조회 없이)
        from types import SimpleNamespace
        return SimpleNamespace(id=0, role='admin', email='api-key', name='API')

    return None
```

---

### 1.5 require_role 수정 — X-Admin-Key 호환

**파일:** `app/services/auth.py` — 기존 require_role (Line 159) 교체

```python
def require_role(*roles: str):
    """
    역할 기반 접근 제어 데코레이터.

    사용법:
        @router.get("/something", dependencies=[Depends(require_role('admin', 'agent'))])

    또는:
        user: Agent = require_role('admin', 'agent')
    """
    async def dependency(
        credentials: HTTPAuthorizationCredentials | None = Depends(security),
        x_admin_key: str | None = Header(None, alias="X-Admin-Key"),
        db: AsyncSession = Depends(get_db),
    ) -> Agent:
        # JWT 토큰
        if credentials:
            payload = verify_jwt(credentials.credentials)
            agent_id = int(payload["sub"])
            result = await db.execute(select(Agent).where(Agent.id == agent_id))
            agent = result.scalar_one_or_none()
            if not agent or not agent.is_active:
                raise HTTPException(401, "비활성화된 계정입니다.")
            if agent.role not in roles:
                raise HTTPException(403, f"권한이 부족합니다. 필요: {', '.join(roles)}")
            return agent

        # X-Admin-Key 폴백 (admin 역할만 허용)
        if x_admin_key and x_admin_key == settings.admin_secret_key:
            if 'admin' in roles:
                from types import SimpleNamespace
                return SimpleNamespace(id=0, role='admin', email='api-key', name='API')
            raise HTTPException(403, "API Key는 admin 전용입니다.")

        raise HTTPException(401, "인증이 필요합니다.")

    return Depends(dependency)
```

---

## 2. 프론트엔드 공용 컴포넌트

### 2.1 ScopeTabs — 공용/개인 전환 탭

**파일:** `admin/src/components/scope-tabs.tsx` (신규)

```tsx
"use client";

import { Tabs, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Badge } from "@/components/ui/badge";

interface ScopeTabsProps {
  value: "shared" | "personal";
  onChange: (scope: "shared" | "personal") => void;
  sharedCount?: number;
  personalCount?: number;
}

/**
 * 모든 엔티티 관리 페이지에서 공용/개인 탭 전환.
 *
 * 사용법:
 *   const [scope, setScope] = useState<"shared" | "personal">("shared");
 *   <ScopeTabs value={scope} onChange={setScope} />
 */
export function ScopeTabs({ value, onChange, sharedCount, personalCount }: ScopeTabsProps) {
  return (
    <Tabs value={value} onValueChange={(v) => onChange(v as "shared" | "personal")}>
      <TabsList>
        <TabsTrigger value="shared">
          공용
          {sharedCount !== undefined && (
            <Badge variant="secondary" className="ml-1.5">{sharedCount}</Badge>
          )}
        </TabsTrigger>
        <TabsTrigger value="personal">
          내 것
          {personalCount !== undefined && (
            <Badge variant="secondary" className="ml-1.5">{personalCount}</Badge>
          )}
        </TabsTrigger>
      </TabsList>
    </Tabs>
  );
}
```

### 2.2 useEntityPermissions — 권한 판단 훅

**파일:** `admin/src/hooks/use-entity-permissions.ts` (신규)

```tsx
import { useAuth } from "@/hooks/use-auth";

interface EntityPermissions {
  canCreate: boolean;
  canEdit: boolean;
  canDelete: boolean;
}

/**
 * 엔티티 CRUD 권한을 반환.
 *
 * 규칙:
 *   shared 엔티티: admin만 수정/삭제
 *   personal 엔티티: owner 본인만 수정/삭제
 *   viewer: 아무것도 수정 불가
 *
 * 사용법:
 *   const { canEdit, canDelete } = useEntityPermissions(item.scope, item.owner_id);
 *   {canEdit && <Button onClick={onEdit}>수정</Button>}
 */
export function useEntityPermissions(
  scope?: string,
  ownerId?: number | null,
): EntityPermissions {
  const { user } = useAuth();
  const role = user?.role || "viewer";
  const userId = user?.id;

  if (role === "viewer") {
    return { canCreate: false, canEdit: false, canDelete: false };
  }

  // 생성: admin은 shared+personal, agent는 personal만
  const canCreate = role === "admin" || scope === "personal";

  // 수정/삭제
  let canEdit = false;
  let canDelete = false;

  if (scope === "shared") {
    canEdit = role === "admin";
    canDelete = role === "admin";
  } else if (scope === "personal") {
    canEdit = ownerId === userId;
    canDelete = ownerId === userId;
  } else {
    // scope 미지정 (목록 레벨) — 생성 권한만 판단
    canEdit = false;
    canDelete = false;
  }

  return { canCreate, canEdit, canDelete };
}
```

### 2.3 RoleGuard — 페이지/컴포넌트 접근 제어

**파일:** `admin/src/components/role-guard.tsx` (신규)

```tsx
"use client";

import { useAuth } from "@/hooks/use-auth";
import { ReactNode } from "react";

type Role = "admin" | "agent" | "viewer";

interface RoleGuardProps {
  /** 접근 허용 역할 */
  allowed: Role[];
  children: ReactNode;
  /** 접근 거부 시 표시 (기본: null — 아무것도 안 보임) */
  fallback?: ReactNode;
}

/**
 * 역할 기반 UI 요소 표시/숨김.
 *
 * 사용법:
 *   <RoleGuard allowed={["admin"]}>
 *     <DeleteButton />
 *   </RoleGuard>
 *
 *   <RoleGuard allowed={["admin", "agent"]} fallback={<p>권한 없음</p>}>
 *     <EditForm />
 *   </RoleGuard>
 */
export function RoleGuard({ allowed, children, fallback = null }: RoleGuardProps) {
  const { user } = useAuth();
  const role = (user?.role || "viewer") as Role;

  if (!allowed.includes(role)) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}
```

### 2.4 useScopedQuery — scope 파라미터 포함 쿼리 훅 팩토리

**파일:** `admin/src/hooks/use-scoped-query.ts` (신규)

```tsx
import { useQuery } from "@tanstack/react-query";
import { apiClient } from "@/lib/api-client";

interface ScopedQueryOptions {
  /** API 경로 (예: "/admin/rules") */
  endpoint: string;
  /** React Query 키 (예: "rules") */
  queryKey: string;
  /** scope 필터 */
  scope?: "all" | "shared" | "personal";
  /** 추가 쿼리 파라미터 */
  params?: Record<string, string | number | undefined>;
  /** staleTime (기본: 10_000) */
  staleTime?: number;
}

/**
 * scope 파라미터를 자동으로 포함하는 useQuery 래퍼.
 * 9개 엔티티에 동일하게 사용.
 *
 * 사용법:
 *   const { data } = useScopedQuery({
 *     endpoint: "/admin/rules",
 *     queryKey: "rules",
 *     scope: "shared",
 *   });
 */
export function useScopedQuery<T = unknown>({
  endpoint,
  queryKey,
  scope = "all",
  params = {},
  staleTime = 10_000,
}: ScopedQueryOptions) {
  const allParams = { scope, ...params };
  const qs = new URLSearchParams(
    Object.entries(allParams)
      .filter(([, v]) => v !== undefined && v !== null && v !== "")
      .map(([k, v]) => [k, String(v)])
  ).toString();

  return useQuery<T>({
    queryKey: [queryKey, scope, params],
    queryFn: () => apiClient<T>(`${endpoint}?${qs}`),
    staleTime,
  });
}
```

### 2.5 useScopedMutation — scope 대응 mutation 팩토리

**파일:** `admin/src/hooks/use-scoped-mutation.ts` (신규)

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { apiClient } from "@/lib/api-client";

interface ScopedMutationOptions {
  endpoint: string;
  queryKey: string;
  additionalInvalidateKeys?: string[];
}

/**
 * CRUD mutation 세트를 한 번에 생성.
 *
 * 사용법:
 *   const { create, update, remove } = useScopedCRUD({
 *     endpoint: "/admin/rules",
 *     queryKey: "rules",
 *   });
 *   create.mutate({ name: "New Rule", scope: "personal" });
 *   update.mutate({ id: 1, data: { name: "Updated" } });
 *   remove.mutate(1);
 */
export function useScopedCRUD({ endpoint, queryKey, additionalInvalidateKeys = [] }: ScopedMutationOptions) {
  const qc = useQueryClient();
  const invalidate = () => {
    qc.invalidateQueries({ queryKey: [queryKey] });
    additionalInvalidateKeys.forEach((k) => qc.invalidateQueries({ queryKey: [k] }));
  };

  const create = useMutation({
    mutationFn: (data: Record<string, unknown>) =>
      apiClient(endpoint, { method: "POST", body: JSON.stringify(data) }),
    meta: { successMessage: "생성되었습니다." },
    onSuccess: invalidate,
  });

  const update = useMutation({
    mutationFn: ({ id, data }: { id: number | string; data: Record<string, unknown> }) =>
      apiClient(`${endpoint}/${id}`, { method: "PUT", body: JSON.stringify(data) }),
    meta: { successMessage: "수정되었습니다." },
    onSuccess: invalidate,
  });

  const remove = useMutation({
    mutationFn: (id: number | string) =>
      apiClient(`${endpoint}/${id}`, { method: "DELETE" }),
    meta: { successMessage: "삭제되었습니다." },
    onSuccess: invalidate,
  });

  return { create, update, remove };
}
```

### 2.6 사이드바 역할별 필터링

**파일:** `admin/src/components/app-sidebar.tsx` — 기존 수정

```tsx
// 각 navGroup item에 roles 필드 추가
interface NavItem {
  title: string;
  href: string;
  icon: LucideIcon;
  badgeKey?: string;
  roles?: ("admin" | "agent" | "viewer")[]; // 없으면 전체 허용
}

// 메뉴 정의에 roles 추가
const navGroups = [
  {
    label: "개요",
    items: [
      { title: "대시보드", href: "/dashboard", icon: LayoutDashboard },
      { title: "통계", href: "/statistics", icon: BarChart3 },
    ],
  },
  {
    label: "이메일 처리",
    items: [
      { title: "내 메일함", href: "/my-inbox", icon: Inbox, roles: ["admin", "agent"] },
      { title: "승인 대기", href: "/approvals", icon: CheckCircle, badgeKey: "pending_approvals", roles: ["admin", "agent"] },
      { title: "공용 메일함", href: "/emails", icon: Mail, badgeKey: "unreplied_count" },
    ],
  },
  {
    label: "자동화 설정",
    items: [
      { title: "룰 관리", href: "/rules", icon: Zap },
      { title: "자동화", href: "/automations", icon: Bot },
      { title: "배정 규칙", href: "/assignment-rules", icon: Users, roles: ["admin"] },
      { title: "스케줄", href: "/schedules", icon: Clock, roles: ["admin"] },
    ],
  },
  {
    label: "지식 베이스",
    items: [
      { title: "레포지토리", href: "/repositories", icon: Database },
      { title: "지식 문서", href: "/knowledge-docs", icon: FileText },
      { title: "템플릿", href: "/email-templates", icon: FileCode },
      { title: "매크로", href: "/macros", icon: Hash },
    ],
  },
  {
    label: "채널 & 설정",
    items: [
      { title: "메일박스", href: "/mailboxes", icon: AtSign, roles: ["admin"] },
      { title: "서명", href: "/signatures", icon: PenTool },
      { title: "고지문", href: "/disclaimers", icon: Shield },
      { title: "알림 채널", href: "/notifications", icon: Bell },
    ],
  },
  {
    label: "시스템",
    items: [
      { title: "담당자", href: "/agents", icon: UserCog, roles: ["admin"] },
      { title: "LLM 엔진", href: "/engines", icon: Cpu, roles: ["admin"] },
      { title: "시스템", href: "/system", icon: Settings, roles: ["admin"] },
    ],
  },
];

// 렌더링 시 필터
const { user } = useAuth();
const userRole = user?.role || "viewer";

{navGroups.map((group) => {
  const visibleItems = group.items.filter(
    (item) => !item.roles || item.roles.includes(userRole)
  );
  if (visibleItems.length === 0) return null;
  // ... 렌더
})}
```

---

## 3. DB 마이그레이션 공용 SQL

**파일:** `scripts/migration_shared.sql` (신규)

```sql
-- ================================================================
-- 공용 인프라 마이그레이션 (모든 Phase 시작 전 1회 실행)
-- ================================================================

-- 1. 9개 테이블에 scope + owner_id 추가
DO $$
DECLARE
  tbl TEXT;
BEGIN
  FOR tbl IN
    SELECT unnest(ARRAY[
      'rules', 'knowledge_repositories', 'knowledge_documents',
      'email_templates', 'canned_responses', 'email_signatures',
      'ai_disclaimers', 'notification_channels', 'notification_routing'
    ])
  LOOP
    -- scope 컬럼
    EXECUTE format(
      'ALTER TABLE %I ADD COLUMN IF NOT EXISTS scope VARCHAR(10) NOT NULL DEFAULT ''shared''',
      tbl
    );
    -- owner_id 컬럼
    EXECUTE format(
      'ALTER TABLE %I ADD COLUMN IF NOT EXISTS owner_id INTEGER REFERENCES agents(id) ON DELETE CASCADE',
      tbl
    );
    -- 제약조건
    EXECUTE format(
      'ALTER TABLE %I DROP CONSTRAINT IF EXISTS chk_%s_scope_owner', tbl, tbl
    );
    EXECUTE format(
      'ALTER TABLE %I ADD CONSTRAINT chk_%s_scope_owner CHECK (
        (scope = ''shared'' AND owner_id IS NULL) OR
        (scope = ''personal'' AND owner_id IS NOT NULL)
      )', tbl, tbl
    );
    -- 인덱스
    EXECUTE format(
      'CREATE INDEX IF NOT EXISTS idx_%s_scope ON %I(scope)', tbl, tbl
    );
    EXECUTE format(
      'CREATE INDEX IF NOT EXISTS idx_%s_owner ON %I(owner_id) WHERE owner_id IS NOT NULL', tbl, tbl
    );
  END LOOP;
END $$;

-- 2. 개인 메일함 설정 테이블
CREATE TABLE IF NOT EXISTS personal_mailbox_settings (
  id SERIAL PRIMARY KEY,
  agent_id INTEGER NOT NULL UNIQUE REFERENCES agents(id) ON DELETE CASCADE,
  signature_id INTEGER REFERENCES email_signatures(id),
  disclaimer_id INTEGER REFERENCES ai_disclaimers(id),
  auto_draft_enabled BOOLEAN DEFAULT true,
  auto_draft_confidence_threshold FLOAT DEFAULT 0.70,
  notification_channel_id INTEGER REFERENCES notification_channels(id),
  auto_save_knowledge BOOLEAN DEFAULT false,
  default_knowledge_repo_id INTEGER REFERENCES knowledge_repositories(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 3. 지식 축적 이력
CREATE TABLE IF NOT EXISTS knowledge_creation_logs (
  id SERIAL PRIMARY KEY,
  email_id INTEGER REFERENCES emails(id),
  email_type VARCHAR(10) NOT NULL CHECK (email_type IN ('personal', 'shared')),
  knowledge_doc_id INTEGER REFERENCES knowledge_documents(id),
  conversation_summary TEXT,
  created_by INTEGER REFERENCES agents(id),
  scope VARCHAR(10) NOT NULL DEFAULT 'personal',
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 4. 파일 구조 요약

### 신규 파일 (공용 인프라)

```
app/
  models/
    mixins.py            ← ScopeMixin (1.1)
  services/
    scope_service.py     ← ScopeService (1.2)
  api/
    crud_helpers.py      ← list/create/update/delete_with_scope (1.3)

admin/src/
  components/
    scope-tabs.tsx       ← ScopeTabs (2.1)
    role-guard.tsx       ← RoleGuard (2.3)
  hooks/
    use-entity-permissions.ts  ← useEntityPermissions (2.2)
    use-scoped-query.ts        ← useScopedQuery (2.4)
    use-scoped-mutation.ts     ← useScopedCRUD (2.5)

scripts/
  migration_shared.sql   ← 공용 마이그레이션 (3)
```

### 수정 파일

```
app/services/auth.py     ← get_current_user_optional 추가, require_role 수정 (1.4, 1.5)
app/models/base.py       ← 9개 모델에 ScopeMixin 상속 추가 (1.1)
app/api/admin.py         ← 기존 CRUD를 crud_helpers 호출로 교체 (1.3)
admin/src/components/app-sidebar.tsx ← roles 필터 추가 (2.6)
```

---

## 5. 구현 순서

```
Step 1: migration_shared.sql 실행 (DB 준비)
Step 2: mixins.py + base.py 수정 (모델)
Step 3: scope_service.py + crud_helpers.py (백엔드 서비스)
Step 4: auth.py 수정 (인증)
Step 5: admin.py CRUD 엔드포인트 교체 (API)
Step 6: 프론트엔드 공용 컴포넌트 (scope-tabs, role-guard, hooks)
Step 7: app-sidebar.tsx 수정 (메뉴 필터링)
```

**이 7단계를 먼저 완료하면, Phase 1~5는 이 인프라를 import하여 사용만 하면 됨.**
