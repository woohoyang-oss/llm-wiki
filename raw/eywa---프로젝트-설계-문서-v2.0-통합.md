# Eywa — Claude Team Orchestration System

> 여러 머신에 분산된 Claude CLI 인스턴스를 팀으로 구성하여 Slack 기반으로 협업하는 오케스트레이션 플랫폼.
> GitHub 공개 프로젝트. MIT 라이선스.

---

## 1. 왜 Eywa인가?

Claude CLI는 3가지 병목이 존재한다:

| 병목 | 원인 | Eywa의 해결 |
|------|------|-------------|
| **디바이스** | CLI subprocess + 파일 I/O가 무거움 | 멀티 머신 분산 |
| **계정** | Anthropic 계정별 rate limit | 노드별 다른 계정 |
| **세션** | 세션 길어지면 컨텍스트 윈도우 포화 | 자동 세션 갱신 |

## 2. 아키텍처

```
[웹 브라우저] ← HTTP/WebSocket → [Eywa 서버 :3001] (로컬 맥북)
                                        |
                                   SSH 연결
                                        |
                         ┌──────────────┼──────────────┐
                     [맥북]         [맥미니]        [맥스튜디오]
                   bridge.py      bridge.py       bridge.py
```

핵심 흐름: Slack 멘션 → bridge.py → Claude CLI subprocess → Slack 스레드 응답

## 3~16. (Full design document covering):
- 기술 스택 (Node.js + Express + SQLite + WebSocket, Next.js + shadcn/ui v4)
- Claude 인증 (OAuth Keychain, API Key, OpenAI Compatible)
- 셋업 위자드 6단계
- 로그 시스템 (bridge + CLI debug, WebSocket streaming)
- 세션 관리 (자동 갱신)
- Slack 통신 구조
- 역할별 모델
- 메모리 레이어 (Akaxa, Notion, Google Sheets)
- 대시보드 (리소스 모니터링, 추천 엔진, 워커 동적 관리)
- 프로젝트 구조
- DB 스키마
- API 엔드포인트
- 디자인 원칙
- Disclaimer

(Full content available in vault key: "Eywa - 프로젝트 설계 문서 v2.0 (통합)")
