---
title: Claude Assistant - 매니저 업무비서 시스템
tags: [claude-assistant, 업무비서, Notion, 체크인, MCP]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-assistant-README.md, claude-assistant-manager-guide-v2.md, claude-assistant-notion-references.md, claude-assistant-phase2-mcp-spec.md]
---
# Claude Assistant - 매니저 업무비서 시스템

## 개요

Claude Assistant는 ToBe Networks의 Claude 기반 업무 비서 시스템이다. 각 매니저의 Claude가 업무를 대화형으로 기록하고, 대표에게 자동 브리핑하는 구조로 운영된다. 현재 18명의 매니저가 등록되어 사용 중이며, Phase 1(Notion 직접 연동)이 완료된 상태이다.

GitHub: github.com/woohoyang-oss/claude-assistant (MIT)

## 핵심 흐름

1. 매니저가 자기 Claude에게 가이드 Notion 링크를 제공
2. Claude가 기존 업무일지(daily work log)를 읽어 과거 업무 파악
3. 대화하면서 업무 상태를 실시간 기록 -> ToBe Daily Report DB(Notion)에 저장
4. 하루 3회 체크인으로 진척도 관리
5. 대표 Claude가 ToBe Daily Report를 크롤링하여 4관점 브리핑 제공

## 하루 3회 체크인

| 시간대 | 체크인 | 내용 |
|--------|--------|------|
| 출근~11시 | 오전 | 어제 미완료 팔로업 + 오늘 세팅, 페이지 생성 |
| 13~16시 | 오후 | 중간 점검, 블로커 파악, 상태 업데이트 |
| 17~19시 | 퇴근 | 하루 마무리, 완료 체크, 최종 기록 |

Cowork 모드에서 스케줄러를 1회 등록하면 매일 자동으로 체크인이 실행된다. 체크인 외에도 대화 중 즉시 업데이트가 가능하다("끝났어" -> 완료 처리, "새 일 들어왔어" -> 항목 추가).

## Notion DB 구조

### ToBe Daily Report DB (쓰기 전용)
- 속성: 제목, 작성자(Select), 유형(Daily/Weekly), 날짜, 상태, 태그(Multi-select)
- 수치: 완료건수, 진행건수, 대기건수
- 관리: 대표확인(Checkbox), 대표코멘트, 긴급플래그, 연관매니저
- 규칙: 하루에 한 페이지, 기존 페이지 업데이트(새로 만들지 않음), 수정 가능/삭제 금지

### daily work log (읽기 전용)
기존 운영팀 업무일지 DB. Claude가 과거 업무 파악용으로만 참조하며 절대 쓰지 않는다.

## 대표 브리핑 4관점

1. **전체 요약**: 총 리포트 수, 작성자, 완료/진행/대기 건수, 주요 성과
2. **미완료/지연**: 블로커, 대기 항목, 심각도
3. **매니저 간 연관**: 크로스팀 업무, 키워드 기반 연결
4. **마감 임박**: D-day 기준 긴급 건

## Phase 2 계획: ToBe Report MCP

Phase 2에서는 Notion DB를 PostgreSQL에 동기화하고, ToBe MCP에 리포트 엔드포인트를 추가하여 토큰 효율을 대폭 개선할 계획이다.

- **엔드포인트**: GET /reports, GET /reports/{id}/content, GET /notices, GET /blockers, GET /briefing, POST /reports/{id}/review
- **동기화**: Notion -> PostgreSQL 5분 폴링 + 하루 1회 전체 스캔
- **토큰 절감 효과**: 리포트 1건 읽기 3,000 -> 400 토큰, 전체 브리핑 15,000 -> 2,000 토큰

## 로드맵

- Phase 1 (완료): Notion 직접 연동
- Phase 2 (계획): ToBe Report MCP (PostgreSQL 동기화)
- Phase 3 (계획): 자동 연관 감지, 매출 크로스 분석, 블로커 대시보드

## 관련 소스

- [claude-assistant-README.md](../../raw/claude-assistant-README.md)
- [claude-assistant-manager-guide-v2.md](../../raw/claude-assistant-manager-guide-v2.md)
- [claude-assistant-notion-references.md](../../raw/claude-assistant-notion-references.md)
- [claude-assistant-phase2-mcp-spec.md](../../raw/claude-assistant-phase2-mcp-spec.md)
