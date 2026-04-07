# Phase 4: 개인 워크스페이스 + AI 지식 축적 — 상세 개발 계획서 v1

> 작성일: 2026-03-13
> 의존성: Phase 3 (RBAC) 완료 후 착수

---

## 1. 현재 상태 분석

### 1.1 문제: 모든 엔티티가 글로벌

현재 모든 룰, 레포지토리, 지식문서, 템플릿, 매크로, 서명, 고지문, 알림 채널이 **전역(global)** 으로 관리됨.

| 엔티티 | 현재 | 문제점 |
|--------|------|--------|
| Rule | 글로벌 | 내 개인 메일에 적용할 룰을 따로 못 만듦 |
| KnowledgeRepository | 글로벌 | 내 업무 전용 레포지토리 불가 |
| KnowledgeDocument | 글로벌 | 내 개인 지식문서와 공용 지식이 혼재 |
| EmailTemplate | 글로벌 | 내가 자주 쓰는 템플릿 별도 관리 불가 |
| CannedResponse (매크로) | 글로벌 | 개인 매크로 불가 |
| EmailSignature | 글로벌 | 내 서명과 공용 서명 구분 없음 |
| AIDisclaimer | 글로벌 | 개인/공용 구분 없음 |
| NotificationChannel | 글로벌 | 내 알림 설정 불가 |
| NotificationRouting | 글로벌 | 개인 알림 라우팅 불가 |

### 1.2 목표 아키텍처: `scope` + `owner_id` 패턴

```
scope = 'shared' → 공용 (기존과 동일, 권한 필요)
scope = 'personal' → 개인 (owner_id = 로그인 agent ID)
```

모든 엔티티에 `scope`, `owner_id` 컬럼 추가. 기존 데이터는 `scope='shared'`, `owner_id=NULL`.

---

## 2. 사용자 워크플로우

### 2.1 개인 메일함 워크플로우 (Before → After)

**Before (현재):**
1. 내 메일함(`/my-inbox`)에서 개인 메일 확인
2. AI 자동초안 생성 → 수정 → 발송
3. 끝. 지식이 축적되지 않음
4. 같은 유형의 메일이 와도 매번 처음부터 작업

**After (Phase 4):**
1. 내 메일함에서 개인 메일 확인
2. AI 자동초안 생성 (내 개인 지식문서 + 개인 룰 기반)
3. AI와 대화하며 답변 품질 개선 (instruct)
4. 최종 답변 발송
5. **[NEW]** "이 답변을 지식으로 저장" → 개인 지식문서로 자동 생성
6. **[NEW]** 반복 패턴 감지 시 "룰로 만들기" 제안
7. **[NEW]** 다음에 비슷한 메일이 오면 AI가 축적된 지식으로 더 정확한 초안 생성

### 2.2 공용 메일함 권한별 워크플로우

**Admin:**
- 공용 룰 생성/수정/삭제
- 공용 레포지토리 관리
- 공용 지식문서 관리
- 공용 템플릿/매크로 관리
- 공용 서명/고지문 관리
- 공용 알림 채널 관리

**Agent:**
- 공용 엔티티 **조회만** (수정 불가)
- 개인 엔티티 자유롭게 CRUD
- 내 메일함 전용 설정 관리

**Viewer:**
- 공용/개인 모두 **조회만**

### 2.3 지식 축적 사이클

```
[개인 메일 수신]
    ↓
[AI 초안 생성] ← (내 개인 지식 + 공용 지식 모두 참조)
    ↓
[사용자가 AI와 대화하여 답변 개선] (instruct)
    ↓
[최종 답변 발송]
    ↓
[지식 저장 제안] → "이 답변을 지식문서로 저장하시겠습니까?"
    ↓                  ├─ 카테고리 선택 (faq/procedure/policy)
    ↓                  ├─ 제목 자동 생성 (AI)
    ↓                  └─ 공용/개인 선택
    ↓
[지식문서 생성] → KnowledgeDocument + KnowledgeEmbedding
    ↓
[다음 유사 메일] → AI가 이 지식을 참조하여 더 높은 confidence 초안 생성
    ↓
[반복 패턴 감지] → "이 유형의 메일에 대한 룰을 만드시겠습니까?"
    ↓
[자동화 룰 생성] → 향후 자동 처리 가능
```

---

## 3. DB 스키마 변경

### 3.1 공통: scope + owner_id 컬럼 추가

다음 테이블 모두에 동일한 컬럼 추가:

**대상 테이블 (9개):**
- `rules`
- `knowledge_repositories`
- `knowledge_documents`
- `email_templates`
- `canned_responses` (매크로)
- `email_signatures`
- `ai_disclaimers`
- `notification_channels`
- `notification_routing`

**추가 컬럼:**
```sql
ALTER TABLE {table_name}
  ADD COLUMN scope VARCHAR(10) NOT NULL DEFAULT 'shared'
    CHECK (scope IN ('shared', 'personal')),
  ADD COLUMN owner_id INTEGER REFERENCES agents(id) ON DELETE CASCADE;

-- scope='shared'이면 owner_id는 NULL
-- scope='personal'이면 owner_id 필수
ALTER TABLE {table_name}
  ADD CONSTRAINT chk_{table_name}_scope_owner
    CHECK (
      (scope = 'shared' AND owner_id IS NULL) OR
      (scope = 'personal' AND owner_id IS NOT NULL)
    );

-- 개인 엔티티 조회 인덱스
CREATE INDEX idx_{table_name}_owner ON {table_name}(owner_id)
  WHERE owner_id IS NOT NULL;
```

### 3.2 개인 메일함 설정 테이블

```sql
-- 에이전트별 개인 메일함 설정
CREATE TABLE personal_mailbox_settings (
  id SERIAL PRIMARY KEY,
  agent_id INTEGER NOT NULL UNIQUE REFERENCES agents(id) ON DELETE CASCADE,

  -- 내 메일함 서명/고지문
  signature_id INTEGER REFERENCES email_signatures(id),
  disclaimer_id INTEGER REFERENCES ai_disclaimers(id),

  -- AI 자동초안 설정
  auto_draft_enabled BOOLEAN DEFAULT true,
  auto_draft_confidence_threshold FLOAT DEFAULT 0.70,

  -- 기본 알림 설정
  notification_channel_id INTEGER REFERENCES notification_channels(id),

  -- 지식 축적 설정
  auto_save_knowledge BOOLEAN DEFAULT false,  -- 발송 시 자동 지식 저장
  default_knowledge_repo_id INTEGER REFERENCES knowledge_repositories(id),

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### 3.3 지식 축적 이력 테이블

```sql
-- AI 대화 → 지식 변환 이력
CREATE TABLE knowledge_creation_logs (
  id SERIAL PRIMARY KEY,

  -- 원본 메일
  email_id INTEGER REFERENCES emails(id),
  email_type VARCHAR(10) NOT NULL CHECK (email_type IN ('personal', 'shared')),

  -- 생성된 지식
  knowledge_doc_id INTEGER REFERENCES knowledge_documents(id),

  -- AI 대화 내역 (instruct 히스토리)
  conversation_summary TEXT,

  -- 메타데이터
  created_by INTEGER REFERENCES agents(id),
  scope VARCHAR(10) NOT NULL DEFAULT 'personal',
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 4. 태스크 분해

### Task 4-1: DB 마이그레이션 — scope/owner_id 추가

**파일:** `app/models/base.py`

**변경 대상 모델 (9개):**

```python
# 모든 대상 모델에 추가
class ScopeMixin:
    """개인/공용 구분을 위한 Mixin"""
    scope = Column(String(10), nullable=False, default='shared')
    owner_id = Column(Integer, ForeignKey('agents.id', ondelete='CASCADE'), nullable=True)

    # CheckConstraint는 각 모델의 __table_args__에 추가
```

**적용 대상:**
| 모델 | 파일:라인 | 추가 필드 |
|------|-----------|-----------|
| Rule | base.py:148 | scope, owner_id |
| KnowledgeRepository | base.py:384 | scope, owner_id |
| KnowledgeDocument | base.py:401 | scope, owner_id |
| EmailTemplate | base.py:464 | scope, owner_id |
| CannedResponse | base.py:265 | scope, owner_id |
| EmailSignature | base.py:284 | scope, owner_id |
| AIDisclaimer | base.py:340 | scope, owner_id |
| NotificationChannel | base.py:420 | scope, owner_id |
| NotificationRouting | base.py:432 | scope, owner_id |

**신규 모델:**
- `PersonalMailboxSettings` (에이전트별 개인 메일함 설정)
- `KnowledgeCreationLog` (지식 축적 이력)

**기존 데이터 마이그레이션:**
```sql
-- 기존 데이터는 모두 shared로 설정 (DEFAULT 값이므로 자동)
-- 별도 마이그레이션 불필요
```

---

### Task 4-2: Scope 필터링 서비스

**파일:** `app/services/scope_service.py` (신규)

```python
from sqlalchemy import and_, or_
from sqlalchemy.ext.asyncio import AsyncSession

class ScopeFilter:
    """엔티티 조회 시 scope 기반 필터링"""

    @staticmethod
    def apply(query, model, current_user_id: int | None, scope_param: str = 'all'):
        """
        scope_param:
          'all' → shared + 내 personal (기본값)
          'shared' → shared만
          'personal' → 내 personal만
        """
        if scope_param == 'shared':
            return query.filter(model.scope == 'shared')
        elif scope_param == 'personal':
            if current_user_id is None:
                raise HTTPException(403, "로그인 필요")
            return query.filter(
                model.scope == 'personal',
                model.owner_id == current_user_id
            )
        else:  # 'all'
            if current_user_id is None:
                return query.filter(model.scope == 'shared')
            return query.filter(
                or_(
                    model.scope == 'shared',
                    and_(model.scope == 'personal', model.owner_id == current_user_id)
                )
            )

    @staticmethod
    def can_modify(entity, current_user_id: int, user_role: str) -> bool:
        """수정 권한 확인"""
        if entity.scope == 'personal':
            return entity.owner_id == current_user_id
        else:  # shared
            return user_role == 'admin'
```

---

### Task 4-3: API 엔드포인트 확장

**파일:** `app/api/admin.py`

모든 CRUD 엔드포인트에 scope 파라미터 추가:

**조회 API (GET) 변경:**
```python
# 기존
@router.get("/rules")
async def list_rules(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Rule).filter(Rule.is_active == True).order_by(Rule.priority)
    )

# 변경 후
@router.get("/rules")
async def list_rules(
    scope: str = Query('all', regex='^(all|shared|personal)$'),
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user_optional)
):
    query = select(Rule).filter(Rule.is_active == True).order_by(Rule.priority)
    query = ScopeFilter.apply(query, Rule, current_user.id if current_user else None, scope)
    result = await db.execute(query)
```

**생성 API (POST) 변경:**
```python
@router.post("/rules")
async def create_rule(
    data: dict = Body(...),
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user)
):
    scope = data.get('scope', 'shared')

    # 권한 확인
    if scope == 'shared' and current_user.role != 'admin':
        raise HTTPException(403, "공용 룰 생성은 admin만 가능")

    rule = Rule(
        **data,
        scope=scope,
        owner_id=current_user.id if scope == 'personal' else None
    )
```

**수정/삭제 API (PUT/DELETE) 변경:**
```python
@router.put("/rules/{rule_id}")
async def update_rule(
    rule_id: str,
    data: dict = Body(...),
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user)
):
    rule = await db.get(Rule, rule_id)
    if not ScopeFilter.can_modify(rule, current_user.id, current_user.role):
        raise HTTPException(403, "수정 권한 없음")
    # ... 업데이트 로직
```

**적용 대상 엔드포인트 (9 × 4 = 36개):**

| 엔티티 | GET | POST | PUT | DELETE |
|--------|-----|------|-----|--------|
| Rules | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Repositories | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Knowledge Docs | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Templates | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Macros | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Signatures | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Disclaimers | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Notification Channels | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |
| Notification Routing | scope 필터 | scope+owner | 권한 확인 | 권한 확인 |

---

### Task 4-4: 개인 메일함 설정 API

**파일:** `app/api/admin.py`

```python
# 개인 메일함 설정
@router.get("/personal-settings")
async def get_personal_settings(
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """내 개인 메일함 설정 조회"""
    settings = await db.execute(
        select(PersonalMailboxSettings)
        .filter(PersonalMailboxSettings.agent_id == current_user.id)
    )
    return settings.scalar_one_or_none() or {}

@router.put("/personal-settings")
async def update_personal_settings(
    data: dict = Body(...),
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """내 개인 메일함 설정 저장"""
    settings = await db.execute(
        select(PersonalMailboxSettings)
        .filter(PersonalMailboxSettings.agent_id == current_user.id)
    )
    existing = settings.scalar_one_or_none()

    if existing:
        for k, v in data.items():
            setattr(existing, k, v)
    else:
        existing = PersonalMailboxSettings(agent_id=current_user.id, **data)
        db.add(existing)

    await db.commit()
    return existing
```

---

### Task 4-5: AI 지식 축적 서비스

**파일:** `app/services/knowledge_accumulator.py` (신규)

```python
class KnowledgeAccumulator:
    """메일 답변 → 지식문서 자동 생성"""

    async def suggest_knowledge(
        self,
        email_id: int,
        sent_response: SentResponse,
        instruct_history: list[dict],
        db: AsyncSession
    ) -> dict:
        """
        발송 후 지식 저장 제안

        Returns:
            {
                "suggest": True/False,
                "title": "자동 생성된 제목",
                "category": "faq",
                "content": "정제된 답변 내용",
                "tags": ["배송", "교환"]
            }
        """
        # 1. AI로 답변 내용을 지식문서 형태로 정제
        # 2. 유사 기존 지식문서 검색 (임베딩 유사도)
        # 3. 중복이면 기존 문서 업데이트 제안, 아니면 신규 생성 제안
        pass

    async def create_from_response(
        self,
        email_id: int,
        title: str,
        content: str,
        category: str,
        scope: str,  # 'personal' or 'shared'
        owner_id: int,
        repo_id: int,
        db: AsyncSession
    ) -> KnowledgeDocument:
        """답변 기반 지식문서 생성 + 임베딩"""

        # 1. KnowledgeDocument 생성
        doc = KnowledgeDocument(
            repo_id=repo_id,
            slug=f"auto-{email_id}-{int(time.time())}",
            title=title,
            content=content,
            category=category,
            scope=scope,
            owner_id=owner_id if scope == 'personal' else None,
            is_active=True
        )
        db.add(doc)
        await db.flush()

        # 2. KnowledgeEmbedding 생성
        embeddings = await self._generate_embeddings(content)
        for i, (chunk, emb) in enumerate(embeddings):
            db.add(KnowledgeEmbedding(
                source_type='md_doc',
                source_id=str(doc.id),
                chunk_index=i,
                content=chunk,
                embedding=emb,
                metadata={"scope": scope, "owner_id": owner_id}
            ))

        # 3. 생성 이력 기록
        db.add(KnowledgeCreationLog(
            email_id=email_id,
            email_type='personal' if scope == 'personal' else 'shared',
            knowledge_doc_id=doc.id,
            created_by=owner_id,
            scope=scope
        ))

        await db.commit()
        return doc

    async def detect_pattern(
        self,
        agent_id: int,
        db: AsyncSession
    ) -> dict | None:
        """
        반복 패턴 감지 → 룰 생성 제안

        최근 30일간 같은 사용자가 유사한 메일에 유사한 답변을 3회 이상 했으면
        "이 유형의 메일에 대한 자동 룰을 만드시겠습니까?" 제안
        """
        pass
```

### Task 4-6: 지식 축적 API 엔드포인트

**파일:** `app/api/admin.py`

```python
@router.post("/emails/{email_id}/save-knowledge")
async def save_as_knowledge(
    email_id: int,
    data: dict = Body(...),
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """
    답변을 지식문서로 저장

    Body:
    {
        "title": "배송 지연 시 안내 방법",
        "content": "...(자동 생성된 내용, 편집 가능)",
        "category": "faq",
        "scope": "personal",  // or "shared" (admin만)
        "repo_id": 1
    }
    """
    scope = data.get('scope', 'personal')
    if scope == 'shared' and current_user.role != 'admin':
        raise HTTPException(403, "공용 지식 생성은 admin만 가능")

    accumulator = KnowledgeAccumulator()
    doc = await accumulator.create_from_response(
        email_id=email_id,
        title=data['title'],
        content=data['content'],
        category=data.get('category', 'faq'),
        scope=scope,
        owner_id=current_user.id,
        repo_id=data['repo_id'],
        db=db
    )
    return {"id": doc.id, "slug": doc.slug}

@router.post("/emails/{email_id}/suggest-knowledge")
async def suggest_knowledge(
    email_id: int,
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """답변 발송 후 지식 저장 제안 (AI 자동 생성)"""
    email = await db.get(Email, email_id)
    sent = await db.execute(
        select(SentResponse)
        .filter(SentResponse.email_id == email_id)
        .order_by(SentResponse.created_at.desc())
        .limit(1)
    )
    sent_response = sent.scalar_one_or_none()
    if not sent_response:
        raise HTTPException(404, "발송된 답변이 없습니다")

    accumulator = KnowledgeAccumulator()
    suggestion = await accumulator.suggest_knowledge(
        email_id=email_id,
        sent_response=sent_response,
        instruct_history=[],  # TODO: instruct 히스토리 조회
        db=db
    )
    return suggestion

@router.get("/knowledge-patterns")
async def detect_patterns(
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """반복 패턴 감지 → 룰 생성 제안"""
    accumulator = KnowledgeAccumulator()
    pattern = await accumulator.detect_pattern(current_user.id, db)
    return pattern or {"suggest": False}
```

---

### Task 4-7: 개인 메일 AI 파이프라인 확장

**파일:** `app/services/pipeline.py` — 개인 메일 처리 시 개인 지식 참조

**현재:** `app/api/admin.py:3507+`에 개인 메일 API 존재하지만, AI 처리 시 개인 지식을 별도로 참조하지 않음

**변경:**
```python
# personal-emails/auto-draft 엔드포인트에서
async def personal_auto_draft(email_data, current_user, db):
    # 1. 개인 룰 매칭 (scope='personal', owner_id=current_user.id)
    personal_rules = await db.execute(
        select(Rule).filter(
            Rule.is_active == True,
            Rule.scope == 'personal',
            Rule.owner_id == current_user.id
        ).order_by(Rule.priority)
    )

    # 2. 개인 + 공용 레포지토리 조합
    repos = await db.execute(
        select(KnowledgeRepository).filter(
            KnowledgeRepository.is_active == True,
            or_(
                KnowledgeRepository.scope == 'shared',
                and_(
                    KnowledgeRepository.scope == 'personal',
                    KnowledgeRepository.owner_id == current_user.id
                )
            )
        )
    )

    # 3. 개인 + 공용 지식문서로 AI 처리
    # 개인 지식이 우선순위 높게 (먼저 매칭)
    # → AIProcessor에 repo_ids 전달 시 개인 repo를 앞에 배치
```

---

### Task 4-8: Frontend — 공용/개인 전환 UI

**모든 엔티티 관리 페이지에 공통 적용:**

#### 탭 UI 구조
```
┌──────────────────────────────────────────────────┐
│  [공용]  [내 것]              [+ 새로 만들기]      │
├──────────────────────────────────────────────────┤
│                                                  │
│  공용 탭:                                        │
│    - admin: CRUD 전체 가능                       │
│    - agent: 조회만 (수정/삭제 버튼 숨김)          │
│    - viewer: 조회만                              │
│                                                  │
│  내 것 탭:                                       │
│    - admin/agent: CRUD 전체 가능                  │
│    - viewer: 조회만                              │
│                                                  │
└──────────────────────────────────────────────────┘
```

**적용 페이지 (9개):**
1. `/rules` — 룰 관리
2. `/repositories` — 레포지토리
3. `/knowledge-docs` — 지식 문서
4. `/email-templates` — 이메일 템플릿
5. `/macros` — 매크로
6. `/signatures` — 서명
7. `/disclaimers` — 고지문
8. `/notifications` — 알림 채널
9. (알림 라우팅은 채널 페이지 내 서브 탭)

**공통 컴포넌트:**

```tsx
// components/scope-tabs.tsx
interface ScopeTabsProps {
  activeScope: 'shared' | 'personal';
  onScopeChange: (scope: 'shared' | 'personal') => void;
  sharedCount: number;
  personalCount: number;
}

function ScopeTabs({ activeScope, onScopeChange, sharedCount, personalCount }: ScopeTabsProps) {
  return (
    <Tabs value={activeScope} onValueChange={onScopeChange}>
      <TabsList>
        <TabsTrigger value="shared">
          공용 <Badge variant="secondary">{sharedCount}</Badge>
        </TabsTrigger>
        <TabsTrigger value="personal">
          내 것 <Badge variant="secondary">{personalCount}</Badge>
        </TabsTrigger>
      </TabsList>
    </Tabs>
  );
}
```

**권한별 버튼 표시:**
```tsx
// hooks/use-entity-permissions.ts
function useEntityPermissions(scope: string, ownerId: number | null) {
  const { user } = useAuth();

  return {
    canCreate: scope === 'personal' || user?.role === 'admin',
    canEdit: scope === 'personal'
      ? ownerId === user?.id
      : user?.role === 'admin',
    canDelete: scope === 'personal'
      ? ownerId === user?.id
      : user?.role === 'admin',
  };
}
```

---

### Task 4-9: Frontend — 개인 메일함 설정 페이지

**경로:** `/my-inbox/settings` (내 메일함 내 설정 탭 또는 별도 페이지)

```
┌──────────────────────────────────────────────────┐
│  내 메일함 설정                                   │
├──────────────────────────────────────────────────┤
│                                                  │
│  ■ 서명 설정                                     │
│    [내 서명 선택 ▼]  (개인 서명만 목록)           │
│    [미리보기]                                    │
│                                                  │
│  ■ 고지문 설정                                   │
│    [내 고지문 선택 ▼]  (개인 고지문만 목록)       │
│                                                  │
│  ■ AI 자동 초안                                  │
│    [✓] 자동 초안 생성 활성화                      │
│    신뢰도 기준: [0.70 ▼]                         │
│                                                  │
│  ■ 알림 설정                                     │
│    [내 알림 채널 선택 ▼]                          │
│                                                  │
│  ■ 지식 축적                                     │
│    [□] 발송 시 자동 지식 저장                     │
│    기본 저장 레포: [내 레포지토리 ▼]              │
│                                                  │
│                              [저장]  [초기화]     │
└──────────────────────────────────────────────────┘
```

---

### Task 4-10: Frontend — 지식 저장 모달

**위치:** 메일 발송 완료 후 팝업 (emails, my-inbox 페이지 공통)

```
┌──────────────────────────────────────────────────┐
│  💡 이 답변을 지식으로 저장하시겠습니까?           │
├──────────────────────────────────────────────────┤
│                                                  │
│  제목: [배송 지연 시 고객 안내 방법_________]     │
│        (AI 자동 생성, 편집 가능)                  │
│                                                  │
│  카테고리: [FAQ ▼]  [절차 ▼]  [정책 ▼]           │
│                                                  │
│  내용:                                           │
│  ┌──────────────────────────────────────────┐    │
│  │ (발송된 답변 내용 기반 AI 정제 텍스트)    │    │
│  │ 편집 가능                                │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  저장 위치:                                      │
│    ○ 내 개인 지식 (기본)                         │
│    ○ 공용 지식 (admin만)                         │
│                                                  │
│  레포지토리: [내 기본 레포 ▼]                     │
│                                                  │
│              [저장하기]  [건너뛰기]  [다시 묻지 않기] │
└──────────────────────────────────────────────────┘
```

---

## 5. 의존성 그래프

```
Task 4-1 (DB 마이그레이션)
  ↓
Task 4-2 (ScopeFilter 서비스)
  ↓
┌─────────────────┬──────────────────┐
Task 4-3          Task 4-4           Task 4-5
(API scope 확장)  (개인설정 API)     (지식축적 서비스)
  ↓                 ↓                  ↓
Task 4-8          Task 4-9           Task 4-6
(공용/개인 탭 UI) (개인설정 페이지)   (지식축적 API)
                                      ↓
                                    Task 4-10
                                    (지식저장 모달)
                                      ↓
                                    Task 4-7
                                    (개인 AI 파이프라인)
```

---

## 6. 테스트 체크리스트

### 권한 테스트
- [ ] admin이 공용 룰 생성/수정/삭제 → 성공
- [ ] agent가 공용 룰 생성 → 403 Forbidden
- [ ] agent가 공용 룰 수정 → 403 Forbidden
- [ ] agent가 개인 룰 생성 → 성공 (scope=personal, owner_id=본인)
- [ ] agent가 타인의 개인 룰 수정 → 403 Forbidden
- [ ] viewer가 어떤 것도 생성/수정/삭제 → 403 Forbidden
- [ ] (위 테스트를 9개 엔티티 모두 반복)

### Scope 필터 테스트
- [ ] GET /rules?scope=all → 공용 + 내 개인 룰만 반환
- [ ] GET /rules?scope=shared → 공용 룰만
- [ ] GET /rules?scope=personal → 내 개인 룰만
- [ ] 미인증 사용자 → shared만 반환

### 지식 축적 테스트
- [ ] 메일 답변 발송 → "지식 저장" 제안 표시
- [ ] 지식 저장 → KnowledgeDocument + KnowledgeEmbedding 생성
- [ ] 저장된 지식이 다음 AI 초안에 반영되는지 확인
- [ ] 반복 패턴 감지 → 룰 생성 제안

### 개인 메일함 설정 테스트
- [ ] 개인 서명 설정 → 개인 메일 발송 시 적용
- [ ] 개인 고지문 설정 → 개인 메일 발송 시 적용
- [ ] AI 자동 초안 비활성화 → 초안 생성 안 됨
- [ ] 알림 채널 설정 → 해당 채널로 알림 수신

---

## 7. 구현 순서 (5일)

```
Day 1: Task 4-1 (DB) + Task 4-2 (ScopeFilter)
Day 2: Task 4-3 (API 36개 엔드포인트 scope 확장)
Day 3: Task 4-4 (개인설정 API) + Task 4-5 (지식축적 서비스)
Day 4: Task 4-6 (지식 API) + Task 4-7 (개인 AI 파이프라인)
Day 5: Task 4-8 ~ 4-10 (Frontend 전체)
```
