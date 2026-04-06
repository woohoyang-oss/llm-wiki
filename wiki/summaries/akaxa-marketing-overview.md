---
title: Akaxa.space 마케팅 활동 요약
tags: [akaxa, 마케팅, Reddit, 토큰비교, AI메모리]
created: 2026-04-06
updated: 2026-04-06
sources: [reddit-post-final.md, reddit-posts.md, reddit-post-claudeai.md, why-akaxa.md, reddit-reply-custom-skill.md, token-comparison-test.md]
---
# Akaxa.space 마케팅 활동 요약

[[Akaxa.space]]의 Reddit 마케팅 전략, 핵심 메시지, 포스트 버전 히스토리를 정리한다.

---

## 핵심 메시지

[[Akaxa]]의 차별점은 **토큰 효율성**과 **제로 설정**이다.

| 비교 항목 | Notion + Gmail | Akaxa |
|-----------|---------------|-------|
| 툴 스키마 | ~4,000 토큰 | ~200 토큰 |
| API 호출 | 4회 | 1회 |
| 요청/응답 | ~4,200 토큰 | ~1,600 토큰 |
| **총 토큰** | **~8,200** | **~1,800** |
| **절감율** | | **~4.5x** |
| 소요 시간 | 30초~1분+ | 5~10초 |

핵심 비유: AI를 위한 포스트잇. 붙이고, 가져가고, 끝.

---

## Reddit 포스팅 전략

### 서브레딧별 포스트 배분

| 서브레딧 | 포스트 버전 | 톤 |
|----------|------------|-----|
| r/ClaudeAI | v3 Post 1 / 전용 포스트 | Claude 중심, 기술 상세 |
| r/ChatGPT | v3 Post 1 | 범용 AI 메모리, 간결 |
| r/artificial | v3 Post 1 | 범용 |
| r/mcp | v3 Post 2 | MCP 서버 기술 상세 (tools, auth, transport) |
| r/SideProject | v3 Post 3 | 빌더 관점, 기술 스택, 교훈 |
| r/webdev | v3 Post 3 | 개발 과정 중심 |

### 포스트 버전 히스토리

1. **v3 (reddit-posts.md):** 서브레딧별 3종 포스트 초안. 기능 설명 중심.
2. **ClaudeAI 전용 (reddit-post-claudeai.md):** Claude 사용자 타겟, readme 페이지 기반 사용법 강조.
3. **Final (reddit-post-final.md):** 개인 경험 중심 스토리텔링으로 전환. "messy vibe coder" 페르소나. Tailscale 경험담 + Notion 비교 표 + 실제 토큰 수치.

---

## 주요 마케팅 자산

### why-akaxa.md
Notion 대비 Akaxa의 장점을 정리한 내부 메시지 가이드. 토큰 비교 수치(최대 7.5x 절감)와 사용 시나리오를 담고 있다.

### token-comparison-test.md
AI에게 직접 실행시키는 **토큰 비교 테스트 프롬프트**. Notion과 Akaxa로 동일 문서를 저장한 뒤 실제 토큰 소비량을 표로 비교하게 한다. 마케팅 증거 자료로 활용.

### reddit-reply-custom-skill.md
FAQ 대응용 답변 템플릿. "커스텀 스킬로 쓸 수 있나?" 질문에 대해 "readme 하나 읽히면 끝, MCP/스킬 설정 불필요"라는 핵심 메시지를 전달한다.

---

## 마케팅 키 포인트

- **진입 장벽 제로:** "Read akaxa.space/readme" 한 문장으로 시작
- **크로스 AI:** GPT에서 저장, Claude에서 불러오기 가능
- **별 이름 인증:** 비밀번호 대신 별자리 이름 사용 (다국어 퍼지 매칭)
- **보안:** AES-256-GCM 사용자별 암호화
- **경쟁 포지셔닝:** Mem0, Hindsight 등은 MCP 설정 필요, Akaxa는 fetch만으로 동작
