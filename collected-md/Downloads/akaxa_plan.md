# AKAXA.SPACE 프로젝트 계획서

## 1. 프로젝트 개요

AI Agent들을 위한 외부 메모리 레이어와 인간이 이를 관찰할 수 있는 시각화 환경을 결합한 플랫폼

> "동전의 양면 구조"

- 앞면: AI Agent 사용 영역 (Memory / API / MCP)
- 뒷면: Human 관찰 영역 (Trace 기반 시각화)

---

## 2. 핵심 컨셉

### 2.1 AI Agent Memory Layer

AI Agent의 본질적 한계인 "stateless" 문제를 해결

- 장기 메모리 저장
- 과거 맥락 유지
- Agent 간 간접적 지식 공유

### 2.2 Human Observatory (수족관 뷰)

AI 활동을 직접 내용이 아닌 "흔적"으로 관찰

- Agent 활동 시각화
- 메모리 저장/조회 이벤트
- 주제 흐름 관찰

> 핵심 원칙: **내용은 숨기고, 행동만 보여준다**

---

## 3. 시스템 구조

```
[ AI Agent Layer ]
  - MCP Server
  - REST API
  - Memory / Thought / Fact API

[ Core System ]
  - Memory Store
  - Event Processor
  - Public Trace Generator

[ Human Layer ]
  - Visualization Dashboard
  - Activity Feed
  - Topic Flow
```

---

## 4. 데이터 설계

### 4.1 핵심 테이블

#### agents
- id
- name
- type
- created_at
- last_active

#### memories
- id
- agent_id
- key
- value (비공개)
- scope (private / shared / public)
- created_at

#### events (내부)
- id
- agent_id
- action
- payload (비공개)
- created_at

#### public_events (공개용)
- id
- agent_label
- action
- topic_hint
- timestamp

---

## 5. 이벤트 구조 (핵심)

### 5.1 내부 이벤트

- full payload 포함
- 디버깅 및 실제 데이터 처리용

### 5.2 공개 이벤트 (Trace)

- 데이터 축약
- 내용 제거
- 익명화 또는 최소 정보만 유지

예시:

```
Agent A → memory_save
Agent B → recall topic: logistics
Agent C → fact_check 수행
```

---

## 6. 인증 및 계정 구조

### 6.1 Agent 중심 구조

- agent_id
- api_key

> 사람 계정보다 Agent identity가 우선

### 6.2 Human 계정 (후순위)

- 대시보드 접근
- Agent 관리
- 분석용

---

## 7. API 설계

### Memory
- POST /memory/save
- GET /memory/recall
- GET /memory/list

### Thought
- POST /thought
- GET /thought

### Fact
- GET /fact/verify

---

## 8. MVP 범위

### 포함
- memory_save / recall
- Agent 인증 (API Key)
- 이벤트 로그
- 기본 Trace 시각화

### 제외
- 집단지성
- 고급 검색
- 평판 시스템

---

## 9. 개발 단계

### Phase 1
- 로컬/단일 서버 메모리
- API 구현
- MCP 연결

### Phase 2
- 이벤트 스트림
- public trace 생성
- 간단 시각화

### Phase 3
- 멀티 Agent 환경
- 확장형 DB
- 고급 기능

---

## 10. 핵심 성공 조건

1. Agent가 반복적으로 사용할 것
2. 메모리 저장/조회가 실제 성능 개선으로 이어질 것
3. 시각화가 흥미를 유발할 것

---

## 11. 핵심 철학

> "우리는 데이터를 보여주지 않는다. 흐름을 보여준다."

> "AI는 기억하고, 인간은 관찰한다."
