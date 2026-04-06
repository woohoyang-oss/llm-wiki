---
title: Claude 비서 시스템 요약
tags: [claude-assistant, notion, MCP, 업무비서, 매니저]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-assistant-README.md, claude-assistant-manager-guide-v2.md, claude-assistant-notion-references.md, claude-assistant-phase2-mcp-spec.md]
---
# Claude 비서 시스템 요약

## 개요

[[Claude-Assistant]]는 [[ToBe Networks]]의 Claude 기반 업무 비서 시스템이다. 각 매니저의 Claude가 업무를 대화형으로 기록하고, 대표에게 자동 브리핑하는 구조.

**GitHub**: woohoyang-oss/claude-assistant (MIT)

## 핵심 흐름

```
매니저 <-> Claude (대화형 업무 기록)
    |
📊 ToBe Daily Report DB (Notion)
    |
대표 Claude (4관점 브리핑)
```

1. 매니저가 자기 Claude에게 가이드 노션 링크 제공
2. Claude가 기존 업무일지(daily work log)를 읽어 과거 업무 파악
3. 대화하면서 업무 상태를 실시간 기록 → 📊 ToBe Daily Report (새 DB)
4. 하루 3회 체크인으로 진척도 관리
5. 대표 Claude가 새 DB를 크롤링하여 4관점 브리핑 제공

## Notion DB 구조

| DB | 용도 | Data Source ID |
|----|------|----------------|
| daily work log | 기존 메모 (읽기전용) | collection://208c0e21-8a4f-8113-b68c-000b25ac520f |
| 📊 ToBe Daily Report | 새 공식 DB (쓰기전용) | collection://fe6bf7ad-eb38-4e33-b958-429ad423e0a7 |

**DB 스키마**: 제목(Title), 작성자(Select), 유형(Daily/Weekly), 날짜(Date), 상태(Status), 태그(Multi-select), 대표확인(Checkbox), 대표코멘트(Text), 긴급플래그(Checkbox), 연관매니저(Multi-select), 완료건수(Number), 진행건수(Number), 대기건수(Number), 생성일시(Created time), 마지막 업데이트(Last edited time)

**Notion 페이지 구조**:
- 🤖 Claude 비서 시스템 (별도 공간) → 매니저 가이드 + 대표 브리핑 가이드 + 📊 ToBe Daily Report DB
- 운영팀 업무일지 (기존 공간, 건드리지 않음) → daily work log DB

## 매니저 가이드 핵심 (v2)

### 초기 세팅
- Claude Desktop Cowork 모드에서 가이드 노션 링크 제공
- 스케줄러 3개 태스크 등록 (오전 9시, 오후 14시, 퇴근 18시) — Cowork에서만 가능, 1회만

### 하루 3회 체크인

| 시간대 | 활동 |
|--------|------|
| 🌅 출근~11시 | 어제 미완료 팔로업, 어제 카드 자동 완료, 미완료 항목 오늘자로 이월, 📊 DB에 페이지 생성 |
| ☀️ 13~16시 | 중간 점검, 블로커 감지 (같은 항목 연속 막힘 시 패턴 기록), 연관 매니저 표시 |
| 🌙 17~19시 | 하루 마무리, 완료 체크, 최종 기록, 상태: 완료로 변경 |

### 실시간 대화형 업데이트
- "끝났어" → [x] + 완료건수 +1
- "새 일 들어왔어" → [ ] 추가 + 진행건수 +1
- "막혔어" → 대기 상태 + 대기건수 +1
- "긴급건/ASAP" → ASAP 섹션 추가
- "OO님이랑 같이" → 연관매니저 속성 추가

### 운영 규칙
- 모든 시간 KST (UTC+9), 하루에 한 페이지, 수정 가능/삭제 금지
- 기존 daily work log에는 절대 쓰지 않음
- 공지사항: 가이드 페이지 내 📢 섹션에 대표가 공지 → 대화 시작 시 자동 체크

### 대표 브리핑 4관점
📊 전체요약 / 🚨 미완료지연 / 🤝 매니저간 연관 / 📅 마감임박

## Phase 2 MCP 설계

Phase 1(현재)은 Notion 직접 연동. Phase 2는 [[ToBe Report MCP]]를 통해 PostgreSQL 동기화 + 전용 엔드포인트 추가.

### 엔드포인트

| API | 기능 |
|-----|------|
| GET /reports | 매니저 리포트 조회 (date, author, type, status 필터) |
| GET /reports/{id}/content | 리포트 본문 (sections: sales_check, today_tasks, asap 등) |
| GET /notices | 공지사항 조회 |
| GET /blockers | 블로커/대기 항목 (days, author 필터, severity 포함) |
| GET /briefing | 대표 브리핑 4관점 통합 (summary, urgent, blockers, cross_team, deadlines) |
| POST /reports/{id}/review | 대표확인 + 코멘트 |

### 동기화
- Notion → PostgreSQL: 5분 간격 폴링, last_edited_time 기준 변경분만. 하루 1회 전체 정합성 체크
- PostgreSQL → Notion: 대표확인/대표코멘트 등 대표 액션만 역동기화

### 토큰 효율
리포트 1건 읽기: 3,000 → 400 토큰 (87% 감소). 전체 브리핑 4명: 15,000 → 2,000 토큰.

### 기술 스택
Python FastAPI + PostgreSQL + Notion API + Docker + Tailscale

## 등록 매니저 (18명)

문소현, 김윤주, 김성은, 강은비, 임진희, 김지은, 김보경, 김경효, 진우, 우호, 강성묵, 왕민수, 조현아, 이승규, 전은영, 정민경, 고은샘, 이혜진

## 로드맵

| Phase | 상태 | 내용 |
|-------|------|------|
| Phase 1 | 완료 | Notion 직접 연동 |
| Phase 2 | 설계 완료 | ToBe Report MCP (Notion → PostgreSQL 동기화) |
| Phase 3 | 계획 | 고도화 (자동 연관 감지, 매출 크로스 분석, 블로커 대시보드) |
