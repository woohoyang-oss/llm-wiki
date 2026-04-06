# Claude-Assistant

ToBe Networks의 Claude 기반 업무 비서 시스템.
각 매니저의 Claude가 업무를 대화형으로 기록하고, 대표에게 자동 브리핑하는 구조.

## 개요

```
매니저 ↔ Claude (대화형 업무 기록)
    ↓
📊 ToBe Daily Report DB (Notion)
    ↓
대표 Claude (4관점 브리핑)
```

### 핵심 흐름

1. 매니저가 자기 Claude에게 가이드 노션 링크를 줌
2. Claude가 기존 업무일지(daily work log)를 읽어 과거 업무 파악
3. 대화하면서 업무 상태를 실시간 기록 → 📊 ToBe Daily Report (새 DB)
4. 하루 3회 체크인 (오전/오후/퇴근)으로 진척도 관리
5. 대표 Claude가 새 DB를 크롤링하여 4관점 브리핑 제공

## 아키텍처

### 현재 (Phase 1 — Notion 직접 연동)

```
┌─────────────────────────────────────────────────┐
│  매니저 Claude (각자의 Claude 계정)               │
│  ├─ 읽기: daily work log (기존 DB, 읽기전용)     │
│  ├─ 쓰기: 📊 ToBe Daily Report (새 DB)          │
│  └─ 체크: 공지사항 (가이드 페이지 내)             │
├─────────────────────────────────────────────────┤
│  📊 ToBe Daily Report DB                        │
│  ├─ 속성: 작성자/유형/날짜/상태/태그              │
│  ├─ 속성: 대표확인/긴급플래그/연관매니저           │
│  ├─ 속성: 완료건수/진행건수/대기건수              │
│  ├─ 속성: 생성일시/마지막 업데이트                │
│  └─ 양방향: 노션 UI + Notion MCP                │
├─────────────────────────────────────────────────┤
│  대표 Claude                                     │
│  ├─ 크롤링: 📊 ToBe Daily Report                │
│  └─ 브리핑: 전체요약/미완료/연관업무/마감임박       │
└─────────────────────────────────────────────────┘
```

### 목표 (Phase 2 — MCP 연동)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 매니저 Claude │     │ 대표 Claude   │     │ Notion (UI)  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────┬───────────┘                    │
                │                                │
       ┌────────▼────────┐              ┌────────▼────────┐
       │ ToBe Report MCP │              │  양방향 동기화    │
       │ (tbe.kr/mcp)    │◄────────────►│  Notion ↔ DB    │
       └────────┬────────┘              └─────────────────┘
                │
       ┌────────▼────────┐
       │   PostgreSQL     │
       │   (백엔드 DB)    │
       └─────────────────┘
```

Phase 2 장점: 토큰 10분의1, 크로스 매니저 쿼리, 투비 MCP 통합, 노션 프론트 유지

## Notion 구조

```
🤖 Claude 비서 시스템 (별도 공간)
├── 🤖 매니저 가이드
├── 🏢 대표 브리핑 가이드
└── 📊 ToBe Daily Report DB

운영팀 업무일지 (기존 공간, 건드리지 않음)
└── daily work log DB (읽기전용 참조)
```

## 주요 기능

### 하루 3회 체크인
- 🌅 출근~11시: 어제 미완료 팔로업 + 오늘 세팅 → 페이지 생성
- ☀️ 13~16시: 중간 점검, 블로커 파악, 상태 업데이트
- 🌙 17~19시: 하루 마무리, 완료 체크, 최종 기록

### 실시간 대화형 업데이트
"이거 끝났어" → 즉시 [x] + 완료건수+1
"새 일 들어왔어" → 즉시 [ ] 추가
"막혔어" → 대기 상태 변경
"긴급건" → ASAP 추가

### 공지사항 폴링
가이드 페이지 내 📢 섹션에 대표가 공지 → 대화 시작 시 Claude가 자동 체크

### 대표 브리핑 4관점
📊 전체요약 / 🚨 미완료지연 / 🤝 매니저간 연관 / 📅 마감임박

## DB 스키마

제목(Title), 작성자(Select), 유형(Select: Daily/Weekly), 날짜(Date), 상태(Status), 태그(Multi-select), 대표확인(Checkbox), 대표코멘트(Text), 긴급플래그(Checkbox), 연관매니저(Multi-select), 완료건수(Number), 진행건수(Number), 대기건수(Number), 생성일시(Created time), 마지막 업데이트(Last edited time)

## 운영 규칙

- 모든 시간: KST (UTC+9)
- 하루에 한 페이지
- 기존 페이지 업데이트 (새로 만들지 않음)
- 수정 가능, 삭제 금지
- 기존 DB(daily work log) 읽기전용

## 로드맵

Phase 1 ✅ — Notion 직접 연동 (완료)
Phase 2 — ToBe Report MCP (Notion→PostgreSQL 동기화, 엔드포인트 추가)
Phase 3 — 고도화 (자동 연관 감지, 매출 크로스 분석, 블로커 대시보드)

## Notion 링크

- 매니저 가이드: https://www.notion.so/333c0e218a4f81a582caf81b15b99642
- 대표 브리핑: https://www.notion.so/333c0e218a4f812b94d2e270da52cb86
- ToBe Daily Report DB: https://www.notion.so/45f30380ec524232b2d32c3c3e90a5fa
- daily work log (읽기전용): https://www.notion.so/208c0e218a4f81c1b3ffe883552c872b

## 라이선스
MIT — ToBe Networks Inc.
레포: github.com/woohoyang-oss/claude-assistant