---
title: Eywa 프로젝트 요약
tags: [eywa, 오케스트레이션, claude-team, bridge, slack]
created: 2026-04-06
updated: 2026-04-06
sources: [eywa---세션-요약-2026-03-25-첫-세션.md, eywa---프로젝트-설계-문서-v1.0.md, eywa---프로젝트-설계-문서-v2.0-통합.md, eywa-1.0-소스코드-1-4.md, eywa-1.0-소스코드-2-4.md, eywa-1.0-소스코드-3-4.md, eywa-1.0-소스코드-4-4.md, eywa-project---current-state-2026-03-25.md]
---
# Eywa 프로젝트 요약

## 개요

[[Eywa]]는 여러 머신에 분산된 [[Claude CLI]] 인스턴스를 팀으로 구성하여 [[Slack]] 기반으로 협업하는 오케스트레이션 플랫폼이다. MIT 라이선스 오픈소스 프로젝트.

**GitHub**: https://github.com/woohoyang-oss/eywa (private)

## 해결하는 3가지 병목

| 병목 | 원인 | Eywa의 해결 |
|------|------|-------------|
| 디바이스 | CLI subprocess + 파일 I/O가 무거움 | 멀티 머신 분산 |
| 계정 | Anthropic 계정별 rate limit | 노드별 다른 계정 |
| 세션 | 세션 길어지면 컨텍스트 윈도우 포화 | 자동 세션 갱신 |

## 아키텍처

```
[웹 브라우저] <-- HTTP/WebSocket --> [Eywa 서버 :3001] (로컬)
                                          |
                                     SSH 연결
                                          |
                       [맥북]         [맥미니]        [맥스튜디오]
                     bridge.py      bridge.py       bridge.py
```

핵심 흐름: Slack 멘션 → [[bridge.py]] → Claude CLI subprocess → Slack 스레드 응답

## 기술 스택

| 레이어 | 기술 |
|--------|------|
| 프론트엔드 | Next.js 16.2.1 + shadcn/ui v4 + Tailwind CSS (dark theme) |
| 백엔드 | Express + WebSocket + SQLite (server/, port 3001) |
| 브릿지 | Python (slack-bolt, slack-sdk, python-dotenv) |
| 통신 | Slack Socket Mode (WebSocket) |
| 모니터링 | SSH tail -f → WebSocket → 브라우저 실시간 스트리밍 |

## 설계 v1 → v2 변화

v1.0은 기본 아키텍처와 셋업 위자드 6단계를 정의했다. v2.0에서 크게 확장:

- **역할별 툴체인**: Tester(Playwright, Jest), Reviewer(ESLint, semgrep), DevOps(Docker, Terraform, kubectl), Documentation(TypeDoc, Sphinx) 등 역할 선택 시 도구 체크리스트 자동 표시
- **클린 세션 아키텍처**: 세션을 일회용으로 취급하고 메모리 레이어로 연속성 유지. 세션 헬스체크 3신호(호출수/시간/레이턴시), 레이턴시 스파이크 선제 감지(롤링 평균 3x baseline), Self-Reset 4단계(Summarize → Persist → Terminate → Restart)
- **메모리 레이어**: [[Akaxa]] 디폴트 + Notion/Google Sheets 옵션. 에이전트가 세션 요약/결정사항/버그/아키텍처를 자동 저장
- **리소스 모니터링**: 노드별 CPU/MEM 실시간 게이지(5초 SSH 수집), 쓰로틀 설정(CPU 80%/95%, MEM 75%/90%), 추천 엔진(리소스 기반 opus/sonnet/haiku 선택), 워커 동적 추가/제거/이동
- **인증 3방식**: CLI OAuth(Keychain 주입), API 키, OpenAI 호환

## 소스코드 구성 (4파트)

| 파트 | 파일 | 내용 |
|------|------|------|
| 1/4 | server/package.json, db.js, deployer.js | DB 스키마(SQLite), 배포 관리 |
| 2/4 | index.js, log-streamer.js, monitor.js, routes/akasha.js | Express 서버, WebSocket 로그 스트리밍, 모니터링 |
| 3/4 | routes/ (deploy-envs, monitor, roles, sessions, setup, team) | REST API 라우터 전체 |
| 4/4 | ssh.js, templates/bridge.py | SSH 연결 풀, Slack-Claude 브릿지 |

## 주요 구현 기능 (2026-03-25 기준)

1. **셋업 위자드 6단계**: Team → Members → Roles/Claude Settings → Integrations → Deploy Envs → Summary/Deploy
2. **대시보드**: CPU/MEM/Session 게이지, 상태 표시, 액션 버튼
3. **노드 추가**: Join 토큰 생성 → curl 설치 명령 표시
4. **Direct Chat**: 대시보드 UI에서 특정 노드의 Claude CLI와 직접 대화
5. **로그 뷰어**: 실시간 로그 스트리밍 + 레벨/키워드 필터
6. **CLI-First Setup**: `curl ... | bash` → Claude CLI가 대화형 셋업 진행
7. **Apply Recommendation**: 리소스 기반 벌크 모델 최적화

## 현재 상태

- GitHub 4 commits, 17+ files
- 프론트/백엔드 기본 구조 완성
- 영문 UI로 전환 완료
- SSH 자동 키 감지 (ed25519 → rsa → ecdsa), 로컬 멤버 `is_local` 플래그
