# 데이터 연동 × 자동화 시나리오 설계 — v1

> 작성일: 2026-03-13
> 목적: 다양한 회사 데이터를 활용한 이메일 자동화 업무 시나리오

---

## 1. 시스템 아키텍처 — 데이터 흐름

```
┌─────────────────────────────────────────────────────────┐
│                    외부 데이터 소스                       │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│ TBNWS    │ 사방넷   │ 네이버   │ ERP      │ 외부 API    │
│ MySQL    │ 주문DB   │ 커머스   │ (발주/   │ (택배사/    │
│ (고객/   │ (멀티    │ (스마트  │  재고/   │  견적시스템 │
│  AS/     │  채널    │  스토어) │  원가)   │  등)       │
│  주문)   │  주문)   │          │          │            │
└──────┬───┴──────┬───┴────┬─────┴────┬─────┴──────┬──────┘
       │          │        │          │            │
       ▼          ▼        ▼          ▼            ▼
┌─────────────────────────────────────────────────────────┐
│              MCP 서버 (KnowledgeRepository)              │
│                                                         │
│  repo: tbnws-crm     → MySQL read-only 연결            │
│  repo: sabangnet     → 사방넷 API 또는 DB               │
│  repo: naver-commerce → 네이버 API                      │
│  repo: erp-system    → ERP DB read-only                 │
│  repo: shipping-api  → 택배사 조회 API                  │
│  repo: internal-docs → 내부 지식문서 (마크다운)          │
│                                                         │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                   AI Processor (Claude)                  │
│                                                         │
│  1. 메일 수신 → 룰 매칭 → 카테고리 분류                │
│  2. 관련 MCP 서버 선택 (룰의 knowledge_repos)            │
│  3. tool_use로 데이터 조회                              │
│     - 고객 정보, 주문 이력, 배송 상태, 재고, 발주 현황  │
│  4. 지식문서 + 실시간 데이터 기반 답변 생성             │
│  5. confidence 기반 자동/반자동 발송                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 업무 시나리오별 데이터 활용

### 시나리오 1: 배송 문의

**수신 메일 예시:**
> "주문한 키크론 Q1이 아직 안 왔습니다. 주문번호: ORD-2026-1234"

**AI 처리 흐름:**
```
1. 룰 매칭: subject contains "배송" → category: shipping
2. MCP 조회:
   - tbnws-crm: 고객 이메일로 주문 조회
   - sabangnet: 주문번호로 배송상태 조회 (송장번호, 택배사)
   - shipping-api: 송장번호로 실시간 배송 추적
3. 컨텍스트 조합:
   - 주문상태: 발송완료 (2026-03-11)
   - 택배사: CJ대한통운, 송장: 1234567890
   - 배송현황: 배달 중 (서울 강남 HUB 도착)
   - 예상 배달: 오늘 18시까지
4. AI 답변 생성 (confidence: 0.92):
   "고객님, 주문하신 키크론 Q1은 현재 배달 중입니다.
    CJ대한통운 송장번호 1234567890으로 조회 가능하며,
    오늘 18시까지 배달 예정입니다."
5. → 자동 발송 (confidence ≥ 0.85)
```

**필요 룰 설정:**
```json
{
  "conditions": {
    "operator": "OR",
    "rules": [
      {"field": "subject_or_body", "match": "contains", "value": ["배송", "택배", "언제 오", "안 왔"]},
      {"field": "subject_or_body", "match": "regex", "value": "주문번호.*ORD-"}
    ]
  },
  "actions": {
    "category": "shipping",
    "action": "auto_reply",
    "knowledge_repos": ["tbnws-crm", "sabangnet", "shipping-api", "internal-docs"]
  }
}
```

---

### 시나리오 2: 교환/반품 요청

**수신 메일 예시:**
> "키크론 K2 키보드의 스페이스바가 불량입니다. 교환 요청합니다."

**AI 처리 흐름:**
```
1. 룰 매칭: body contains "불량" + "교환" → category: exchange
2. MCP 조회:
   - tbnws-crm: 고객 정보 + 주문 이력 + AS 이력
   - erp-system: 해당 제품 재고 확인 (교환 가능 여부)
   - internal-docs: 교환/반품 절차 가이드
3. 컨텍스트 조합:
   - 구매일: 2026-02-15 (보증기간 내)
   - 교환 가능 재고: 3개
   - AS 이력: 없음 (첫 AS)
   - 절차: 제품 사진 → AS접수 → 수거 → 교환품 발송
4. AI 답변 생성 (confidence: 0.78):
   "고객님, 불편을 드려 죄송합니다.
    보증기간 내 제품이므로 무상 교환이 가능합니다.
    아래 절차로 진행됩니다: ..."
5. → 승인 큐 (0.70 ≤ confidence < 0.85)
6. 담당자 확인 후 승인 → 발송
```

---

### 시나리오 3: 견적 요청 (B2B)

**수신 메일 예시:**
> "안녕하세요, ABC 회사입니다. 키크론 K8 Pro 50대 단체 구매 견적을 요청합니다."

**AI 처리 흐름:**
```
1. 룰 매칭: body contains "견적" + quantity > 10 → category: b2b_quote
2. MCP 조회:
   - erp-system: K8 Pro 현재 재고, 원가, 소비자가
   - tbnws-crm: ABC 회사 거래 이력 (기존 고객 여부, 이전 견적)
   - internal-docs: B2B 할인 정책 (수량별 할인율)
3. 컨텍스트 조합:
   - 재고: 충분 (120개)
   - 소비자가: ₩139,000
   - B2B 50대 할인율: 15% (정책 기준)
   - 견적가: ₩118,150 × 50 = ₩5,907,500 (VAT 별도)
   - 이전 거래: 없음 (신규)
4. AI 답변 생성 (confidence: 0.65):
   "ABC 회사 담당자님께,
    키크론 K8 Pro 50대 단체 구매 견적서를 아래와 같이 안내드립니다: ..."
5. → 에스컬레이션 (confidence < 0.70)
6. B2B 담당자가 확인 → 수정 → 발송
```

**추가 필요 기능:**
- 견적서 PDF 자동 생성 (선택)
- 견적 이력 관리 (재견적 시 이전 견적 참조)

---

### 시나리오 4: AS 접수

**수신 메일 예시:**
> "키크론 EX75를 사용 중인데 블루투스 연결이 자주 끊깁니다."

**AI 처리 흐름:**
```
1. 룰 매칭: body contains "AS" OR 불량 증상 키워드 → category: after_service
2. MCP 조회:
   - tbnws-crm: 고객 주문이력 + AS 이력 + 보증 상태
   - internal-docs: EX75 블루투스 문제 해결 가이드
   - erp-system: AS 센터 현황 (대기 건수)
3. 컨텍스트 조합:
   - 구매일: 2025-12-01 (보증 6개월 남음)
   - 기존 AS: 1건 (2026-01-15, 키캡 교체)
   - 자가해결: 펌웨어 업데이트로 해결 가능 (해결율 70%)
4. AI 답변 생성:
   "먼저 펌웨어 업데이트를 시도해 주세요:
    1. VIA 프로그램 다운로드...
    2. 업데이트 실행...
    해결되지 않으면 AS 접수 도와드리겠습니다."
```

---

### 시나리오 5: 재고/발주 관련 내부 메일

**수신 메일 (내부):**
> "EX75 Hot-swap 재고 현황 확인 부탁드립니다. 이번 주 입고 예정 있나요?"

**개인 메일함 AI 처리:**
```
1. 개인 룰 매칭: body contains "재고" → internal_inquiry
2. MCP 조회:
   - erp-system: EX75 Hot-swap 현재 재고 (GT: 45, CJ: 23)
   - erp-system: 발주 현황 (ORDERED: 100대, ETA: 2026-03-18)
   - erp-system: 최근 30일 판매 추이 (일평균 8대)
3. AI 초안 생성:
   "현재 EX75 Hot-swap 재고는 GT 45개 + CJ 23개 = 총 68개입니다.
    발주 100대가 3/18 입고 예정이며,
    현재 일평균 판매 8대 기준 약 8.5일분 재고입니다."
4. → 사용자가 확인/수정 후 발송
```

---

### 시나리오 6: 고객 재문의 (Follow-up)

**수신 메일 (동일 스레드):**
> "아직 택배가 안 왔는데요, 확인 좀 부탁드립니다."

**자동화 처리:**
```
1. 트리거: email.customer_followup (동일 thread_id)
2. 자동 액션:
   - 우선순위 상향 (normal → high)
   - 담당자에게 알림 (이전 담당자 or SLA 담당)
   - AI가 최신 배송 상태 재조회 → 초안 갱신
3. AI 답변:
   "고객님, 확인 결과 택배가 [지역 터미널]에서
    지연되고 있습니다. 택배사에 재배달 요청하였으며..."
```

---

## 3. 데이터 연결 구성

### 3.1 이미 연결 가능한 데이터 (MCP 기반)

| 데이터 소스 | repo_type | 연결 방법 | 현재 상태 |
|-------------|-----------|-----------|-----------|
| TBNWS CRM (MySQL) | database | MySQL MCP 서버 | ✅ 연결 가능 |
| ERP (PostgreSQL) | database | PostgreSQL MCP | ✅ 연결 가능 |
| 사방넷 주문 | database | MySQL MCP | ✅ 연결 가능 |
| 내부 지식문서 | filesystem | 파일시스템 MCP | ✅ 연결 가능 |
| GitHub 문서 | github | GitHub MCP | ✅ 연결 가능 |
| Notion 위키 | notion | Notion MCP | ✅ 연결 가능 |

### 3.2 추가 연결 필요한 데이터

| 데이터 소스 | 연결 방법 | 우선순위 |
|-------------|-----------|----------|
| 택배사 배송조회 API | Custom MCP 서버 개발 | 높음 |
| 네이버 커머스 API | Custom MCP 서버 개발 | 중간 |
| 견적 시스템 | DB 직접 연결 or API | 낮음 |
| 외부 이메일 (협력사) | Gmail API 확장 | 낮음 |

### 3.3 Custom MCP 서버 개발 가이드

택배사 배송조회 MCP 예시:

```python
# mcp_servers/shipping_tracker/server.py
"""택배사 배송 조회 MCP 서버"""

@server.tool("track_shipment")
async def track_shipment(carrier: str, tracking_number: str) -> dict:
    """
    택배 배송 상태 조회

    Args:
        carrier: 택배사 코드 (cj, hanjin, lotte, logen)
        tracking_number: 송장번호

    Returns:
        {"status": "배달중", "location": "강남HUB", "eta": "2026-03-13 18:00"}
    """
    # 택배사 API 호출
    pass

@server.tool("get_shipping_info")
async def get_shipping_info(order_id: str) -> dict:
    """주문번호로 배송 정보 조회 (내부 DB → 택배사 API)"""
    pass
```

**KnowledgeRepository 등록:**
```json
{
  "repo_id": "shipping-tracker",
  "name": "택배 배송 조회",
  "type": "custom_mcp",
  "config": {
    "command": "python",
    "args": ["-m", "mcp_servers.shipping_tracker.server"],
    "cwd": "/app",
    "env": {"CJ_API_KEY": "...", "HANJIN_API_KEY": "..."}
  },
  "description": "택배사별 실시간 배송 상태를 조회합니다. 송장번호 또는 주문번호로 조회 가능.",
  "tags": ["배송", "택배", "추적"]
}
```

---

## 4. 룰 × 데이터 연결 매트릭스

AI가 답변 생성 시 어떤 데이터를 조회할지는 **룰의 `knowledge_repos`** 필드로 결정됨.

| 업무 카테고리 | 필요 데이터 소스 | knowledge_repos 설정 |
|---------------|-----------------|---------------------|
| 배송 문의 | 고객DB + 주문DB + 배송API + 지식문서 | `["tbnws-crm", "sabangnet", "shipping-tracker", "internal-docs"]` |
| 교환/반품 | 고객DB + 주문DB + 재고DB + 지식문서 | `["tbnws-crm", "sabangnet", "erp-system", "internal-docs"]` |
| AS 접수 | 고객DB + AS이력 + 제품가이드 | `["tbnws-crm", "internal-docs"]` |
| 결제 문의 | 고객DB + 주문DB | `["tbnws-crm", "sabangnet"]` |
| 제품 문의 | 제품 스펙 + 재고 | `["erp-system", "internal-docs"]` |
| B2B 견적 | 재고 + 원가 + B2B 정책 | `["erp-system", "internal-docs"]` |
| 재고 문의 (내부) | 재고DB + 발주DB | `["erp-system"]` |
| 발주 요청 (내부) | 재고DB + 판매추이 + 발주이력 | `["erp-system", "sabangnet"]` |

---

## 5. 자동화 수준 로드맵

### Level 1: 분류 + 배정 (현재 가능)
```
메일 수신 → 룰 매칭 → 카테고리 분류 → 에이전트 배정 → 알림
```
- 사람이 답변 작성

### Level 2: AI 초안 + 승인 (현재 가능)
```
메일 수신 → 룰 매칭 → AI 초안 생성 → 승인 큐 → 담당자 확인 → 발송
```
- AI가 초안, 사람이 확인/수정

### Level 3: 데이터 연동 자동 답변 (Phase 1 완료 후)
```
메일 수신 → 룰 매칭 → MCP로 실시간 데이터 조회 → AI 답변 생성 (데이터 포함) → 자동 발송
```
- AI가 실시간 데이터를 포함한 정확한 답변을 자동 발송

### Level 4: 지식 축적 + 자동 개선 (Phase 4 완료 후)
```
답변 발송 → 지식 저장 → 다음 유사 메일 시 더 높은 confidence
→ 반복 패턴 감지 → 새로운 룰 자동 제안 → 자동화 영역 확대
```
- 사용할수록 자동화가 개선되는 선순환

### Level 5: 전방위 자동화 (최종 목표)
```
고객 메일, B2B 메일, 내부 메일, 재고 알림, 발주 알림, 견적 처리
→ 모두 AI 기반 자동/반자동 처리
→ 사람은 예외 케이스와 의사결정만
```

---

## 6. 현재 계획의 갭 분석

### 이미 충분한 것
| 항목 | 문서 | 상태 |
|------|------|------|
| MCP 기반 데이터 연결 인프라 | 코드 완성 | ✅ |
| AI + tool_use 답변 생성 | pipeline.py + ai_processor.py | ✅ |
| 룰 엔진 (조건 → 데이터소스 → 액션) | rule_engine.py | ✅ |
| 고객 정보 조회 (TBNWS) | customer-lookup API | ✅ |
| 자동 파이프라인 | Phase 1 문서 | ✅ |
| 알림 + 자동배정 | Phase 2 문서 | ✅ |
| RBAC | Phase 3 문서 | ✅ |
| 개인/공용 분리 | Phase 4 문서 | ✅ |
| 대시보드/통계 | Phase 5 문서 | ✅ |

### 보충 필요한 것
| 항목 | 필요 작업 | 우선순위 |
|------|-----------|----------|
| 택배사 배송 MCP 서버 | Custom MCP 개발 | 높음 |
| B2B 견적 워크플로우 | 견적서 생성 + 이력 관리 | 중간 |
| 발주 알림 자동화 | 재고 부족 시 자동 알림 메일 | 중간 |
| 내부 메일 자동화 시나리오 | 개인 룰 + 데이터 연동 | Phase 4에 포함 |
| 룰 조건에 외부 데이터 참조 | 주문상태/재고 기반 조건 분기 | 향후 개선 |

---

## 7. 결론

**기술 인프라는 이미 95% 준비됨.** MCP 서버 패턴으로 어떤 데이터든 연결 가능하고, AI가 tool_use로 실시간 데이터를 조회하여 답변을 생성하는 구조가 완성되어 있습니다.

**핵심 남은 작업:**
1. **Phase 1~5 구현** (파이프라인 자동화, 알림, 권한, 개인화, 대시보드)
2. **업무 시나리오별 룰 + 레포지토리 설정** (데이터로 문제가 아니라 설정의 문제)
3. **Custom MCP 서버 개발** (택배사 API 등 외부 서비스 연동)
4. **지식문서 작성** (업무 매뉴얼, 정책, FAQ를 마크다운으로 정리)
