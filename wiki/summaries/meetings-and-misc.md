---
title: 미팅 및 기타 문서 요약
tags: [미팅, UT Austin, akaxa, 세션로그, GPT]
created: 2026-04-06
updated: 2026-04-06
sources: [2026-04-03-오전미팅.md, ut-austin-입시-전략-자녀-로드맵-2026-03-27.md, from-gpt.md, claude-session-2026-03-21.md, akaxa-session-2026-03-21.md]
---
# 미팅 및 기타 문서 요약

미팅 기록, 개인 프로젝트, AI 세션 로그 등 분류가 다양한 문서 5건을 정리한다.

---

## 문서 목록

| 문서 | 유형 | 날짜 | 핵심 |
|------|------|------|------|
| 2026-04-03 오전 미팅 | 회의록 | 2026-04-03 | ToBe Networks 주간 업무 점검 |
| UT Austin 입시 전략 | 개인/교육 | 2026-03-27 | 자녀 대학 입시 로드맵 |
| from-gpt | 세션 메모 | 2026-03-21 | GPT 세션 저장 요청 기록 |
| claude-session-2026-03-21 | 세션 로그 | 2026-03-21 | Claude Code에서 Akaxa REST API 테스트 |
| akaxa-session-2026-03-21 | 세션 로그 | 2026-03-21 | Akaxa.space 개발 현황 전체 스냅샷 |

---

## 1. 2026-04-03 오전 미팅

[[ToBe Networks]] 경영 회의. 4개 핵심 안건을 다룸:

- **키크론 광고:** 매출 감소 대응 방향 결정 필요 (유지/축소/재입고 후 재집행)
- **에이퍼 4월 행사:** 캠페인 우선순위, 풀빌라갈래 컨택, S1 입고 지연 영향
- **브랜드스토어:** 메타 협력광고, 가정의달 행사, IT템 발굴단, 29CM 기획전
- **발주/운영 리스크:** KEYCHRON46099 잔금, J8 SKU명, M6S 납기, PI 미수령

담당자: 강성묵(총괄), 문소현(브랜드스토어), 김성은(에이퍼), 전은영(콘텐츠), 김경효(발주/납기)

---

## 2. UT Austin 입시 전략

중2 자녀의 [[UT Austin]] **AET(Arts and Entertainment Technologies)** 전공 입시 로드맵.

- **전공 적합성:** 블렌더/로블록스/마인크래프트 코딩 경험이 AET 커리큘럼(3D, 게임, PLAI)과 잘 맞음
- **입시 핵심:** 포트폴리오 불요, 에세이가 합격의 열쇠. 창작 경험을 구체적으로 기술
- **비용:** 연 약 7,900만원 (4년 총 3억+), 장학금 미조사
- **고교 경로:** 일반고 + 독학 포트폴리오 전략 권장
- **로드맵:** 중2(블렌더 완성작 3개) -> 중3(Python+AI) -> 고1~2(포트폴리오 웹사이트) -> 고3(지원서)

---

## 3. from-gpt

GPT 세션에서 [[Akaxa.space]]에 대화 내용 저장을 요청한 짧은 메모. 인증 플로우와 vault 데이터 11건 로딩 내역을 기록한다.

---

## 4. Claude Code 세션 (2026-03-21)

Claude Code 환경에서 [[Akaxa.space]] REST API를 직접 테스트한 세션 로그.

- akaxa.space llms.txt 확인 및 REST API 인증 (별 이름: Fornax)
- vault/list API로 저장 메모리 4건 조회
- 별 이름 뒤 공백 포함 필요, Bearer 토큰 인증 방식 확인

---

## 5. Akaxa.space 세션 (2026-03-21)

[[Akaxa.space]] 프로젝트의 전체 현황 스냅샷.

- **스택:** Hono + PostgreSQL + Redis + Next.js + Three.js, EC2 t3.small
- **기능:** remember/recall/forget/share/explore, 크로스 AI 인증, AES-256-GCM 암호화
- **인프라:** Let's Encrypt SSL, GitHub Actions CI/CD, PM2, nginx
- **SEO:** Google Search Console, Smithery, Glama.ai, awesome-mcp-servers PR 제출
- **사용자 계정:** wooho.yang@tbnws.com / Fornax
- **미완료:** Custom GPT 등록, Reddit 포스팅, Bing Webmaster, mcp.so 등록
