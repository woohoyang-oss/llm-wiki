# Eywa — Claude Team Orchestration System

> 여러 머신에 분산된 Claude CLI 인스턴스를 팀으로 구성하여 Slack 기반으로 협업하는 오케스트레이션 플랫폼.
> GitHub 공개 프로젝트.

---

## 1. 아키텍처

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

## 2. 기술 스택
- 서버: Node.js + Express + SQLite + WebSocket
- 프론트엔드: 순수 HTML + vanilla JS + CSS
- 브릿지: Python (slack-bolt, slack-sdk, python-dotenv)
- 통신: Slack Socket Mode (WebSocket)
- 모니터링: SSH tail -f → WebSocket → 브라우저

## 3. Claude 인증
- 방식 A: CLI OAuth (Keychain 주입)
- 방식 B: API 키

## 4. 셋업 위자드 (6단계)
Step 1~6: 팀 생성 → 멤버 추가 → 역할/Claude 설정 → 연동 → 배포 환경 → 요약 & 시작

## 5~11. (Covering):
- 로그 시스템, 세션 관리, Slack 통신 구조
- 역할별 모델, 서버 구조, 외부 연동
- Disclaimer
