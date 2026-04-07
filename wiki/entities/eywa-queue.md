---
title: Eywa Queue - 태스크 오케스트레이션 시스템
tags: [eywa, task-queue, orchestration, sqlite]
created: 2026-04-06
updated: 2026-04-06
sources: []
---
# Eywa Queue - 태스크 오케스트레이션 시스템

## 개요

Eywa Queue는 [[eywa]]에서 분리된 독립 태스크 오케스트레이션 시스템이다. DAG 기반 의존성 관리와 리소스 락, 충돌 감지를 지원하며, 다양한 Worker Adapter를 통해 태스크를 실행한다.

## 데이터 모델

SQLite DB에 다음 5개 테이블로 구성된다:

| 테이블 | 역할 |
|--------|------|
| tasks | 태스크 정의 및 상태 |
| pipeline_stages | 파이프라인 단계 |
| dependencies | DAG 의존성 관계 |
| workers | 워커 등록 및 상태 |
| resource_locks | 리소스 락 관리 |

## Worker Adapters

- **Subprocess**: Claude CLI를 subprocess로 실행
- **HTTP Callback**: 외부 서비스에 HTTP 요청으로 태스크 위임
- **Stdio**: 표준 입출력 기반 통신

## Task Lifecycle

```
queued → dispatched → in_progress → done / failed → retry
```

실패 시 자동 재시도하며, DAG 의존성에 따라 선행 태스크 완료 후 후속 태스크가 디스패치된다. 리소스 락으로 동시 접근 충돌을 방지한다.

## 서버

Port 3001에서 WebSocket + HTTP API를 제공한다. 실시간 태스크 상태 모니터링과 제어가 가능하다.

## 관련 시스템

- [[eywa]] - 원본 오케스트레이션 플랫폼
- [[claude-team]] - 태스크 실행 주체
