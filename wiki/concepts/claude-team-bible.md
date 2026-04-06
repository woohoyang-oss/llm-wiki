---
title: Claude Team 구축 바이블
tags: [claude-team, bible, slack, agent, command-center]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-team-bible-v2.0---클론-구축-매뉴얼.md, claude-team-bible---구축-매뉴얼-v1.0.md]
---
# Claude Team 구축 바이블

## 개요

[[claude-team]] 시스템을 새 프로젝트에 복제하기 위한 종합 매뉴얼이다. [[auto-email]] 프로젝트에서 검증된 팀형 Claude 에이전트 아키텍처를 표준화한 문서로, v1.0에서 v2.0으로 발전하였다.

## 팀 구성 (6+1 구조)

| 역할 | 모델 | 배치 머신 |
|------|------|----------|
| Team Lead (TL) | Sonnet | 맥북 |
| Project Leader (PL) | Opus | 맥북 |
| Coder | Sonnet | 맥미니 |
| Reviewer | Sonnet | 맥미니 |
| Tester (Slack) | Sonnet | 맥미니 |
| Tester (아카샤) | Sonnet | 맥스튜디오 |
| 커맨드센터 | - | 맥스튜디오 |

## 시스템 구조

Slack 워크스페이스를 실시간 통신 채널로 사용하고, [[bridge-architecture|bridge.py]]가 각 에이전트와 Claude CLI를 연결한다. 아카샤는 비동기 태스크 기록과 영구 저장소 역할을 담당한다.

## Slack App 구축

에이전트별로 독립 Slack App을 생성한다(총 5개). 각 App에 Bot Token Scopes(`chat:write`, `channels:history`, `app_mentions:read`)를 부여하고, Socket Mode를 활성화하여 WebSocket 연결을 확보한다. 자기 토큰으로 자기 멘션 시 무시되는 규칙에 주의해야 한다.

## 에이전트 정의

`.claude/agents/*.md` 파일에 각 에이전트의 역할과 시스템 프롬프트를 정의한다. Frontmatter 형식으로 메타데이터를 포함하며, bridge.py가 실행 시 해당 파일을 로드하여 `--append-system-prompt`로 전달한다.

## 커맨드센터 운영

커맨드센터는 팀 전체를 조율하는 중추 역할이다. 이관 패키지를 통해 세션 간 컨텍스트를 전달하고, 아카샤 연동으로 태스크를 배분한다. 메시지 전파는 하향(지시), 상향(보고), 수평(연계) 3방향으로 이루어진다.

## 새 프로젝트 클론 체크리스트

1. **Phase 1**: Slack 워크스페이스 + App 생성
2. **Phase 2**: bridge.py 복사 + 환경변수 설정
3. **Phase 3**: 에이전트 정의 파일 작성
4. **Phase 4**: 대시보드 구축 + 커맨드센터 세팅

## 알려진 제약사항

세션 영속화 없음, 동시 처리 제한, 에러 복구 미비, 대시보드 로그 50개 제한, 환경변수 토큰 노출 위험 등이 있다. [[ci-cd-pipeline|배포 파이프라인]]과 연계하여 안정성을 보완한다.

## 관련 소스
- [claude-team-bible-v2.0---클론-구축-매뉴얼.md](../../raw/claude-team-bible-v2.0---클론-구축-매뉴얼.md)
- [claude-team-bible---구축-매뉴얼-v1.0.md](../../raw/claude-team-bible---구축-매뉴얼-v1.0.md)
