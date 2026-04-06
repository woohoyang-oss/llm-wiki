---
name: GSD-2 Methodology
description: GSD-2 (Get Shit Done) autonomous agent development methodology - work hierarchy, execution pipeline, auto mode, configuration
type: reference
---

## GSD-2 (Get Shit Done) - Autonomous Agent Development System
**Repo**: https://github.com/gsd-build/gsd-2
**Docs**: gsd.build

### Work Hierarchy (핵심 구조)
- **Milestone**: 배포 가능한 버전 (4-10 slices 포함)
- **Slice**: 시연 가능한 수직적 기능 단위 (1-7 tasks 포함)
- **Task**: 하나의 컨텍스트 윈도우에 맞는 작업 단위 (Iron Rule: 각 task는 반드시 1개 컨텍스트 윈도우 내에 수행 가능해야 함)

### Execution Pipeline (5단계)
1. **Plan** — 코드베이스 탐색, 문서 조사, task로 분해 (검증 가능한 결과물 정의)
2. **Execute** — 격리된 컨텍스트에서 각 task 실행 (사전 로드된 관련 파일 포함)
3. **Complete** — 요약 작성, 로드맵 갱신, 커밋 메시지 자동 생성
4. **Reassess** — 새 정보 기반으로 로드맵 유효성 재평가
5. **Validate** — 로드맵 성공 기준 vs 실제 결과 비교하는 최종 게이트

### 5대 원칙
1. **Extension-First**: 코어는 린하게, 가능하면 확장으로
2. **Simplicity Over Abstraction**: 불필요한 추상화보다 코드 중복이 낫다
3. **Tests Define Contracts**: 테스트가 행동을 문서화
4. **Rapid Iteration**: 완벽보다 빠른 출시 + 반복
5. **Provider Neutrality**: 특정 LLM 벤더에 종속 안 함

### Auto Mode 핵심 기법
- **Fresh Context Per Unit**: 각 task마다 깨끗한 컨텍스트 윈도우 (오염 방지)
- **Pre-loaded Context**: 디스패치 프롬프트에 task plan, 의존성 요약, 이전 결과 인라인 포함
- **Git Isolation**: worktree(기본)/branch/none 모드
- **Crash Recovery**: 락 파일 + 세션 합성으로 전체 상태 복구
- **Stuck Detection**: 슬라이딩 윈도우 패턴 분석 (A→B→A 순환 감지)
- **Context Pressure Monitor**: 70% 사용 시 wrap-up 시그널
- **KNOWLEDGE.md**: 프로젝트별 패턴/교훈의 append-only 레지스터 (컨텍스트 경계 생존)

### Configuration
- **Global**: `~/.gsd/PREFERENCES.md` (YAML frontmatter)
- **Project**: `.gsd/PREFERENCES.md`
- **token_profile**: budget/balanced/quality
- **verification_commands**: 매 task 후 자동 실행 (lint, test)
- **budget_ceiling**: 자동 모드 최대 지출
- **phases**: skip_research, skip_reassess 등 세밀한 제어

### 상태 관리
- `.gsd/STATE.md`가 유일한 진실의 원천 (disk-driven state machine)
- 디스크 상태가 결정을 주도; 도구 호출은 진행 전 영속화
- 모든 결정에 비용, 토큰, 이상 플래그 로깅

### How to apply
프로젝트 작업 시 GSD-2의 계층 구조(Milestone→Slice→Task)와 실행 파이프라인을 참고하여 작업을 분해하고, 각 단위를 격리된 컨텍스트로 실행하는 패턴을 적용할 수 있음.
