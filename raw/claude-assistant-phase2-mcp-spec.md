# ToBe Report MCP — Phase 2 설계

## 개요
Notion DB를 PostgreSQL에 동기화하고, ToBe MCP에 리포트 엔드포인트를 추가하여 Claude가 가볍고 빠르게 리포트 데이터에 접근하는 구조.

## 엔드포인트

### GET /reports
매니저 리포트 조회. params: date, date_range, author, type, status
Response: { reports: [{ id, title, author, type, date, status, tags, counts: {완료,진행,대기}, urgent, reviewed, related_managers, created_at, updated_at }] }

### GET /reports/{id}/content
특정 리포트 본문. sections: { sales_check, today_tasks[{task,status,category}], asap, this_week, daily_routine, notes }

### GET /notices
공지사항 조회. Response: { notices: [{ id, content, created_at, author }], last_checked }

### GET /blockers
블로커/대기 항목. params: days(default:7), author
Response: { blockers: [{ task, author, first_seen, days_blocked, severity, related_managers }] }

### GET /briefing
대표 브리핑 (4관점 통합). params: date, scope(today|this_week|last_week)
Response: { summary: {total_reports, authors, counts, top_achievements}, urgent, blockers, cross_team: [{managers, keyword, description}], deadlines: [{date, author, task, d_day}] }

### POST /reports/{id}/review
대표확인 처리. Body: { reviewed: true, comment: "확인. 발주 승인." }

## 동기화 설계

### Notion → PostgreSQL
방식: Notion API 폴링 (5분 간격), last_edited_time 기준 변경분만 동기화
전체 스캔: 하루 1회 정합성 체크
테이블: reports, report_tasks, notices, sync_log

### PostgreSQL → Notion
방식: 변경 감지 후 Notion API 호출
대상: 대표확인, 대표코멘트 등 대표 액션만 역동기화

## 기술 스택
Python FastAPI, PostgreSQL, Notion API, Docker + Tailscale (기존 투비 인프라)

## 토큰 효율 비교
리포트 1건 읽기: 3,000 → 400 토큰
전체 브리핑 (4명): 15,000 → 2,000 토큰
공지사항 체크: 5,000 → 200 토큰
블로커 조회: 12,000 → 500 토큰