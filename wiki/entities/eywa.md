---
title: Eywa - Claude Team 오케스트레이션 플랫폼
tags: [eywa, 오케스트레이션, Claude CLI, 멀티머신, 오픈소스]
created: 2026-04-06
updated: 2026-04-06
sources: [eywa---프로젝트-설계-문서-v2.0-통합.md, eywa---프로젝트-설계-문서-v1.0.md, eywa-project---current-state-2026-03-25.md, eywa---세션-요약-2026-03-25-첫-세션.md]
---
# Eywa - Claude Team 오케스트레이션 플랫폼

## 개요

Eywa는 여러 머신에 분산된 Claude CLI 인스턴스를 팀으로 구성하여 Slack 기반으로 협업하는 오케스트레이션 플랫폼이다. [[claude-team]]에서 검증된 bridge.py 패턴을 범용화하여 누구나 멀티머신 AI 에이전트 팀을 구축할 수 있도록 설계되었다. MIT 라이선스의 오픈소스 프로젝트이다.

GitHub: github.com/woohoyang-oss/eywa

## Claude CLI의 3가지 병목과 Eywa의 해결

| 병목 | 원인 | Eywa의 해결 |
|------|------|-------------|
| 디바이스 | CLI subprocess + 파일 I/O가 무거움 | 멀티 머신 분산 |
| 계정 | Anthropic 계정별 rate limit | 노드별 다른 계정 |
| 세션 | 컨텍스트 윈도우 포화 | 자동 세션 갱신 |

## 아키텍처

```
[웹 브라우저] <-- HTTP/WebSocket --> [Eywa 서버 :3001] (로컬)
                                          |
                                     SSH 연결
                                          |
                         +---------+------+--------+
                       [맥북]    [맥미니]    [맥스튜디오]
                      bridge.py  bridge.py   bridge.py
```

핵심 흐름: Slack 멘션 -> bridge.py -> Claude CLI subprocess -> Slack 스레드 응답. 웹 대시보드에서 전체 팀 상태를 모니터링하고 제어할 수 있다.

## 기술 스택

- **서버**: Node.js + Express + SQLite + WebSocket (port 3001)
- **프론트엔드**: Next.js 16.2.1 + shadcn/ui v4 + Tailwind CSS (다크 테마)
- **브릿지**: Python (slack-bolt, slack-sdk) - Claude CLI를 subprocess로 래핑
- **통신**: Slack Socket Mode (WebSocket) + SSH (노드 간)
- **인증**: CLI OAuth(Keychain 주입), API Key, OpenAI Compatible 3방식 지원

## 주요 기능

1. **셋업 위자드 (6단계)**: 팀 생성 -> 멤버 추가 -> 역할/Claude 설정 -> 연동 -> 배포 환경 -> 요약/시작
2. **대시보드**: 노드별 CPU/MEM 실시간 게이지(5초 SSH 수집), 상태 인디케이터, 액션 버튼
3. **노드 추가**: Join Token 생성 -> curl 설치 명령으로 새 노드 자동 등록
4. **Direct Chat**: 대시보드 UI에서 특정 노드의 Claude CLI와 직접 대화
5. **로그 뷰어**: 실시간 로그 스트리밍(SSH tail -f -> WebSocket), 레벨/키워드 필터링
6. **리소스 모니터링**: 쓰로틀 설정(CPU 80%/95%, MEM 75%/90%), 모델 추천 엔진
7. **클린 세션**: 세션을 일회용으로 취급하고 메모리 레이어(Akaxa, Notion, Google Sheets)로 연속성 유지

### 역할별 툴체인

Tester(Playwright, Jest, pytest), Reviewer(ESLint, Ruff, semgrep), DevOps(Docker, AWS CLI, Terraform) 등 역할 선택 시 도구 체크리스트가 표시된다.

## 현재 상태 (2026-03-25)

- GitHub 커밋 4건, 17+ 파일
- 셋업 위자드, 대시보드, Direct Chat, 로그 뷰어, 설정 페이지 구현 완료
- CLI-first 셋업 시스템(setup.sh + Claude CLI 인터랙티브) 추가
- 전체 UI 영문화 완료

## 분리 프로젝트

태스크 큐 기능이 **eywa-queue**로 독립 분리되었다. SQLite 기반 태스크 오케스트레이션, DAG 의존성 관리, 리소스 락 등의 기능을 전담한다. 상세는 [[eywa-queue]]를 참조.

## 관련 시스템

- [[claude-team]] - Eywa의 원형이 된 실전 운영 시스템
- [[auto-email]] - Claude Team이 개발한 프로젝트

## 관련 소스

- [eywa---프로젝트-설계-문서-v2.0-통합.md](../../raw/eywa---프로젝트-설계-문서-v2.0-통합.md)
- [eywa---프로젝트-설계-문서-v1.0.md](../../raw/eywa---프로젝트-설계-문서-v1.0.md)
- [eywa-project---current-state-2026-03-25.md](../../raw/eywa-project---current-state-2026-03-25.md)
- [eywa---세션-요약-2026-03-25-첫-세션.md](../../raw/eywa---세션-요약-2026-03-25-첫-세션.md)
