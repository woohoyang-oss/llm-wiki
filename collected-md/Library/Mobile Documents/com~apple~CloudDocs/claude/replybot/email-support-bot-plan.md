# 이메일 서포트 봇 기획서 — "1차 상담원" 시스템

> TBNWS 내부 프로젝트 · 2026-03-11 · v0.3 (의사결정 확정)

---

## 1. 프로젝트 개요

회사로 유입되는 서포트 이메일을 **자동 분류 → AI 판단 → 회신(또는 승인 대기)**하는 시스템.
단순 문의는 즉시 자동 회신하고, 복잡한 건은 초안을 생성해 담당자 승인 후 발송한다.

### 메일 라우팅 구조 (확정)

```
┌─────────────────────────────────────────────────────────┐
│                  외부 고객 메일 발신                      │
└────────┬──────────────────┬──────────────────┬──────────┘
         │                  │                  │
         ▼                  ▼                  ▼
  서포트주소 A       서포트주소 B        서포트주소 C  ...
  (브랜드A CS)       (브랜드B CS)        (브랜드B B2B)
         │                  │                  │
         │   Gmail alias / forwarding          │
         └──────────┬───────┴──────────────────┘
                    ▼
            ┌──────────────┐
            │  통합 수신함   │  ◄── AI 처리 전용 메일박스
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │  AI 봇 처리   │
            │  분류/판단    │
            └──────┬───────┘
                   │
          회신 시: 원래 수신 주소로 발송 (Send As)
                   │
         ┌─────────┼─────────────────┐
         ▼         ▼                 ▼
  FROM:            FROM:             FROM:
  서포트주소 A      서포트주소 B       서포트주소 C
```

**핵심 원칙:**
- 고객은 통합 수신함의 존재를 모른다. 항상 원래 문의한 주소에서 회신이 온 것처럼 보여야 한다.
- 메일 주소는 **Admin Dashboard에서 자유롭게 추가/수정/삭제** 가능. 코드 배포 없이 운영 중 즉시 반영.
- 새 브랜드/부서가 생겨도 DB 테이블에 한 줄 추가 + Gmail 포워딩 설정만으로 확장.

### 핵심 목표

- 반복 문의 응답 시간 **5분 이내** (현재 수 시간~1일)
- 담당자 업무 부담 **50% 이상 절감**
- 오답/오발송률 **2% 미만** 유지

---

## 2. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│                    Gmail (Google Workspace)                    │
│                                                              │
│  support@aaa.com ─────┐                                       │
│  support@bbb.kr  ─────┼── alias/forwarding ──▶ 통합 수신함    │
│  b2b@ccc.kr      ─────┘                                     │
│  (+ Admin에서 추가한 모든 주소)                                │
│                                                              │
│  ※ 향후 주소 추가 시 forwarding만 설정하면 자동 확장          │
└──────────┬───────────────────────────────────────┬────────────┘
           │ Gmail API (ai@tbe.kr 감시)            │ Gmail API "Send As"
           │ + Pub/Sub push                        │ (원래 주소로 발송)
           ▼                                       ▲
┌──────────────────────────────────────────────────────────────┐
│                    Mail Gateway (AWS EC2)                     │
│                                                              │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │  Ingestion   │──▶│ Rule Engine  │──▶│  AI Processing   │  │
│  │  Service     │   │ (분류/라우팅) │   │  (Claude / Local)│  │
│  └─────────────┘   └──────┬───────┘   └────────┬─────────┘  │
│                           │                     │            │
│                    ┌──────▼─────────────────────▼──────┐     │
│                    │        Action Router               │     │
│                    │  자동발송 | 승인대기 | 에스컬레이션  │     │
│                    └──────┬────────┬────────┬──────────┘     │
│                           │        │        │                │
│  ┌────────────────────────▼────────▼────────▼────────────┐   │
│  │              Response Sender                           │   │
│  │  수신자/참조/숨은참조 지정 · 발신자 매핑 · 서명 삽입    │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Knowledge Layer                           │   │
│  │  MD 문서 폴더 │ PostgreSQL │ MCP 서버 │ 벡터 DB(RAG)   │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Admin Dashboard (웹 UI)                    │   │
│  │  룰 관리 │ 승인 큐 │ 로그/통계 │ 설정                   │   │
│  └────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 핵심 모듈 상세

### 3.1 Ingestion Service (메일 수집)

| 항목 | 설명 |
|------|------|
| 방식 | Gmail API + Google Cloud Pub/Sub (실시간 push) |
| 감시 대상 | **ai@tbe.kr 단일 메일박스** (모든 서포트 메일이 여기로 통합) |
| 파싱 | 제목, 본문(text/html), 첨부파일, 발신자, **원래 수신 주소(original_to)**, Thread ID |
| 중복 방지 | Message-ID 기반 dedup, 이미 처리된 thread 스킵 |
| 저장 | PostgreSQL `emails` 테이블 + 원본 EML S3 백업 |

**원래 수신 주소 식별 (핵심 로직):**

포워딩된 메일에서 "원래 어느 주소로 왔는지" 판별하는 것이 이 시스템의 핵심.
아래 3가지 방법을 우선순위대로 시도:

```python
def detect_original_recipient(message_headers: dict) -> str:
    """포워딩된 메일에서 원래 수신 주소를 식별"""

    # 방법 1: X-Forwarded-To 헤더 (Gmail 포워딩 시 자동 추가)
    if x_forwarded_to := headers.get("X-Forwarded-To"):
        return x_forwarded_to  # "ai@tbe.kr" 이 아닌 원래 주소

    # 방법 2: Delivered-To 헤더 체인 분석
    # 포워딩 시 Delivered-To가 여러 개 → 가장 첫 번째가 원래 수신 주소
    delivered_to_list = headers.get_all("Delivered-To")
    if len(delivered_to_list) > 1:
        return delivered_to_list[0]  # 원래 수신 주소

    # 방법 3: To/Cc 헤더에서 매핑 테이블과 대조
    to_addresses = parse_to_header(headers.get("To"))
    for addr in to_addresses:
        if addr in KNOWN_SUPPORT_ADDRESSES:
            return addr

    # 폴백: 매핑 실패 시 기본 주소 사용 + 알림
    return DEFAULT_REPLY_ADDRESS
```

**메일 주소 매핑은 DB 테이블로 관리** — Admin Dashboard에서 CRUD, 코드 변경 없이 자유롭게 추가/수정/삭제:

```sql
-- 서포트 메일 주소 매핑 (동적 관리)
CREATE TABLE mailbox_config (
    id              SERIAL PRIMARY KEY,
    address         VARCHAR UNIQUE NOT NULL,  -- 서포트 메일 주소
    brand           VARCHAR NOT NULL,         -- 브랜드 구분
    category        VARCHAR NOT NULL,         -- cs, b2b, as 등
    display_name    VARCHAR NOT NULL,         -- 발신 시 표시 이름
    signature_id    VARCHAR,                  -- 서명 템플릿 ID
    default_cc      JSONB DEFAULT '[]',       -- 기본 참조 주소
    default_bcc     JSONB DEFAULT '[]',       -- 기본 숨은참조
    forwarding_to   VARCHAR NOT NULL,         -- 통합 수신함 주소
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 예시 데이터 (실제 주소는 운영 시 설정)
-- INSERT INTO mailbox_config (address, brand, category, display_name, forwarding_to)
-- VALUES ('support@example.com', 'brand_a', 'cs', 'Brand A 고객지원', 'ai@company.com');
```

```python
# 런타임에 DB에서 매핑 로드 (캐시 적용, Admin에서 변경 시 갱신)
class MailboxResolver:
    def __init__(self):
        self._cache = {}
        self._load_from_db()

    def resolve(self, headers: dict) -> MailboxConfig:
        """포워딩된 메일에서 원래 수신 주소를 식별하고 매핑 정보 반환"""
        original_to = self._detect_original_recipient(headers)
        config = self._cache.get(original_to)
        if not config:
            # 매핑 실패 → 알림 발송 + 폴백
            notify_admin(f"Unknown address: {original_to}")
            return self._default_config
        return config

    def reload(self):
        """Admin에서 매핑 변경 시 호출 (webhook or polling)"""
        self._load_from_db()
```

**새 메일 주소 추가 절차 (코드 변경 없음):**

| 단계 | 작업 | 누가 |
|------|------|------|
| 1 | Gmail에서 새 주소 → 통합 수신함으로 포워딩 설정 | Workspace 관리자 |
| 2 | 통합 수신함에서 "Send As"로 새 주소 등록 | Workspace 관리자 |
| 3 | Admin Dashboard에서 매핑 테이블에 새 주소 추가 | 운영 담당자 |
| 4 | (선택) 해당 주소용 서명 템플릿, 기본 CC 설정 | 운영 담당자 |

**Gmail 초기 설정:**

| 단계 | 작업 | 비고 |
|------|------|------|
| 1 | 통합 수신 계정 생성 | Google Workspace 관리 콘솔 |
| 2 | 각 서포트 주소에서 통합 수신함으로 포워딩 설정 | Gmail 설정 > 전달 |
| 3 | 통합 수신함에서 "Send As" 설정 | 각 서포트 주소를 발신 주소로 등록 |
| 4 | Gmail API + Pub/Sub 활성화 | Google Cloud Console |
| 5 | 서비스 계정 생성 + 도메인 전체 위임 | 통합 수신함에 대한 API 접근 |

**구현 포인트:**
- ai@tbe.kr **단일 메일박스만 감시** → 아키텍처 단순화
- Pub/Sub → SQS/Lambda 또는 EC2 워커 프로세스로 수신
- 첨부파일은 S3에 별도 저장, 메타데이터만 DB 기록
- 새 서포트 주소 추가 시: ① 포워딩 설정 ② Send As 등록 ③ 매핑 테이블에 추가 — 코드 변경 불필요

### 3.2 Rule Engine (룰 엔진)

룰은 **JSON 기반 설정 파일** 또는 **DB 테이블**로 관리하며, Admin Dashboard에서 CRUD 가능.

#### 룰 구조 예시

```json
{
  "rule_id": "R001",
  "name": "AS 접수 자동분류",
  "priority": 10,
  "conditions": {
    "operator": "OR",
    "rules": [
      { "field": "subject", "match": "contains", "value": ["AS", "수리", "고장", "불량"] },
      { "field": "body", "match": "contains", "value": ["AS 접수", "수리 요청"] }
    ]
  },
  "actions": {
    "category": "AS",
    "auto_reply": true,
    "confidence_threshold": 0.85,
    "reply_from": "auto",
    "reply_cc": ["as-team@company.com"],
    "template_id": "TPL_AS_RECEIPT",
    "escalate_to": "as-manager@company.com",
    "use_ai": true
  }
}
```

#### 룰 매칭 우선순위

```
1. 정확 매칭 (제목/본문에 특정 키워드 정확히 포함)
2. 패턴 매칭 (정규식, 예: 주문번호 패턴 [A-Z]{2}\d{8})
3. 발신자 매칭 (특정 도메인 or 이메일)
4. AI 분류 (위 룰에 매칭 안 되면 Claude/LLM이 카테고리 판단)
```

#### 주요 액션 타입

| 액션 | 설명 | 예시 |
|------|------|------|
| `auto_reply` | 즉시 자동 회신 | FAQ, 접수 확인 |
| `draft_and_wait` | 초안 생성 → 승인 큐 | 교환/환불, 견적 |
| `escalate` | 담당자에게 전달만 | VIP 고객, 법률 관련 |
| `tag_only` | 라벨만 붙이고 패스 | 광고/스팸, 내부 참고 |
| `forward` | 특정 부서로 전달 | 부서 잘못 온 메일 |

### 3.3 AI Processing (AI 처리 레이어)

#### LLM 엔진 레지스트리 (확정)

Phase 1에서는 Claude API만 사용하고, 이후 로컬 LLM을 **엔드포인트 URL 형태로 자유롭게 추가**.
각 엔진에 우선순위를 지정하여 룰별로 어떤 엔진을 쓸지 제어한다.

```sql
-- LLM 엔진 레지스트리 (Admin Dashboard에서 CRUD)
CREATE TABLE llm_engines (
    id              SERIAL PRIMARY KEY,
    engine_id       VARCHAR UNIQUE NOT NULL,    -- 'claude-sonnet', 'local-qwen', 'local-llama' 등
    name            VARCHAR NOT NULL,           -- 표시 이름
    type            VARCHAR NOT NULL,           -- 'cloud_api' | 'local_openai_compat'
    endpoint_url    VARCHAR NOT NULL,           -- 'https://api.anthropic.com/v1' 또는 'http://x.x.x.x:18787/v1'
    model_name      VARCHAR NOT NULL,           -- 'claude-sonnet-4-6' 또는 'qwen2.5-7b-instruct'
    api_key         VARCHAR,                    -- 암호화 저장 (로컬은 null 가능)
    priority        INT DEFAULT 100,            -- 낮을수록 우선 (10=최우선, 100=기본)
    capabilities    JSONB DEFAULT '[]',         -- ['tool_use', 'classification', 'generation', 'embedding']
    max_tokens      INT DEFAULT 4096,
    timeout_sec     INT DEFAULT 30,
    cost_per_1k_in  DECIMAL,                    -- 입력 토큰 1K당 비용 (비용 추적용)
    cost_per_1k_out DECIMAL,                    -- 출력 토큰 1K당 비용
    is_active       BOOLEAN DEFAULT true,
    health_status   VARCHAR DEFAULT 'unknown',  -- 'healthy' | 'degraded' | 'down'
    last_health_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

```
엔진 레지스트리 예시:

┌──────────────────┬─────────────────────────────────────┬──────────┐
│ engine_id        │ endpoint_url                        │ priority │
├──────────────────┼─────────────────────────────────────┼──────────┤
│ claude-sonnet    │ https://api.anthropic.com/v1        │ 10       │  ◄ Phase 1 기본
│ claude-opus      │ https://api.anthropic.com/v1        │ 20       │  ◄ 고난도 전용
│ local-qwen       │ http://10.0.1.50:18787/v1           │ 50       │  ◄ Phase 2 추가
│ local-llama      │ http://10.0.1.50:18788/v1           │ 60       │  ◄ Phase 2 추가
│ local-embed      │ http://10.0.1.50:18789/v1           │ 30       │  ◄ 임베딩 전용
└──────────────────┴─────────────────────────────────────┴──────────┘
```

#### 엔진 선택 로직

```python
class LLMRouter:
    """룰 설정 또는 작업 유형에 따라 최적 엔진 선택"""

    def select_engine(self, task: str, rule: Rule = None) -> LLMEngine:
        # 1) 룰에 엔진이 명시되어 있으면 해당 엔진 사용
        if rule and rule.actions.get("engine_id"):
            engine = self.registry.get(rule.actions["engine_id"])
            if engine and engine.is_healthy():
                return engine

        # 2) 작업 유형(capability)에 맞는 엔진 중 우선순위 최상위
        capable = self.registry.find_by_capability(task)  # 'tool_use', 'classification' 등
        healthy = [e for e in capable if e.is_healthy()]
        return sorted(healthy, key=lambda e: e.priority)[0]

    def health_check(self):
        """1분 주기 헬스체크 — 장애 시 자동 페일오버"""
        for engine in self.registry.all_active():
            try:
                resp = httpx.post(f"{engine.endpoint_url}/health", timeout=5)
                engine.update_health('healthy')
            except:
                engine.update_health('down')
                notify_admin(f"LLM engine {engine.name} is DOWN")
```

#### 룰에서 엔진 지정 예시

```json
{
  "rule_id": "R001",
  "actions": {
    "engine_id": "claude-sonnet",
    "use_ai": true
  }
}
```

```json
{
  "rule_id": "R010",
  "name": "단순 FAQ 분류",
  "actions": {
    "engine_id": "local-qwen",
    "use_ai": true
  }
}
```

엔진을 지정하지 않으면 LLMRouter가 capability + priority 기반으로 자동 선택.
엔진 장애 시 자동으로 다음 우선순위 엔진으로 페일오버.

#### Claude Tool Use 활용 (MCP 연동)

Claude에게 **도구(tool)**를 부여하여 실시간 데이터 조회 후 답변 생성:

```python
# Claude가 사용할 수 있는 도구 예시
tools = [
    {
        "name": "lookup_order",
        "description": "주문번호로 주문 상태/배송 정보 조회",
        "input_schema": { "order_id": "string" }
    },
    {
        "name": "check_inventory",
        "description": "상품 코드로 현재 재고 확인",
        "input_schema": { "product_code": "string" }
    },
    {
        "name": "get_as_status",
        "description": "AS 접수번호로 진행 상태 확인",
        "input_schema": { "as_id": "string" }
    },
    {
        "name": "search_knowledge_base",
        "description": "내부 문서에서 관련 정보 검색 (RAG)",
        "input_schema": { "query": "string" }
    },
    {
        "name": "get_product_info",
        "description": "상품 스펙, 가격, 호환성 정보 조회",
        "input_schema": { "product_name": "string" }
    }
]
```

> **참고:** Tool Use는 현재 Claude API만 지원. 로컬 LLM은 function calling 지원 모델(Qwen2.5 등) 사용 시 별도 어댑터로 래핑.

### 3.4 Knowledge Layer (지식 베이스)

#### 데이터 소스 통합

```
┌────────────────────────────────────────────────┐
│              Knowledge Layer                    │
│                                                │
│  ① MD 문서 폴더                                │
│     /knowledge/                                │
│       ├── faq/           # FAQ 문서            │
│       ├── products/      # 제품 스펙            │
│       ├── policies/      # 교환/환불 정책       │
│       ├── templates/     # 응답 템플릿          │
│       └── guides/        # AS 가이드            │
│                                                │
│  ② DB 직접 연결                                 │
│     - PostgreSQL: 주문, 고객, AS 이력           │
│     - 기존 TBNWS 데이터: ERP/CRM 연동          │
│                                                │
│  ③ MCP 서버 연결                                │
│     - Notion MCP → 내부 위키/매뉴얼             │
│     - Gmail MCP → 과거 응대 이력               │
│     - 커스텀 MCP → 자체 API 래핑               │
│                                                │
│  ④ 벡터 검색 (RAG) — PostgreSQL + pgvector       │
│     - 별도 벡터 DB 없이 기존 PostgreSQL 활용     │
│     - MD 문서 + 과거 응대 사례 임베딩           │
│     - 유사 문의 검색 → 참고 답변 제공           │
└────────────────────────────────────────────────┘
```

#### RAG 파이프라인 (pgvector 기반)

별도 벡터 DB를 두지 않고 **PostgreSQL + pgvector 확장**으로 통합 운영.
인프라 단순화 + 기존 DB 운영 노하우 그대로 활용.

```sql
-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- 지식 문서 임베딩 테이블
CREATE TABLE knowledge_embeddings (
    id              BIGSERIAL PRIMARY KEY,
    source_type     VARCHAR NOT NULL,          -- 'md_doc', 'past_response', 'faq', 'product_spec'
    source_id       VARCHAR NOT NULL,          -- 원본 문서 식별자
    chunk_index     INT DEFAULT 0,             -- 문서 분할 인덱스
    content         TEXT NOT NULL,             -- 원본 텍스트 청크
    embedding       vector(1024) NOT NULL,     -- 임베딩 벡터 (BGE-M3 = 1024차원)
    metadata        JSONB DEFAULT '{}',        -- 브랜드, 카테고리, 태그 등
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW 인덱스 (검색 속도 최적화)
CREATE INDEX ON knowledge_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

```
수신 메일 → 임베딩 → pgvector 유사도 검색(top-k) → 관련 문서 추출
                                                       ↓
                                         Claude에게 컨텍스트로 전달
                                                       ↓
                                         답변 생성 (근거 명시)
```

### 3.5 Response Sender (발송 모듈)

#### 발신자 결정 로직

발신 주소는 하드코딩하지 않고, **수신 시 식별된 원래 주소(original_to)를 그대로 사용:**

```python
def build_reply(email: IncomingEmail, draft: str) -> OutgoingEmail:
    # mailbox_config DB에서 원래 수신 주소의 설정을 가져옴
    config = mailbox_resolver.resolve(email.original_to)

    return OutgoingEmail(
        from_address=config.address,           # 원래 수신 주소로 회신
        from_display=config.display_name,      # 설정된 표시 이름
        to=[email.from_address],               # 원래 보낸 사람에게
        cc=config.default_cc + rule.extra_cc,  # 기본 CC + 룰 추가 CC
        bcc=config.default_bcc,                # 로그용 숨은참조
        thread_id=email.thread_id,             # 같은 스레드에 회신
        signature=load_signature(config.signature_id),
        body=draft,
    )
```

**핵심:** 메일 주소가 추가/변경되어도 이 코드는 변경할 필요 없음. `mailbox_config` DB 테이블만 업데이트하면 자동 반영.

#### 발송 옵션 (룰별 오버라이드 가능)

```yaml
# 룰에서 기본 설정을 오버라이드 할 수 있음
send_config:
  from: "auto"                   # 원래 수신 주소 자동 사용 (기본값)
  to: ["original_sender"]        # 원래 발신자에게 회신
  cc: ["추가참조@example.com"]    # 룰에서 추가 CC 지정 가능
  bcc: []                        # 룰에서 추가 BCC 지정 가능
  reply_to_thread: true          # 같은 스레드에 회신
  delay_seconds: 0               # 발송 딜레이 (0 = 즉시)
```

#### AI 작성 고지문 시스템 (확정: 고지 포함, CRUD 관리)

발송 메일 하단에 AI 작성 고지문을 삽입한다. 고지문은 Admin Dashboard에서 추가/수정/삭제 가능.

```sql
-- AI 고지문 템플릿 (Admin에서 CRUD)
CREATE TABLE ai_disclaimers (
    id              SERIAL PRIMARY KEY,
    disclaimer_id   VARCHAR UNIQUE NOT NULL,
    name            VARCHAR NOT NULL,           -- '기본 고지문', '영문 고지문' 등
    content_html    TEXT NOT NULL,              -- HTML 형태 고지문
    content_text    TEXT NOT NULL,              -- 평문 형태 고지문
    language        VARCHAR DEFAULT 'ko',       -- 'ko', 'en', 'ja' 등
    is_default      BOOLEAN DEFAULT false,      -- 기본 고지문 여부
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

고지문은 **메일박스별** 또는 **룰별**로 다르게 지정 가능:

```python
def attach_disclaimer(email: OutgoingEmail, config: MailboxConfig, rule: Rule) -> str:
    # 우선순위: 룰 지정 > 메일박스 지정 > 기본 고지문
    disclaimer_id = (
        rule.actions.get("disclaimer_id")
        or config.disclaimer_id
        or get_default_disclaimer().disclaimer_id
    )
    disclaimer = load_disclaimer(disclaimer_id)

    # 고지문 없이 보내야 하는 경우 (룰에서 명시적 비활성화)
    if rule.actions.get("skip_disclaimer"):
        return email.body

    return f"{email.body}\n\n---\n{disclaimer.content_html}"
```

예시 고지문:
> 본 메일은 AI 기반 고객지원 시스템에 의해 작성되었습니다.
> 내용에 오류가 있거나 추가 도움이 필요하시면 회신해 주세요.

#### Gmail "Send As" 발송 흐름

```
봇이 회신 결정
     │
     ▼
Gmail API users.messages.send 호출
  - userId: "ai@통합수신함"  (API 인증 주체)
  - From 헤더: "support@원래주소"  (Send As 기능)
  - In-Reply-To: 원래 메일 Message-ID
  - References: 스레드 체인
     │
     ▼
고객에게는 "support@원래주소" 에서 온 것으로 표시
```

### 3.6 Admin Dashboard (확정: 자체 개발, 사용성 + 유지보수 우선)

**기술 선택:** Next.js 15 + shadcn/ui + Tailwind CSS
MVP부터 자체 개발. Retool 같은 로코드 대신 처음부터 자체 구축하여 장기 유지보수성 확보.
shadcn/ui 컴포넌트 라이브러리로 개발 속도와 UX 품질 동시 확보.

#### 핵심 화면

| 화면 | 기능 |
|------|------|
| **대시보드** | 오늘 처리 건수, 자동/수동 비율, 평균 응답시간, 엔진별 사용량/비용 |
| **승인 큐** | AI 초안 확인 → 수정/승인/반려. 원문 메일 + AI 초안 + 참고 문서 나란히 표시 |
| **룰 관리** | 룰 CRUD, 우선순위 드래그, 조건 빌더 (GUI), 테스트 시뮬레이션 |
| **메일박스 관리** | 서포트 주소 CRUD, 발신자 매핑, 서명 템플릿, 기본 CC/BCC |
| **LLM 엔진 관리** | 엔진 등록/수정/삭제, 우선순위 설정, 헬스 상태, 비용 추적 |
| **AI 고지문 관리** | 고지문 CRUD, 메일박스별/룰별 매핑, 미리보기 |
| **지식 베이스** | MD 문서 업로드/편집, 벡터 재인덱싱 트리거, 검색 테스트 |
| **로그/이력** | 전체 처리 이력, 필터 검색, 성능 통계, 비용 분석 |
| **알림 채널 관리** | 알림 채널 등록 (Slack/Telegram/웹훅), 이벤트별 채널 매핑 |
| **설정** | 시스템 전역 설정, 사용자 관리, SSO 설정 |

### 3.7 알림/메시징 채널 (확정: 웹 승인 기본 + 외부 채널 확장)

**핵심 원칙:** 승인/리뷰 등 모든 핵심 기능은 **웹 Admin Dashboard**에서 완결.
Slack, Telegram 등은 **알림 + 간단 인터랙션** 채널로 확장 연동.

```sql
-- 알림 채널 레지스트리 (Admin에서 CRUD)
CREATE TABLE notification_channels (
    id              SERIAL PRIMARY KEY,
    channel_id      VARCHAR UNIQUE NOT NULL,
    type            VARCHAR NOT NULL,           -- 'slack', 'telegram', 'webhook', 'email'
    name            VARCHAR NOT NULL,
    config          JSONB NOT NULL,             -- 채널별 설정 (webhook URL, bot token 등)
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 이벤트-채널 매핑 (어떤 이벤트를 어디로 알릴지)
CREATE TABLE notification_routing (
    id              SERIAL PRIMARY KEY,
    event_type      VARCHAR NOT NULL,           -- 'approval_needed', 'escalation', 'auto_sent', 'error', 'daily_report'
    channel_id      VARCHAR REFERENCES notification_channels(channel_id),
    template        TEXT,                       -- 알림 메시지 템플릿 (변수 치환)
    is_active       BOOLEAN DEFAULT true
);
```

```
알림 이벤트 흐름:

┌──────────────────────┐
│  이벤트 발생          │
│  (승인 필요, 에러 등) │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  Notification Router  │
│  이벤트 → 채널 매핑   │
└──────┬───────────────┘
       │
       ├──▶ Slack (승인 요청 알림 + 버튼으로 바로 웹 이동)
       ├──▶ Telegram (에스컬레이션 알림)
       ├──▶ Webhook (외부 시스템 연동)
       └──▶ Email (일일/주간 리포트)
```

**Slack 연동 예시:**
- 승인 필요 시 → Slack 채널에 메시지 + "승인하기" 버튼 (클릭 시 Admin Dashboard로 이동)
- 에스컬레이션 시 → 담당자 DM
- 일일 리포트 → 지정 채널에 자동 포스팅

**Telegram 연동 예시:**
- 봇을 통해 승인 필요 알림 수신
- 인라인 버튼으로 승인/반려 (간단한 건)
- 복잡한 수정은 웹으로 유도

**새 채널 추가:** Notification Channel 인터페이스를 구현하는 어댑터만 작성하면 됨.
```python
class NotificationChannel(ABC):
    @abstractmethod
    async def send(self, event: Event, template: str) -> bool: ...

class SlackChannel(NotificationChannel): ...
class TelegramChannel(NotificationChannel): ...
class WebhookChannel(NotificationChannel): ...
```

---

## 4. 처리 플로우 (상세)

### 4.1 전체 플로우

```
메일 수신
  │
  ▼
[1] 파싱 & 저장
  │  - 제목/본문/발신자/첨부 추출
  │  - DB 저장 + 원본 백업
  │
  ▼
[2] 룰 엔진 매칭
  │  - 우선순위순으로 룰 체크
  │  - 매칭된 룰의 category + action 결정
  │
  ├─ 매칭됨 ──────────────────────────┐
  │                                    │
  ▼                                    ▼
[3-A] 정형 응답                    [3-B] AI 응답 생성
  │  - 템플릿 기반                   │  - Knowledge Layer 조회
  │  - 변수 치환만                   │  - Claude/Local LLM 호출
  │  (접수번호, 이름 등)             │  - Tool Use로 데이터 조회
  │                                  │  - 답변 초안 생성
  │                                  │
  ▼                                  ▼
[4] 자동화 레벨 판단
  │
  ├─ confidence ≥ 0.85 + auto_reply ──▶ [5-A] 즉시 발송
  │
  ├─ confidence ≥ 0.7 ────────────────▶ [5-B] 승인 큐 (초안 + 근거)
  │                                          │
  │                                    담당자 승인/수정
  │                                          │
  │                                          ▼
  │                                      발송 or 반려
  │
  └─ confidence < 0.7 ────────────────▶ [5-C] 에스컬레이션
                                         (담당자 직접 처리)
```

### 4.2 Confidence Score 산정

```
최종 confidence = 가중 평균(
  룰 매칭 정확도    × 0.3,
  AI 자체 확신도    × 0.3,
  유사 사례 유사도  × 0.2,
  고객 이력 안정성  × 0.1,
  민감도 역보정     × 0.1
)
```

| 점수 구간 | 동작 | 예시 |
|-----------|------|------|
| 0.85~1.0 | 자동 발송 | "배송 언제 되나요?" → 운송장 조회 후 자동 답변 |
| 0.70~0.84 | 초안 → 승인 | "제품이 불량인데 교환해주세요" → 교환 안내 초안 |
| 0.50~0.69 | 에스컬레이션 | "계약 조건 변경 논의" → B2B 담당자 직접 처리 |
| 0.00~0.49 | 태깅만 | 판별 불가, 스팸 의심 등 |

---

## 5. 기술 스택 (확정)

| 영역 | 기술 | 이유 |
|------|------|------|
| **백엔드** | Python 3.12 + FastAPI | 비동기, LLM 생태계 최적 |
| **메일 연동** | Gmail API + Google Pub/Sub | 실시간 push, Workspace 통합 |
| **AI (Phase 1)** | Anthropic Claude API (Sonnet/Opus) | Tool Use, 한국어 품질 |
| **AI (Phase 2+)** | vLLM/Ollama + OpenAI-compat API | `http://x.x.x.x:port/v1` 형태로 추가 |
| **벡터 검색** | PostgreSQL 16 + pgvector | 별도 DB 없이 기존 PG 활용, 운영 단순화 |
| **임베딩** | BGE-M3 (로컬) 또는 Voyage AI | 한국어 지원, 1024차원 |
| **DB** | PostgreSQL 16 | 전체 데이터 + 벡터 검색 통합 |
| **큐** | Redis + Celery | 비동기 작업 처리 |
| **프론트** | Next.js 15 + shadcn/ui + Tailwind | 사용성 + 유지보수성 |
| **알림** | 채널 추상화 (Slack/Telegram/Webhook/Email) | 플러그인 방식 확장 |
| **인프라** | AWS EC2 + S3 + RDS | 기존 AWS 인프라 활용 |
| **모니터링** | Grafana + CloudWatch | 처리량, 에러율, 비용 추적 |

---

## 6. 데이터 모델 (주요 테이블)

```sql
-- 수신 메일
CREATE TABLE emails (
    id              BIGSERIAL PRIMARY KEY,
    message_id      VARCHAR UNIQUE,
    thread_id       VARCHAR,
    original_to     VARCHAR,          -- 원래 수신 주소 (포워딩 전)
    mailbox_brand   VARCHAR,          -- mailbox_config에서 조회된 브랜드
    mailbox_type    VARCHAR,          -- mailbox_config에서 조회된 타입 (cs/b2b/as)
    from_address    VARCHAR,
    to_addresses    JSONB,
    cc_addresses    JSONB,
    subject         TEXT,
    body_text       TEXT,
    body_html       TEXT,
    attachments     JSONB,
    received_at     TIMESTAMPTZ,
    category        VARCHAR,          -- 룰엔진 분류 결과
    matched_rule_id VARCHAR,
    status          VARCHAR DEFAULT 'pending',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 룰 설정
CREATE TABLE rules (
    id              SERIAL PRIMARY KEY,
    rule_id         VARCHAR UNIQUE,
    name            VARCHAR,
    priority        INT,
    conditions      JSONB,
    actions         JSONB,
    is_active       BOOLEAN DEFAULT true,
    created_by      VARCHAR,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- AI 처리 결과
CREATE TABLE ai_responses (
    id              BIGSERIAL PRIMARY KEY,
    email_id        BIGINT REFERENCES emails(id),
    llm_provider    VARCHAR,           -- 'claude' or 'local'
    model           VARCHAR,
    prompt_tokens   INT,
    response_tokens INT,
    confidence      FLOAT,
    draft_body      TEXT,
    tools_used      JSONB,             -- 사용된 도구 목록
    knowledge_refs  JSONB,             -- 참고한 문서 목록
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 발송 이력
CREATE TABLE sent_responses (
    id              BIGSERIAL PRIMARY KEY,
    email_id        BIGINT REFERENCES emails(id),
    ai_response_id  BIGINT REFERENCES ai_responses(id),
    from_address    VARCHAR,
    to_addresses    JSONB,
    cc_addresses    JSONB,
    bcc_addresses   JSONB,
    subject         TEXT,
    body            TEXT,
    send_type       VARCHAR,           -- 'auto', 'approved', 'manual'
    approved_by     VARCHAR,
    sent_at         TIMESTAMPTZ,
    gmail_msg_id    VARCHAR
);

-- 승인 큐
CREATE TABLE approval_queue (
    id              BIGSERIAL PRIMARY KEY,
    email_id        BIGINT REFERENCES emails(id),
    ai_response_id  BIGINT REFERENCES ai_responses(id),
    assignee        VARCHAR,
    status          VARCHAR DEFAULT 'pending', -- pending/approved/rejected/edited
    edited_body     TEXT,
    reviewed_at     TIMESTAMPTZ,
    reviewer        VARCHAR
);

-- ※ 아래 테이블들은 각 모듈 상세 섹션에서 정의됨 (여기서는 목록만 참조)
-- mailbox_config        → 3.1 Ingestion Service
-- llm_engines           → 3.3 AI Processing
-- ai_disclaimers        → 3.5 Response Sender
-- knowledge_embeddings  → 3.4 Knowledge Layer (pgvector)
-- notification_channels → 3.7 알림/메시징 채널
-- notification_routing  → 3.7 알림/메시징 채널
```

---

## 7. MCP 서버 구성안

자체 MCP 서버를 구축하여 Claude가 실시간으로 내부 데이터에 접근:

```json
{
  "mcpServers": {
    "tbnws-orders": {
      "description": "주문/배송 조회",
      "tools": ["lookup_order", "track_shipping", "get_order_history"]
    },
    "tbnws-inventory": {
      "description": "재고/상품 조회",
      "tools": ["check_inventory", "get_product_info", "get_product_price"]
    },
    "tbnws-as": {
      "description": "AS 현황 조회",
      "tools": ["get_as_status", "get_as_history", "lookup_warranty"]
    },
    "tbnws-knowledge": {
      "description": "내부 문서 검색 (RAG)",
      "tools": ["search_docs", "get_faq", "get_policy"]
    },
    "tbnws-crm": {
      "description": "고객 정보 조회",
      "tools": ["get_customer_info", "get_customer_history"]
    }
  }
}
```

---

## 8. 구현 로드맵

### Phase 1 — MVP (5주)

> 목표: 통합 수신함 + 룰 기반 + Claude + 웹 Admin 동작 확인

| 주차 | 작업 |
|------|------|
| 1주 | DB 스키마 전체 구축 (pgvector 포함), Gmail API 연동, 메일 수집/파싱 |
| 2주 | 룰 엔진 구현, 메일박스 매핑, Send As 발송 모듈 |
| 3주 | Claude API 연동 (LLM 엔진 레지스트리), 기본 RAG (MD 문서 → pgvector) |
| 4주 | Admin Dashboard MVP (승인 큐, 룰 관리, 메일박스 관리, 로그) |
| 5주 | AI 고지문 시스템, 통합 테스트, 실 메일 드라이런 |

**Phase 1 산출물:**
- 통합 수신함 연동, 3개 서포트 주소 포워딩 동작
- 룰 기반 자동/승인 분기 동작
- Claude로 초안 생성 + 승인 큐
- 웹 Admin에서 승인/관리 가능
- AI 고지문 포함 발송

### Phase 2 — 확장 (4주)

> 목표: 로컬 LLM 추가, MCP 연동, 알림 채널 확장

| 주차 | 작업 |
|------|------|
| 6주 | LLM 엔진 레지스트리 확장 (로컬 LLM 연동), AI Router + 헬스체크 |
| 7주 | MCP 서버 구축 (주문/재고/AS 조회), Tool Use 연동 |
| 8주 | 알림 채널 추상화 + Slack 어댑터 + Telegram 어댑터 |
| 9주 | Admin Dashboard 완성 (LLM 관리, 알림 관리, 고지문 CRUD, 통계) |

### Phase 3 — 안정화 (3주)

> 목표: 운영 안정성, 모니터링, 최적화

| 주차 | 작업 |
|------|------|
| 10주 | Grafana 모니터링, 에러 핸들링 강화, 성능 튜닝 |
| 11주 | 피드백 루프 (오답 사례 → 룰/프롬프트 자동 개선 제안) |
| 12주 | 부하 테스트, 문서화, 운영 매뉴얼 |

---

## 9. 보안 및 운영 고려사항

### 보안

- Gmail 서비스 계정은 최소 권한 원칙 적용 (읽기 + 발송만)
- 고객 개인정보는 DB 암호화 저장 (AES-256)
- Claude API 호출 시 PII 마스킹 옵션 검토
- Local LLM 처리 시 데이터 외부 유출 없음 보장
- Admin Dashboard 접근은 회사 SSO 인증 필수

### 운영

- 자동 발송 메일에 AI 작성 고지문 포함 (Admin에서 CRUD 관리)
- 오답 발생 시 즉시 자동발송 중단 → 전체 승인 모드 전환 (kill switch)
- 일일 리포트: 처리 건수, 자동/승인/에스컬레이션 비율, 비용
- 주간 리뷰: 오답 사례 분석 → 룰/프롬프트 개선

### 비용 추정 (월간, 일 100건 기준)

| 항목 | Phase 1 | Phase 2+ |
|------|---------|----------|
| Claude API (Sonnet, 일 ~50건) | ~$50~100 | ~$30~70 (로컬 분담) |
| AWS EC2 (t3.xlarge) | ~$150 | ~$150 + GPU 인스턴스 |
| RDS PostgreSQL (+ pgvector) | ~$50 | ~$70 (벡터 데이터 증가) |
| 기타 (S3, Pub/Sub 등) | ~$20 | ~$20 |
| **합계** | **~$270~320/월** | **~$270~500+/월** |

---

## 10. 의사결정 사항 (확정)

| # | 항목 | 결정 | 상세 |
|---|------|------|------|
| 1 | **LLM 엔진** | Phase 1 Claude만 → 이후 `host:port/엔진명`으로 추가, 우선순위 지정 | `llm_engines` 테이블에서 CRUD, 페일오버 자동 |
| 2 | **AI 작성 고지** | 고지 포함, Admin에서 CRUD(추가/수정/삭제) | `ai_disclaimers` 테이블, 메일박스별/룰별 다르게 적용 가능 |
| 3 | **자동발송 범위** | 룰 설정에 따라 동작 (룰이 곧 범위) | 각 룰의 `auto_reply` + `confidence_threshold`로 제어 |
| 4 | **Admin Dashboard** | 자체 개발 (Next.js + shadcn/ui) | 사용성 + 유지보수 우선, MVP부터 자체 구축 |
| 5 | **알림/대화 채널** | 웹에서 전체 기능 + Slack/Telegram 등 플러그인 확장 | 채널 추상화 레이어, 어댑터 패턴 |
| 6 | **벡터 DB** | PostgreSQL + pgvector (자체 DB 활용) | 별도 인프라 없이 기존 PG에 통합 |

---

## 11. 성공 지표 (KPI)

| 지표 | 목표 (3개월 후) |
|------|----------------|
| 자동 처리율 | 전체 메일의 40% 이상 |
| 평균 응답 시간 (자동) | 5분 이내 |
| 평균 응답 시간 (승인) | 30분 이내 |
| 오답률 | 2% 미만 |
| 담당자 업무 절감 | 50% 이상 |
| 고객 만족도 (CSAT) | 현재 대비 유지 또는 개선 |

---

*v0.3 — 6개 의사결정 확정 반영 완료. 다음 단계: 상세 설계서(API 스펙, 화면 와이어프레임, 프롬프트 엔지니어링) 작성.*
