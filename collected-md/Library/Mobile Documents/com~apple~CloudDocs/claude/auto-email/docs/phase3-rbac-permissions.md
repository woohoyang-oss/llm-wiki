# Phase 3: Role-Based Access Control (RBAC) — 상세 개발 명세서 v2

> 목표: admin/agent/viewer 역할별 API 접근 제어 + 에이전트 전용 대시보드
> 갱신: 2026-03-13 — 코드 재점검 반영

---

## 0. 사용자 워크플로우 (Before → After)

### Before (현재)
```
1. admin 역할로 로그인 → 모든 메뉴, 모든 데이터 접근 가능
2. agent 역할로 로그인 → 역시 모든 메뉴, 모든 데이터 접근 가능 (문제!)
3. viewer 역할로 로그인 → 역시 모든 것 가능, 규칙 삭제도 가능 (위험!)
4. JWT에 role 필드 포함되지만 아무도 검증하지 않음
```

### After (구현 후)
```
1. admin 로그인 → 전체 메뉴 표시, 모든 설정 CRUD 가능
2. agent 로그인 →
   - 메뉴: 대시보드, 내 메일함, 승인 대기, 지식문서, 매크로, 템플릿
   - 메일: 본인 담당만 조회, 처리, 회신 가능
   - 설정: 룰, 메일박스, 엔진 등 변경 불가
3. viewer 로그인 →
   - 메뉴: 대시보드, 통계 (읽기 전용)
   - 메일: 목록 열람만 가능, 수정/발송 불가
```

---

## 1. 현재 코드 정밀 분석 — 핵심 발견

### 1.1 require_role()이 이미 존재하지만 사용되지 않음!

```python
# app/services/auth.py:159-167 — 이미 구현됨!
def require_role(*roles: str):
    """특정 role 필요 (admin, agent, viewer)"""
    async def dependency(user: Agent = Depends(get_current_user)) -> Agent:
        if user.role not in roles:
            raise HTTPException(403, f"권한이 부족합니다. 필요: {', '.join(roles)}")
        return user
    return Depends(dependency)
```

→ **Phase 3의 핵심은 새로 만드는 게 아니라 이것을 전체 엔드포인트에 적용하는 것**

### 1.2 verify_admin()의 문제

```python
# app/api/admin.py:50-69
async def verify_admin(request, x_admin_key, db):
    # JWT 인증
    auth_header = request.headers.get("Authorization", "")
    if auth_header.startswith("Bearer "):
        token = auth_header[7:]
        payload = verify_jwt(token)  # 검증만 하고 role 확인 안함
        return  # ← 여기서 그냥 통과! role 무관

    # admin_key 폴백
    if x_admin_key == settings.admin_secret_key:
        return

    raise HTTPException(401)
```

**문제**: `verify_jwt(token)` 후 `payload["role"]`을 전혀 확인하지 않음. JWT만 유효하면 무조건 통과.

### 1.3 프론트엔드 — useAuth() 존재하지만 미사용

```typescript
// admin/src/hooks/use-auth.ts:19-56
export function useAuth() {
    // ... parseToken으로 role 추출 ...
    return {
        token, user, isLoggedIn,
        isAdmin: user?.role === "admin",  // ← 이미 있음!
        login, logout
    };
}
```

→ `isAdmin` 프로퍼티가 있지만 어떤 페이지에서도 사용하지 않음

### 1.4 사이드바 — 역할 필터링 없음

```typescript
// admin/src/components/app-sidebar.tsx:54-105
const navGroups = [
    // ... 20+ 메뉴 항목 — role 속성 없음
];
```

---

## 2. 구현 태스크 목록

### Task 3-1: auth.py 보강 + 데이터 스코프 (백엔드 핵심)

**파일**: `app/services/auth.py` — 기존 코드 확장

**기존 require_role() 활용** (line 159-167), 추가 유틸리티:

```python
# ── 기존 (유지) ──
def require_role(*roles: str):
    async def dependency(user: Agent = Depends(get_current_user)) -> Agent:
        if user.role not in roles:
            raise HTTPException(403, f"권한이 부족합니다. 필요: {', '.join(roles)}")
        return user
    return Depends(dependency)

# ── 신규 추가 ──

def require_admin():
    """admin 전용 (편의 함수)"""
    return require_role("admin")

def require_agent_or_above():
    """admin + agent"""
    return require_role("admin", "agent")

def require_any_authenticated():
    """모든 인증 사용자 (admin + agent + viewer)"""
    return require_role("admin", "agent", "viewer")


async def get_current_user_optional(request: Request, db: AsyncSession = Depends(get_db)) -> Agent | None:
    """
    현재 사용자 반환
    - JWT: Agent 객체
    - X-Admin-Key: None (슈퍼어드민)
    - 인증 없음: HTTPException(401)
    """
    auth_header = request.headers.get("Authorization", "")
    if auth_header.startswith("Bearer "):
        token = auth_header[7:]
        payload = verify_jwt(token)
        agent = await db.get(Agent, int(payload["sub"]))
        if agent:
            return agent

    x_admin_key = request.headers.get("X-Admin-Key")
    if x_admin_key == settings.admin_secret_key:
        return None  # 슈퍼어드민

    raise HTTPException(401, "인증 필요")


async def require_email_access(
    email_id: int,
    current_user: Agent | None = Depends(get_current_user_optional),
    db: AsyncSession = Depends(get_db),
) -> Email:
    """
    이메일 접근 제어
    - 슈퍼어드민(None) / admin: 모든 이메일
    - agent: 본인 assignee_id만
    - viewer: 읽기 접근 (write 여부는 호출 측에서 판단)
    """
    email = await db.get(Email, email_id)
    if not email:
        raise HTTPException(404, "이메일 없음")

    # 슈퍼어드민 또는 admin
    if current_user is None or current_user.role == "admin":
        return email

    # agent — 본인 담당만
    if current_user.role == "agent" and email.assignee_id != current_user.id:
        raise HTTPException(403, "본인 담당 이메일만 접근 가능합니다")

    return email
```

### Task 3-2: Admin API 전체 엔드포인트에 require_role 적용 (백엔드 핵심)

**파일**: `app/api/admin.py` — verify_admin을 require_role로 교체

**변경 방식**: 라우터 레벨 dependencies에서 개별 엔드포인트 dependencies로 변경

```python
# 현재 (라우터 전체에 verify_admin 적용):
router = APIRouter(prefix="/admin", dependencies=[Depends(verify_admin)])

# 수정: 라우터 레벨 dependencies 제거, 개별 적용
router = APIRouter(prefix="/admin")
```

**적용 매트릭스** (23개 그룹, 65+ 엔드포인트):

```python
# ═══════════════ 모든 인증 사용자 (V) ═══════════════
from app.services.auth import require_any_authenticated, require_admin, require_agent_or_above

@router.get("/dashboard", dependencies=[require_any_authenticated()])
@router.get("/dashboard/volume", dependencies=[require_any_authenticated()])
@router.get("/dashboard/statistics", dependencies=[require_any_authenticated()])
@router.get("/recent-activities", dependencies=[require_any_authenticated()])
@router.get("/emails/tags", dependencies=[require_any_authenticated()])
@router.get("/emails/brands", dependencies=[require_any_authenticated()])

# ═══════════════ admin + agent (G) ═══════════════
@router.get("/macros", dependencies=[require_agent_or_above()])
@router.post("/macros", dependencies=[require_agent_or_above()])
@router.put("/macros/{id}", dependencies=[require_agent_or_above()])
@router.get("/email-templates", dependencies=[require_agent_or_above()])
@router.post("/email-templates", dependencies=[require_agent_or_above()])

# ═══════════════ admin 전용 (A) ═══════════════
@router.get("/system-overview", dependencies=[require_admin()])
@router.get("/emails/export", dependencies=[require_admin()])
@router.post("/emails/batch-process", dependencies=[require_admin()])
@router.post("/emails/bulk", dependencies=[require_admin()])
@router.patch("/emails/{id}/assign", dependencies=[require_admin()])
@router.patch("/emails/{id}/priority", dependencies=[require_admin()])

# CRUD: rules, mailboxes, agents, agent-groups, engines, disclaimers,
#        signatures, notification-channels, notification-routings,
#        assignment-rules, schedules, repositories, sla-policies,
#        business-hours, automations
# → 모두 dependencies=[require_admin()]

# ═══════════════ 특수 로직 (데이터 스코프) ═══════════════

@router.get("/emails")
async def list_emails(
    current_user: Agent | None = Depends(get_current_user_optional),
    db = Depends(get_db),
    ...
):
    """이메일 목록 — 역할별 데이터 범위 제한"""
    query = select(Email)

    # agent/viewer → 본인 담당만
    if current_user and current_user.role in ("agent", "viewer"):
        query = query.where(Email.assignee_id == current_user.id)

    # ... 기존 필터 로직 ...


@router.get("/emails/{email_id}")
async def get_email(
    email: Email = Depends(require_email_access),  # 접근 제어 자동
    ...
):
    ...


@router.patch("/emails/{email_id}/status")
async def update_status(
    email: Email = Depends(require_email_access),
    current_user: Agent | None = Depends(get_current_user_optional),
    ...
):
    if current_user and current_user.role == "viewer":
        raise HTTPException(403, "읽기 전용 계정")
    ...


@router.post("/emails/{email_id}/send-reply")
async def send_reply(
    email: Email = Depends(require_email_access),
    current_user: Agent | None = Depends(get_current_user_optional),
    ...
):
    if current_user and current_user.role == "viewer":
        raise HTTPException(403, "읽기 전용 계정")
    ...


@router.get("/approvals")
async def list_approvals(
    current_user: Agent | None = Depends(get_current_user_optional),
    db = Depends(get_db),
    ...
):
    query = select(ApprovalQueue).where(ApprovalQueue.status == "pending")
    # agent → 본인 담당만
    if current_user and current_user.role == "agent":
        query = query.join(Email).where(Email.assignee_id == current_user.id)
    ...


@router.post("/approvals/{id}/action")
async def approval_action(
    current_user: Agent | None = Depends(get_current_user_optional),
    ...
):
    if current_user and current_user.role == "viewer":
        raise HTTPException(403, "읽기 전용 계정")
    # agent → 본인 담당 승인만
    if current_user and current_user.role == "agent":
        email = await db.get(Email, approval.email_id)
        if email.assignee_id != current_user.id:
            raise HTTPException(403, "본인 담당만 처리 가능")
    ...
```

**GET /admin/me 엔드포인트 추가**:

```python
@router.get("/me")
async def get_me(current_user: Agent | None = Depends(get_current_user_optional)):
    if current_user is None:
        return {"role": "admin", "name": "System Admin", "is_super": True}
    return {
        "id": current_user.id,
        "email": current_user.email,
        "name": current_user.name,
        "role": current_user.role,
        "avatar_url": current_user.avatar_url,
        "group_id": current_user.group_id,
        "is_super": False,
    }
```

---

### Task 3-3: 프론트엔드 권한 체크 + 사이드바 필터링 (병렬 가능: Frontend)

**3-3a: 사이드바 역할 필터링** — `app-sidebar.tsx` (line 54-105)

```typescript
// NavItem 타입에 roles 추가
interface NavItem {
  label: string;
  href: string;
  icon: LucideIcon;
  badgeKey?: string;
  roles?: ("admin" | "agent" | "viewer")[];  // 추가
}

const navGroups: { label: string; items: NavItem[] }[] = [
  {
    label: "개요",
    items: [
      { label: "대시보드", href: "/dashboard", icon: LayoutDashboard },
      { label: "통계", href: "/statistics", icon: BarChart3 },
    ],
  },
  {
    label: "이메일 처리",
    items: [
      { label: "내 메일함", href: "/my-inbox", icon: Inbox, roles: ["admin", "agent"] },
      { label: "승인 대기", href: "/approvals", icon: ClipboardCheck, badgeKey: "pending_approvals" },
      { label: "공용 메일함", href: "/emails", icon: Mail, badgeKey: "unreplied_count" },
    ],
  },
  {
    label: "자동화 설정",
    items: [
      { label: "룰 관리", href: "/rules", icon: Settings2, roles: ["admin"] },
      { label: "스케줄 관리", href: "/schedules", icon: Timer, roles: ["admin"] },
      { label: "배정 규칙", href: "/assignment-rules", icon: UserCheck, roles: ["admin"] },
      { label: "자동화", href: "/automations", icon: Zap, roles: ["admin"] },
      { label: "SLA 정책", href: "/sla-policies", icon: Shield, roles: ["admin"] },
      { label: "영업시간", href: "/business-hours", icon: Clock, roles: ["admin"] },
    ],
  },
  // ... 지식 베이스, 채널 & 설정, 시스템 — 각 항목에 roles 지정
];

// 렌더링 시 필터
const { user } = useAuth();
const filteredGroups = navGroups.map(group => ({
  ...group,
  items: group.items.filter(item =>
    !item.roles || item.roles.includes(user?.role || "viewer")
  ),
})).filter(group => group.items.length > 0);
```

**3-3b: 대시보드 역할별 분기** — `dashboard/page.tsx`

```typescript
const { user } = useAuth();

// admin → 기존 전체 대시보드
// agent → 본인 담당 메일 + 승인 대기 위젯
// viewer → 통계 요약만

if (user?.role === "agent") {
  return (
    <div>
      <h1>안녕하세요, {user.name}님</h1>
      <div className="grid grid-cols-4 gap-4">
        <StatCard title="내 담당" value={myEmailCount} />
        <StatCard title="승인 대기" value={myApprovalCount} />
        <StatCard title="오늘 처리" value={todayProcessed} />
        <StatCard title="SLA 위반" value={slaBreached} />
      </div>
      <MyEmailList />
      <MyApprovalList />
    </div>
  );
}
```

**3-3c: 페이지별 가드** — 권한 없는 페이지 접근 시 리다이렉트

```typescript
// admin/src/components/role-guard.tsx (신규)
export function RoleGuard({
  allowed,
  children,
}: {
  allowed: ("admin" | "agent" | "viewer")[];
  children: React.ReactNode;
}) {
  const { user } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (user && !allowed.includes(user.role)) {
      router.push("/dashboard");
    }
  }, [user, allowed, router]);

  if (!user || !allowed.includes(user.role)) {
    return <div className="p-8 text-muted-foreground">접근 권한이 없습니다.</div>;
  }

  return <>{children}</>;
}

// 사용:
// admin/src/app/rules/page.tsx
export default function RulesPage() {
  return (
    <RoleGuard allowed={["admin"]}>
      <RulesContent />
    </RoleGuard>
  );
}
```

---

## 3. 의존성 그래프 + 병렬 개발 구조

```
Task 3-1 (auth.py 보강) ──→ Task 3-2 (admin.py 전체 적용)

Task 3-3a (사이드바)  ┐
Task 3-3b (대시보드)  ├── 병렬 가능 (Backend 독립)
Task 3-3c (RoleGuard) ┘

병렬 가능 그룹:
  A: Task 3-1 (Backend)
  B: Task 3-3a, 3-3b, 3-3c (Frontend, A와 병렬 — API 응답에 role 이미 포함)
  C: Task 3-2 (A 완료 후 — admin.py 대규모 수정)
  D: 통합 테스트 (B, C 완료 후)
```

---

## 4. Alembic 마이그레이션

이 Phase에서 **DB 스키마 변경 없음**. 기존 `agents.role` 필드 활용.
(Phase 2의 notification_channel_id, notification_events는 Phase 2 마이그레이션에 포함)

---

## 5. 보안 고려사항

### 5.1 X-Admin-Key 취급
- X-Admin-Key는 서버 간 통신용 (Celery 태스크, 내부 API)
- `get_current_user_optional()`에서 X-Admin-Key → None(슈퍼어드민) 반환
- require_role()에서 `user.role` 체크 → user가 None이면 통과 (슈퍼어드민)
- **주의**: require_role()이 Agent 의존성이므로 X-Admin-Key 경우 별도 처리 필요

### 5.2 기존 require_role() 수정 필요

```python
# 현재 (auth.py:159-167):
def require_role(*roles: str):
    async def dependency(user: Agent = Depends(get_current_user)) -> Agent:
        if user.role not in roles:
            raise HTTPException(403)
        return user
    return Depends(dependency)

# 문제: get_current_user는 X-Admin-Key를 처리하지 않음
# 수정: get_current_user_optional 사용
def require_role(*roles: str):
    async def dependency(
        current_user: Agent | None = Depends(get_current_user_optional)
    ) -> Agent | None:
        # 슈퍼어드민 → 항상 통과
        if current_user is None:
            return None
        if current_user.role not in roles:
            raise HTTPException(403, f"권한 부족. 필요: {', '.join(roles)}")
        return current_user
    return Depends(dependency)
```

### 5.3 프론트엔드는 UX용
메뉴 숨기기는 사용성 개선이지 보안이 아님. **모든 보안은 API 레벨에서 처리**.

---

## 6. 테스트 체크리스트

### 인증/권한
- [ ] JWT 인증 + role=admin → 모든 엔드포인트 접근
- [ ] JWT 인증 + role=agent → 설정 CRUD 403 거부
- [ ] JWT 인증 + role=viewer → 수정 API 403 거부
- [ ] X-Admin-Key → 모든 엔드포인트 접근 (슈퍼어드민)
- [ ] 인증 없음 → 401

### 데이터 스코프
- [ ] agent GET /emails → assignee_id=본인 메일만
- [ ] agent GET /emails/{id} → 타인 담당 메일 403
- [ ] agent GET /approvals → 본인 담당 승인건만
- [ ] agent POST /approvals/{id}/action → 타인 승인건 403
- [ ] admin GET /emails → 전체 메일
- [ ] viewer GET /emails → assignee_id=본인 메일 (읽기 전용)
- [ ] viewer PATCH /emails/{id}/status → 403

### UI
- [ ] admin 로그인 → 전체 메뉴 (20+ 항목)
- [ ] agent 로그인 → 제한 메뉴 (대시보드, 메일, 승인, 지식문서, 매크로, 템플릿)
- [ ] viewer 로그인 → 최소 메뉴 (대시보드, 통계)
- [ ] agent 대시보드 → "내 담당 메일" + "내 승인 대기" 위젯
- [ ] 권한 없는 URL 직접 접근 → 대시보드로 리다이렉트
- [ ] RoleGuard → "접근 권한이 없습니다" 표시

### 기존 기능 비파괴 확인
- [ ] verify_admin → require_role 교체 후 기존 API 정상 동작
- [ ] X-Admin-Key 인증 → Celery 태스크 정상 동작
- [ ] OAuth 로그인 플로우 정상
