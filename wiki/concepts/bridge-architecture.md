---
title: Slack-Claude 브릿지 아키텍처
tags: [bridge, slack, claude-cli, socket-mode, subprocess]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-team-bible---bridge.py-상세-가이드.md, tester---bridge.py-소스코드.md, claude-team---맥미니-브릿지-운영-가이드.md]
---
# Slack-Claude 브릿지 아키텍처

## 개요

[[claude-team]]의 핵심 인프라인 bridge.py는 Slack 워크스페이스와 Claude CLI를 연결하는 브릿지 프로그램이다. 각 에이전트(team-lead, coder, reviewer, tester, project-leader)마다 독립 프로세스로 실행되며, Slack 멘션을 수신하면 Claude CLI를 subprocess로 호출하고 결과를 스레드에 응답한다.

## 시스템 구조

```
[Slack Workspace] → Socket Mode (WebSocket) → [bridge.py x 5] → subprocess → [Claude CLI] → Claude API
                                                    ↓
                                            [Dashboard :5555]
```

## 핵심 메커니즘

### Socket Mode 연결
Slack Bolt의 `SocketModeHandler`를 사용하여 WebSocket 기반 실시간 이벤트를 수신한다. 각 에이전트별로 별도의 Bot Token과 App-Level Token이 필요하며, `app_mentions:read` 이벤트를 구독한다.

### Claude CLI subprocess 호출
`subprocess.run()`으로 Claude CLI를 호출한다. `--output-format json`으로 구조화된 응답을 받고, `--dangerously-skip-permissions` 플래그로 권한 확인을 생략한다. 에이전트별 시스템 프롬프트는 `.claude/agents/*.md` 파일에서 로드한다.

### 세션 관리
스레드 단위로 세션을 유지한다. `thread_ts`를 키로 `session_id`를 매핑하여 대화 맥락이 이어지도록 하며, 새 세션 생성 후 `--resume` 플래그로 기존 세션을 이어간다. 세션은 인메모리로만 관리되어 프로세스 재시작 시 소실된다.

### 응답 처리
Slack 메시지 길이 제한(4000자)에 대응하여 3900자 초과 시 자동 분할 전송한다. 대시보드 서버(:5555)에 에이전트 상태와 작업 로그를 HTTP POST로 전송한다.

## 타임아웃 이슈

초기 타임아웃은 300초(5분)로 설정되었으나, CI 분석+수정+푸시 같은 복합 작업에서 초과 문제가 발생하여 600초(10분)로 상향 조정하였다. 근본 대책으로 PL이 지시할 때 **한 지시당 1가지 작업, 5분 내 완료 가능 크기로 분할**하는 규칙을 수립했다.

## 알려진 제약사항

- 세션 영속화 없음 (인메모리 딕셔너리)
- 동시 처리 제한 (동일 스레드 내 순차 실행)
- 에러 복구 메커니즘 없음
- 토큰 노출 위험 (환경변수 관리 필수)

## 관련 소스
- [claude-team-bible---bridge.py-상세-가이드.md](../../raw/claude-team-bible---bridge.py-상세-가이드.md)
- [tester---bridge.py-소스코드.md](../../raw/tester---bridge.py-소스코드.md)
- [claude-team---맥미니-브릿지-운영-가이드.md](../../raw/claude-team---맥미니-브릿지-운영-가이드.md)
