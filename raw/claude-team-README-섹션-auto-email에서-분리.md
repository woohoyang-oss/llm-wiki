## Claude Team 구조

```mermaid
flowchart TB
    subgraph Slack["💬 Slack #claude-team"]
        CH["멘션 기반 태스크 분배"]
    end

    subgraph M1["🖥️ Mac Studio — 커맨드센터"]
        CMD["Claude Code Opus\n소스 수정 + 배포\n노션 티켓 관리"]
    end

    subgraph M2["🖥️ Mac Mini"]
        CODER["💻 Coder\nbridge.py\n구현"]
        REV["🔍 Reviewer\nbridge.py\n코드 리뷰"]
        TEST["🧪 Tester\nbridge.py\n테스트"]
    end

    subgraph M3["💻 MacBook Pro"]
        TL["🎖️ Team Lead\nbridge.py\n전략"]
        PL["📋 Project Leader\nbridge.py\n관리"]
    end

    CH <--> TL & PL & CODER & REV & TEST
    CMD -->|"git push"| GH["GitHub\nCI/CD"]
    GH -->|"auto deploy"| EC2["☁️ EC2"]
```

### bridge.py 동작 방식

```
Slack 멘션 수신 → bridge.py → Claude CLI subprocess → 결과를 Slack 스레드로 응답
```

| 머신 | 역할 | 모델 |
|------|------|------|
| Mac Studio | 커맨드센터 (직접 운영) | Opus |
| Mac Mini | Coder, Reviewer, Tester | Sonnet |
| MacBook Pro | Team Lead, Project Leader | Sonnet |