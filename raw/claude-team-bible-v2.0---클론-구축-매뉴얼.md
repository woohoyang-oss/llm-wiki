# Claude Team Bible -- 새 프로젝트 클론 구축 매뉴얼 v2.0

> auto-email 프로젝트에서 검증된 Claude Team 시스템을 새 프로젝트에 이식하기 위한 완전 가이드.
> 원본 소스: `/Users/yangwooho/auto-email/claude-team/`

---

## 1. 아키텍처 개요

### 1.1 시스템 구조
```
[Slack Workspace]
    |
    | Socket Mode (WebSocket)
    |
[bridge.py x 5 프로세스]     ← 각 에이전트 1개씩
    |
    | subprocess (claude CLI)
    |
[Claude CLI]  →  Claude API (opus / sonnet)
    |
    | HTTP POST
    |
[Dashboard Server :5555]     ← 실시간 모니터링
```

핵심 흐름: Slack 멘션 수신 → bridge.py가 Claude CLI를 subprocess로 호출 → 결과를 Slack 스레드에 응답

### 1.2~1.4 머신 배치, 역할별 모델, 통신 구조

## 2. Slack App 구축
- 5개 App 필요 (역할별)
- Bot Token Scopes, Event Subscriptions, Socket Mode
- 멘션 규칙: 자기 토큰으로 자기 멘션하면 무시됨

## 3. bridge.py 상세
- AGENT_CONFIG, Socket Mode 연결, Claude CLI 호출
- 세션 관리 (thread_ts → session_id)
- 응답 분할, 대시보드 연동

## 4. 에이전트 정의 (.claude/agents/*.md)
- Frontmatter 형식, 각 에이전트 역할 요약

## 5. 환경변수 (.env.claude-team)

## 6. 실행 방법

## 7. 커맨드센터 운영
- 이관 패키지, 아카샤 연동, 태스크 배분

## 8. 대시보드 시스템

## 9. 새 프로젝트 클론 체크리스트 (Phase 1~6)

## 10. 알려진 제약사항
1. 세션 영속화 없음 (인메모리)
2. 동시 처리 제한
3. 에러 복구 없음
4. 로그 제한 (50개)
5. 토큰 노출 위험
