# Phase 2: 알림 라우팅 + 자동배정 — 상세 개발 명세서 v2

> 목표: 담당 에이전트에게만 알림 전송, 브랜드×카테고리 기반 자동 배정
> 갱신: 2026-03-13 — 코드 재점검 반영

---

## 0. 사용자 워크플로우 (Before → After)

### Before (현재)
```
1. 새 메일 수신 → AI 처리 → 승인 대기
2. Telegram 알림 "승인 필요" → 모든 관리자에게 동일하게 전송
3. 누가 처리할지 모름 → Admin이 수동으로 에이전트 배정
4. 에이전트는 자기 담당인지 확인하려면 Admin UI 직접 열어야 함
```

### After (구현 후)
```
1. 새 메일 수신 (support@keychron.kr)
2. 자동배정 엔진: keychron × cs → CS팀 라운드로빈 → 김상담원 배정
3. AI 처리 → 승인 대기
4. 알림 라우팅:
   - 김상담원 개인 Telegram: "키크론 CS 승인 필요: 배송 문의"
   - 전체 Slack 채널: "[승인대기] support@keychron.kr → 김상담원"
5. 김상담원이 Telegram 알림 확인 → Admin UI에서 승인
```

### 자동배정 시나리오

```
시나리오 1: 키크론 CS 문의
  메일: support@keychron.kr → brand=keychron, category=cs
  배정규칙 매칭: "assign-keychron-cs" (priority=10, strategy=round_robin, group=CS팀)
  CS팀 에이전트: [김상담원, 이상담원, 박상담원]
  마지막 배정: 김상담원 → 이번: 이상담원

시나리오 2: B2B 문의
  메일: b2b@keychron.kr → brand=keychron, category=b2b
  배정규칙 매칭: "assign-b2b" (priority=30, strategy=fixed, agent=최영업)
  항상 최영업에게 배정

시나리오 3: 매칭 안되는 메일
  배정규칙 매칭: "assign-default" (priority=999, strategy=least_load)
  현재 미처리 메일 수: 김=5, 이=3, 박=7 → 이상담원에 배정
```

---

## 1. 현재 코드 정밀 분석

### 1.1 알림 시스템 현재 코드

**notification.py 핵심 구조**:

```python
# line 91-95 — 어댑터 매핑 (lambda 팩토리)
ADAPTER_MAP = {
    "slack": lambda cfg: SlackWebhookAdapter(cfg["webhook_url"]),
    "telegram": lambda cfg: TelegramAdapter(cfg["bot_token"], cfg["chat_id"]),
    "webhook": lambda cfg: WebhookAdapter(cfg["url"], cfg.get("headers")),
}

# line 101-107 — notify() 현재 시그니처
async def notify(
    self,
    event_type: str,
    message: str,
    db: AsyncSession,
    metadata: dict | None = None,
) -> list[bool]:
```

**주의: 테이블명이 `notification_routing` (단수)**:
```python
# base.py:432 — 실제 테이블명
class NotificationRouting(Base):
    __tablename__ = "notification_routing"  # ← 's' 없음!
```

**현재 notify() 라우팅 로직** (line 119-125):
```python
# event_type + is_active 매칭만 수행
result = await db.execute(
    select(NotificationRouting).where(
        NotificationRouting.event_type == event_type,
        NotificationRouting.is_active == True,
    )
)
```
→ brand, category, agent_id 필터 없음

### 1.2 파이프라인에서 알림 호출

```python
# pipeline.py:329-338 — _auto_send() 내
await self.notification.notify(
    event_type="auto_sent",
    message=f"[자동발송] {email.from_address} → {subject[:50]}",
    db=db,
    metadata={
        "email_id": email.id,
        "from": email.from_address,
        "subject": email.subject,
        "confidence": ai_response.confidence,
    },
)
```
→ brand, category, assignee_id 전달하지 않음

### 1.3 현재 Admin API 알림 엔드포인트

```python
# admin.py — 채널 CRUD만 존재
GET/POST/PUT/DELETE /admin/notification-channels  # line 1783-1829
# 라우팅 CRUD 엔드포인트: 없음!
```

---

## 2. 구현 태스크 목록

### Task 2-1: DB 스키마 + 모델 변경 (병렬 가능: Backend)

**새 테이블: `assignment_rules`**

```sql
CREATE TABLE assignment_rules (
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
```

**기존 테이블 변경**:

```sql
-- notification_routing (단수 — 기존 테이블명 유지!)
ALTER TABLE notification_routing ADD COLUMN agent_id INTEGER REFERENCES agents(id);
ALTER TABLE notification_routing ADD COLUMN brand VARCHAR;
ALTER TABLE notification_routing ADD COLUMN category VARCHAR;
ALTER TABLE notification_routing ADD COLUMN priority_filter VARCHAR;

-- agents
ALTER TABLE agents ADD COLUMN notification_channel_id VARCHAR
    REFERENCES notification_channels(channel_id);
ALTER TABLE agents ADD COLUMN notification_events JSONB
    DEFAULT '["approval_needed","escalation"]'::jsonb;
```

**모델 추가/수정** — `app/models/base.py`:

```python
# ── 새 모델 ──
class AssignmentRule(Base):
    __tablename__ = "assignment_rules"
    id: Mapped[int] = mapped_column(primary_key=True)
    rule_id: Mapped[str] = mapped_column(String, unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String, nullable=False)
    description: Mapped[str | None] = mapped_column(String)
    brand: Mapped[str | None] = mapped_column(String)
    category: Mapped[str | None] = mapped_column(String)
    priority_match: Mapped[str | None] = mapped_column(String)
    agent_id: Mapped[int | None] = mapped_column(Integer, ForeignKey("agents.id"))
    group_id: Mapped[int | None] = mapped_column(Integer, ForeignKey("agent_groups.id"))
    strategy: Mapped[str] = mapped_column(String, default="round_robin")
    priority: Mapped[int] = mapped_column(Integer, default=100)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    last_assigned_agent_id: Mapped[int | None] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    agent: Mapped["Agent | None"] = relationship("Agent", foreign_keys=[agent_id])
    group: Mapped["AgentGroup | None"] = relationship("AgentGroup")

# ── NotificationRouting 확장 (line 432-440에 필드 추가) ──
# 기존 필드 뒤에:
agent_id: Mapped[int | None] = mapped_column(Integer, ForeignKey("agents.id"))
brand: Mapped[str | None] = mapped_column(String)
category: Mapped[str | None] = mapped_column(String)
priority_filter: Mapped[str | None] = mapped_column(String)

# ── Agent 확장 (line 34-54에 필드 추가) ──
# google_id 앞에:
notification_channel_id: Mapped[str | None] = mapped_column(
    String, ForeignKey("notification_channels.channel_id")
)
notification_events: Mapped[list] = mapped_column(
    JSONB, default=lambda: ["approval_needed", "escalation"]
)
```

**seed_data.py SEED_MAP 추가**:

```python
("assignment_rules.json", AssignmentRule, "rule_id"),
```

---

### Task 2-2: 자동배정 서비스 (Task 2-1 의존)

**파일**: `app/services/assignment_service.py` (신규)

**핵심 로직**:

```python
class AssignmentService:
    @staticmethod
    async def auto_assign(db: AsyncSession, email: Email) -> int | None:
        """
        이메일에 에이전트 자동 배정

        매칭 우선순위 (priority ASC):
        1. brand=keychron, category=cs (priority=10) ← 가장 구체적
        2. brand=keychron, category=NULL (priority=20) ← 브랜드만
        3. brand=NULL, category=NULL (priority=999) ← 기본 규칙

        Returns: agent_id or None
        """
        # 활성 규칙을 priority 순으로 조회
        rules = await db.execute(
            select(AssignmentRule)
            .where(AssignmentRule.is_active == True)
            .order_by(AssignmentRule.priority.asc())
        )

        for rule in rules.scalars():
            if not _matches(rule, email):
                continue

            agent_id = await _resolve_agent(db, rule)
            if agent_id:
                return agent_id

        return None
```

**배정 전략 상세**:

| 전략 | 동작 | 사용 케이스 |
|------|------|-----------|
| `fixed` | 항상 동일 에이전트 | B2B 전담, VIP 고객 전담 |
| `round_robin` | 그룹 내 순환 배정 | 일반 CS (균등 분배) |
| `least_load` | 미처리 메일 최소 에이전트 | 피크 시간대 (부하 분산) |

**round_robin 구현 핵심**:

```python
# 그룹의 활성 에이전트 목록 (id 기준 정렬)
agent_ids = [1, 3, 5]

# last_assigned_agent_id = 3 이면
idx = agent_ids.index(3)  # → 1
next_idx = (1 + 1) % 3    # → 2
next_agent_id = agent_ids[2]  # → 5

# rule.last_assigned_agent_id = 5 로 갱신
```

**least_load 구현 핵심**:

```python
# 각 에이전트의 활성 메일 수 조회
SELECT agent_id, COUNT(*) as load
FROM emails
WHERE assignee_id IN (1, 3, 5)
  AND status IN ('new', 'open', 'pending', 'processing')
GROUP BY agent_id

# load가 가장 적은 에이전트 선택
# 동률이면 agent_id가 작은 쪽 (안정적 정렬)
```

---

### Task 2-3: 알림 서비스 확장 (Task 2-1 의존)

**파일**: `app/services/notification.py` 수정

**변경 1: EmailAdapter 추가**

```python
class EmailAdapter(BaseChannelAdapter):
    """내부 알림 메일 발송"""
    def __init__(self, to: str, from_addr: str = "ai@tbe.kr"):
        self.to = to
        self.from_addr = from_addr

    async def send(self, message: str, metadata: dict | None = None) -> bool:
        from app.services.gmail_service import GmailService
        gmail = GmailService()
        subject = f"[알림] {metadata.get('subject', '새 알림')}" if metadata else "[알림]"
        gmail.send_email(to=self.to, subject=subject, body_html=f"<p>{message}</p>")
        return True
```

**변경 2: ADAPTER_MAP 수정** (line 91-95):

```python
# 현재: lambda 팩토리
ADAPTER_MAP = {
    "slack": lambda cfg: SlackWebhookAdapter(cfg["webhook_url"]),
    "telegram": lambda cfg: TelegramAdapter(cfg["bot_token"], cfg["chat_id"]),
    "webhook": lambda cfg: WebhookAdapter(cfg["url"], cfg.get("headers")),
}

# 수정: email 추가
ADAPTER_MAP = {
    "slack": lambda cfg: SlackWebhookAdapter(cfg["webhook_url"]),
    "telegram": lambda cfg: TelegramAdapter(cfg["bot_token"], cfg["chat_id"]),
    "webhook": lambda cfg: WebhookAdapter(cfg["url"], cfg.get("headers")),
    "email": lambda cfg: EmailAdapter(cfg["to"], cfg.get("from", "ai@tbe.kr")),
}
```

**변경 3: notify() 시그니처 확장** (line 101-107):

```python
# 현재:
async def notify(self, event_type, message, db, metadata=None)

# 수정 (기존 호출 깨지지 않도록 kwargs):
async def notify(
    self,
    event_type: str,
    message: str,
    db: AsyncSession,
    metadata: dict | None = None,
    *,
    brand: str | None = None,
    category: str | None = None,
    assignee_id: int | None = None,
    priority: str | None = None,
) -> list[bool]:
```

**변경 4: 라우팅 로직 수정** (line 119-125):

```python
# 현재: event_type만 매칭
# 수정: 컨텍스트 기반 매칭

routings = (await db.execute(
    select(NotificationRouting).where(
        NotificationRouting.event_type == event_type,
        NotificationRouting.is_active == True,
    )
)).scalars().all()

matched = []
for r in routings:
    if r.brand and r.brand != brand:
        continue
    if r.category and r.category != category:
        continue
    if r.agent_id and r.agent_id != assignee_id:
        continue
    if r.priority_filter and priority:
        allowed = [p.strip() for p in r.priority_filter.split(",")]
        if priority not in allowed:
            continue
    matched.append(r)
```

**변경 5: 에이전트 개인 채널 알림 추가** (notify() 끝에):

```python
# 배정된 에이전트의 개인 채널로도 알림
if assignee_id:
    agent = await db.get(Agent, assignee_id)
    if agent and agent.notification_channel_id:
        events = agent.notification_events or []
        if event_type in events:
            ch_result = await db.execute(
                select(NotificationChannel).where(
                    NotificationChannel.channel_id == agent.notification_channel_id
                )
            )
            ch = ch_result.scalar_one_or_none()
            if ch:
                adapter = ADAPTER_MAP.get(ch.type)
                if adapter:
                    success = await adapter(ch.config).send(message, metadata)
                    results.append(success)
```

---

### Task 2-4: 파이프라인 연동 (Task 2-2, 2-3 의존)

**파일**: `app/services/pipeline.py` 수정

**변경 1**: AI 처리 후 자동배정 삽입 (line 186-212 이후):

```python
# AI 처리 완료 후, confidence 분기 전에:
if not email.assignee_id:
    from app.services.assignment_service import AssignmentService
    agent_id = await AssignmentService.auto_assign(db, email)
    if agent_id:
        email.assignee_id = agent_id
        await db.flush()
```

**변경 2**: 알림 호출에 컨텍스트 추가 (3곳):

```python
# _auto_send() — line 329
await self.notification.notify(
    "auto_sent", message, db, metadata,
    brand=email.mailbox_brand,
    category=email.mailbox_type,
    assignee_id=email.assignee_id,
    priority=email.priority,
)

# _queue_for_approval() — line 356
await self.notification.notify(
    "approval_needed", message, db, metadata,
    brand=email.mailbox_brand,
    category=email.mailbox_type,
    assignee_id=email.assignee_id,
    priority=email.priority,
)

# _escalate() — line 389
await self.notification.notify(
    "escalation", message, db, metadata,
    brand=email.mailbox_brand,
    category=email.mailbox_type,
    assignee_id=email.assignee_id,
    priority=email.priority,
)
```

---

### Task 2-5: Admin API 엔드포인트 (Task 2-1 의존)

**배정 규칙 CRUD** (4개):

| Method | Path | 기능 |
|--------|------|------|
| GET | `/admin/assignment-rules` | 배정 규칙 목록 (agent_name, group_name 포함) |
| POST | `/admin/assignment-rules` | 배정 규칙 생성 |
| PUT | `/admin/assignment-rules/{rule_id}` | 배정 규칙 수정 |
| DELETE | `/admin/assignment-rules/{rule_id}` | 배정 규칙 삭제 |

**알림 라우팅 CRUD** (4개 — 현재 0개):

| Method | Path | 기능 |
|--------|------|------|
| GET | `/admin/notification-routings` | 라우팅 목록 (확장 필드 포함) |
| POST | `/admin/notification-routings` | 라우팅 생성 |
| PUT | `/admin/notification-routings/{id}` | 라우팅 수정 |
| DELETE | `/admin/notification-routings/{id}` | 라우팅 삭제 |

**에이전트 알림 설정** — 기존 PUT /admin/agents/{id} (line 1423) 확장:

```python
class AgentUpdateRequest(BaseModel):
    # ... 기존 필드 ...
    notification_channel_id: str | None = None
    notification_events: list[str] | None = None
```

**Pydantic 모델**:

```python
class AssignmentRuleCreateRequest(BaseModel):
    rule_id: str
    name: str
    description: str | None = None
    brand: str | None = None
    category: str | None = None
    priority_match: str | None = None
    agent_id: int | None = None
    group_id: int | None = None
    strategy: str = "round_robin"
    priority: int = 100
    is_active: bool = True

    @model_validator(mode="after")
    def validate_target(self):
        if self.strategy == "fixed" and not self.agent_id:
            raise ValueError("fixed 전략은 agent_id 필수")
        if self.strategy in ("round_robin", "least_load") and not self.group_id:
            raise ValueError(f"{self.strategy} 전략은 group_id 필수")
        return self

class NotificationRoutingCreateRequest(BaseModel):
    event_type: str  # 'approval_needed' | 'auto_sent' | 'escalation' | 'error'
    channel_id: str
    template: str | None = None
    is_active: bool = True
    agent_id: int | None = None
    brand: str | None = None
    category: str | None = None
    priority_filter: str | None = None

    @field_validator("event_type")
    @classmethod
    def validate_event(cls, v):
        allowed = {"approval_needed", "auto_sent", "escalation", "error"}
        if v not in allowed:
            raise ValueError(f"event_type은 {allowed} 중 하나여야 합니다")
        return v
```

---

### Task 2-6: Admin UI (병렬 가능: Frontend)

**2-6a: 배정 규칙 페이지** — `admin/src/app/assignment-rules/page.tsx` (신규)

```
기능:
1. 규칙 목록 테이블 (priority 순)
   - 컬럼: 우선순위, 규칙명, 브랜드, 카테고리, 전략, 대상(에이전트/그룹), 상태
   - 드래그로 우선순위 변경 또는 숫자 입력

2. 생성/수정 모달
   - 매칭 조건: 브랜드 드롭다운, 카테고리 드롭다운, 우선순위 필터
   - 배정 전략: fixed/round_robin/least_load 라디오
   - 대상 선택: 에이전트 드롭다운 (fixed) / 그룹 드롭다운 (RR/LL)

3. 미리보기
   - "이 조건에 해당하는 최근 메일 N건" 표시 (선택 사항)
```

**사이드바 추가** — app-sidebar.tsx "자동화 설정" 그룹에:
```typescript
{ label: "배정 규칙", href: "/assignment-rules", icon: UserCheck },
```

**2-6b: 알림 라우팅 탭** — `admin/src/app/notifications/page.tsx` 확장

현재: 채널 관리 1개 탭만 존재
추가: 탭 3개 구조

```
[채널 관리]  [라우팅 규칙]  [에이전트 알림]

라우팅 규칙 탭:
┌──────────────────────────────────────────────────────┐
│ 이벤트          │ 채널       │ 브랜드  │ 에이전트 │ 상태│
├──────────────────────────────────────────────────────┤
│ approval_needed │ tg-admin   │ 전체   │ 전체    │ 🟢 │
│ approval_needed │ tg-kim     │ keychron│ 김상담원│ 🟢 │
│ escalation      │ slack-dev  │ 전체   │ 전체    │ 🟢 │
└──────────────────────────────────────────────────────┘

에이전트 알림 탭:
┌──────────────────────────────────────────────────────┐
│ 에이전트   │ 개인 채널     │ 수신 이벤트                 │
├──────────────────────────────────────────────────────┤
│ 김상담원   │ [tg-kim ▼]   │ ☑ approval ☑ escalation   │
│ 이관리자   │ [tg-admin ▼] │ ☑ all events              │
│ 박담당    │ [(미설정) ▼]  │ -                          │
└──────────────────────────────────────────────────────┘
```

---

## 3. 의존성 그래프 + 병렬 개발 구조

```
Task 2-1 (모델+스키마) ──┬──→ Task 2-2 (AssignmentService)
                         ├──→ Task 2-3 (notification.py 확장)
                         └──→ Task 2-5 (Admin API)

Task 2-2 + 2-3 ──→ Task 2-4 (pipeline.py 연동)

Task 2-5 ──→ Task 2-6 (Admin UI)

병렬 가능 그룹:
  A: Task 2-1 (독립)
  B: Task 2-2, 2-3, 2-5 (A 완료 후, 서로 독립)
  C: Task 2-4 (B 완료 후)
  D: Task 2-6a, 2-6b (B 완료 후, 서로 독립, Mock API로 A와도 병렬 가능)
```

---

## 4. Seed Data

**seed-data/assignment_rules.json**:

```json
[
  {
    "rule_id": "assign-keychron-cs",
    "name": "키크론 CS 자동배정",
    "description": "키크론 CS 문의 → CS팀 라운드로빈",
    "brand": "keychron",
    "category": "cs",
    "strategy": "round_robin",
    "priority": 10,
    "is_active": true
  },
  {
    "rule_id": "assign-gtgear-cs",
    "name": "지티기어 CS 자동배정",
    "brand": "gtgear",
    "category": "cs",
    "strategy": "round_robin",
    "priority": 20,
    "is_active": true
  },
  {
    "rule_id": "assign-b2b",
    "name": "B2B 전담 배정",
    "brand": null,
    "category": "b2b",
    "strategy": "fixed",
    "priority": 30,
    "is_active": true
  },
  {
    "rule_id": "assign-default",
    "name": "기본 배정 (최소부하)",
    "brand": null,
    "category": null,
    "strategy": "least_load",
    "priority": 999,
    "is_active": true
  }
]
```

---

## 5. 테스트 체크리스트

### 자동배정
- [ ] brand+category 모두 매칭 → 해당 규칙 적용
- [ ] brand만 매칭 (category=NULL) → 브랜드 규칙 적용
- [ ] 매칭 없음 → 기본 규칙(priority=999) 적용
- [ ] fixed 전략 → 항상 동일 에이전트
- [ ] round_robin → 순환 배정, last_assigned_agent_id 갱신 확인
- [ ] least_load → 미처리 메일 수 최소 에이전트 선택
- [ ] 비활성 에이전트 → 배정 대상 제외
- [ ] 비활성 규칙 → 매칭 건너뜀
- [ ] agent_id/group_id에 해당하는 에이전트 없음 → None 반환, 다음 규칙으로

### 알림 라우팅
- [ ] event_type만 매칭 (brand/category/agent 모두 NULL) → 전체 발송
- [ ] brand 필터 → 해당 브랜드 메일만
- [ ] agent_id 필터 → 해당 에이전트 담당만
- [ ] priority_filter "high,urgent" → normal 메일에는 발송 안함
- [ ] 에이전트 개인 채널 → notification_channel_id로 직접 발송
- [ ] notification_events 체크 → 선택한 이벤트만
- [ ] EmailAdapter → Gmail API로 알림 메일 발송
- [ ] notify() 기존 호출 (kwargs 없이) → 깨지지 않음

### Admin UI
- [ ] 배정 규칙 CRUD
- [ ] 알림 라우팅 CRUD (확장 필드 포함)
- [ ] 에이전트 알림 설정 (채널 선택, 이벤트 체크박스)
- [ ] 알림 페이지 3개 탭 전환

### E2E
- [ ] 메일 수신 → 자동배정 → AI 처리 → 담당 에이전트에게만 알림
