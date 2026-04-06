---
title: Claude Team - 멀티머신 AI 에이전트 팀
tags: [claude-team, AI에이전트, Slack, bridge.py, 멀티머신]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-team-bible-v2.0---클론-구축-매뉴얼.md, claude-team-bible---구축-매뉴얼-v1.0.md, claude-team---머신-배치도.md, claude-team---팀-운영-현황-2026-03-23.md, claude-team-README-섹션-auto-email에서-분리.md]
---
# Claude Team - 멀티머신 AI 에이전트 팀

## 개요

Claude Team은 여러 물리 머신에 분산된 Claude CLI 인스턴스를 Slack 기반으로 협업시키는 AI 에이전트 팀 시스템이다. 6+1 구조(TL, PL, Coder, Reviewer, Tester + 커맨드센터)로 구성되어 [[auto-email]] 프로젝트 개발에 투입되었다. 이 시스템의 운영 경험을 바탕으로 범용 오케스트레이션 플랫폼 [[eywa]]가 설계되었다.

## 아키텍처: Slack -> bridge.py -> Claude CLI

핵심 통신 흐름은 다음과 같다:

1. **Slack 멘션 수신**: Socket Mode(WebSocket)로 실시간 연결
2. **bridge.py 중계**: 각 에이전트별 독립 프로세스로 구동. Slack 멘션을 받아 Claude CLI를 subprocess로 호출
3. **Claude CLI 실행**: 에이전트 MD 파일(.claude/agents/*.md)을 시스템 프롬프트로 로드하여 실행
4. **결과 응답**: Claude 출력을 Slack 스레드에 분할 전송 (3,900자 제한)

bridge.py는 세션 관리(thread_ts 기반), 자동 세션 갱신(호출 20회/30분/응답 느림 감지), 대시보드 연동(localhost:5555) 기능을 포함한다. 인증은 OAuth(Keychain), API Key, OpenAI Compatible 3가지를 지원한다.

## 머신 배치

| 머신 | 역할 | 모델 |
|------|------|------|
| MacBook Pro | Team Lead(전략 어드바이저), Project Leader(프로젝트 관리) | Sonnet, Opus |
| Mac Mini | Coder(구현), Reviewer(코드 리뷰), Tester(테스트) | Sonnet |
| Mac Studio | 커맨드센터 (총괄 지휘, 브릿지 안 돌림) | Opus |

각 역할별로 별도의 Slack Bot App이 필요하며, 자기 토큰으로 자기에게 멘션하면 무시되는 규칙이 존재한다.

## 역할 및 운영 파이프라인

- **PL**: 과제 발굴 -> 팀원 지시 -> 결과 확인 -> 커맨드센터 보고
- **TL**: PL 계획 검증 + 리스크 분석 + 대시보드 관리
- **Coder/Reviewer/Tester**: PL 지시 수행 (구현/리뷰/테스트)
- **커맨드센터**: Mac Studio에서 직접 Claude Code Opus 운영. 보고 수신, 의사결정, git push, 배포

통신은 Slack(실시간 지시/보고)과 Akaxa(비동기 태스크/영구 기록)를 병행한다. Wave 패턴으로 병렬 처리하며, 같은 파일 충돌을 방지한다.

## 새 프로젝트 클론

Claude Team Bible v2.0에 6단계 체크리스트가 정의되어 있다: Slack App 5개 생성 -> bridge.py 복사 -> 에이전트 MD 작성 -> 환경변수 설정 -> 대시보드 구성 -> 커맨드센터 이관 패키지 준비.

## 알려진 제약사항

- 세션 영속화 없음 (인메모리, JSON 파일 백업)
- 동시 처리 제한 (에이전트당 1건)
- 에러 자동 복구 없음
- 로그 제한 (50개)
- CLI 5분 타임아웃 대응 필요

## 관련 시스템

- [[auto-email]] - 주요 개발 대상 프로젝트
- [[eywa]] - Claude Team 경험 기반의 범용 오케스트레이션 플랫폼

## 관련 소스

- [claude-team-bible-v2.0---클론-구축-매뉴얼.md](../../raw/claude-team-bible-v2.0---클론-구축-매뉴얼.md)
- [claude-team-bible---구축-매뉴얼-v1.0.md](../../raw/claude-team-bible---구축-매뉴얼-v1.0.md)
- [claude-team---머신-배치도.md](../../raw/claude-team---머신-배치도.md)
- [claude-team---팀-운영-현황-2026-03-23.md](../../raw/claude-team---팀-운영-현황-2026-03-23.md)
- [claude-team-README-섹션-auto-email에서-분리.md](../../raw/claude-team-README-섹션-auto-email에서-분리.md)
