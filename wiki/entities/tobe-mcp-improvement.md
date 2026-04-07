---
title: ToBe MCP 개선 프로젝트
tags: [tobe-mcp, improvement, sql-optimization, architecture]
created: 2026-04-06
updated: 2026-04-06
sources: []
---
# ToBe MCP 개선 프로젝트

## 개요

[[tobe-mcp]] 서버의 보안, 아키텍처, 성능, 코드 품질을 체계적으로 개선하는 프로젝트다. 현재 약 1,400줄의 `mcp_server.py` 단일 파일 구조를 계층화된 아키텍처로 전환한다.

## 현재 문제점 (As-Is)

- 단일 파일(mcp_server.py) 1,400줄에 모든 로직 집중
- erp_stock 테이블 복합 인덱스 부재
- COLLATE 문제로 인한 쿼리 비효율
- CURRENT_STOCK 계산이 O(N x M) 복잡도

## 목표 아키텍처 (To-Be)

```
Entry Points (MCP stdio, SSE, HTTP Proxy, REST)
    ↓
Services (비즈니스 로직)
    ↓
Data Layer (SQL 쿼리, 캐시)
```

## SQL 최적화

- erp_stock 복합 인덱스 4개 이상 추가
- COLLATE 문제 해소
- CURRENT_STOCK O(N x M) 쿼리 최적화

## 실행 계획

4 Phase, 총 18 Task로 구성:

| Phase | 내용 |
|-------|------|
| Phase 0 | 긴급 보안/성능 패치 |
| Phase 1 | 아키텍처 분리 (Entry → Service → Data) |
| Phase 2 | SQL 최적화 및 캐시 레이어 |
| Phase 3 | 모니터링 및 운영 안정화 |

## 스킬 제안

개선 완료 후 추가 예정인 스킬:

- `/morning-brief` — 아침 경영 브리핑
- `/cs-urgency` — CS 긴급 대응
- `/ad-daily` — 일일 광고 성과 분석

## 관련 시스템

- [[tobe-mcp]] - 개선 대상 시스템
- [[tobe-ai]] - 상위 AI 전략
