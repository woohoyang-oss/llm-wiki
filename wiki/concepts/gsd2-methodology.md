---
title: GSD-2 자율 에이전트 방법론
tags: [gsd-2, methodology, agent, pipeline, parallel]
created: 2026-04-06
updated: 2026-04-06
sources: [auto-dm-gsd2-methodology.md, 병렬-파이프라인-프로세스-v1-2026-03-26.md]
---
# GSD-2 자율 에이전트 방법론

## 개요

GSD-2(Get Shit Done)는 자율 에이전트 개발을 위한 체계적 방법론이다. 작업 계층 구조, 실행 파이프라인, 자동 모드를 정의하며, [[claude-team]]의 병렬 파이프라인 프로세스에도 이 원칙이 적용된다.

## 작업 계층 구조 (Work Hierarchy)

- **Milestone**: 배포 가능한 버전 단위 (4~10개 Slice 포함)
- **Slice**: 시연 가능한 수직적 기능 단위 (1~7개 Task 포함)
- **Task**: 하나의 컨텍스트 윈도우에 맞는 작업 단위

**Iron Rule**: 각 Task는 반드시 1개 컨텍스트 윈도우 내에서 수행 가능해야 한다.

## 실행 파이프라인 (5단계)

1. **Plan** -- 코드베이스 탐색, 문서 조사, Task로 분해
2. **Execute** -- 격리된 컨텍스트에서 각 Task 실행
3. **Complete** -- 요약 작성, 로드맵 갱신, 커밋 메시지 생성
4. **Reassess** -- 새 정보 기반 로드맵 유효성 재평가
5. **Validate** -- 성공 기준 vs 실제 결과 비교 (최종 게이트)

## 5대 원칙

1. **Extension-First**: 코어는 린하게, 가능하면 확장으로 구현
2. **Simplicity Over Abstraction**: 불필요한 추상화보다 코드 중복이 낫다
3. **Tests Define Contracts**: 테스트가 행동을 문서화
4. **Rapid Iteration**: 완벽보다 빠른 출시 + 반복
5. **Provider Neutrality**: 특정 LLM 벤더 종속 금지

## Auto Mode 핵심 기법

- **Fresh Context Per Unit**: 각 Task마다 깨끗한 컨텍스트 (오염 방지)
- **Pre-loaded Context**: 디스패치 프롬프트에 의존성 요약, 이전 결과 인라인 포함
- **Git Isolation**: worktree/branch/none 모드 선택
- **Stuck Detection**: 슬라이딩 윈도우로 A->B->A 순환 감지
- **Context Pressure Monitor**: 70% 사용 시 wrap-up 시그널
- **KNOWLEDGE.md**: 프로젝트별 패턴/교훈의 append-only 레지스터

## 병렬 파이프라인 프로세스

[[claude-team]]에서 GSD-2 원칙을 적용한 5-Phase 병렬 프로세스이다. **순차 진행 금지**가 핵심 원칙이다.

| Phase | 동시 실행 Agent | 산출물 |
|-------|---------------|--------|
| 1. 병렬 사전 분석 | Explore + Reviewer + Tester | 분석/리뷰/테스트 명세 |
| 2. 구현 명세 확정 | Plan | 통합 명세서 |
| 3. 구현 + 테스트 | Coder + Tester | 코드 + 테스트 코드 |
| 4. 검증 | Reviewer + Tester | 리뷰 + 테스트 결과 |
| 5. 수정 및 완료 | Coder | 최종 머지 |

Phase 1은 3개 Agent 동시, Phase 3은 Coder+Tester 동시, Phase 4는 Reviewer+Tester 동시 실행이 필수이다. 단순 수정(오타 등)은 Phase 1만 스킵 가능하나 Phase 3~4는 항상 수행한다.

## 관련 소스
- [auto-dm-gsd2-methodology.md](../../raw/auto-dm-gsd2-methodology.md)
- [병렬-파이프라인-프로세스-v1-2026-03-26.md](../../raw/병렬-파이프라인-프로세스-v1-2026-03-26.md)
