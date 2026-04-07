---
title: Wiki Log
---

# Log

## [2026-04-06] init | Wiki initialized
- 비즈니스 위키 초기 구조 생성
- 디렉토리: raw/, wiki/summaries/, wiki/entities/, wiki/concepts/
- CLAUDE.md 스키마 작성

## [2026-04-06] bulk-ingest | Akaxa vault → raw/ 일괄 저장
- akaxa.space vault 141개 항목 중 116개 마크다운 파일로 저장
- 제외: PEM 키 3개, 토큰/크리덴셜 3개, 테스트 데이터 5개 등 (보안/무의미)
- 카테고리: auto-email, claude-team, eywa, keychron, akaxa-marketing, claude-assistant 등
- 인제스트 대기 상태 (wiki 페이지 미생성)

## [2026-04-06] local-copy | 로컬 Claude 생성 파일 복사
- 로컬 Mac에서 Claude가 생성한 md 파일 34개 추가 복사
- CLAUDE.md 2개: tobe-repo, kychr.space
- Plan 파일 8개: rhkorea 리부트, auto-dm 기획, kychr 대시보드 등
- Skill 파일 8개: tobe-repo 스킬 (inventory-check, cs-pending, trend 등)
- AGENTS.md 2개: rhkorea, auto-dm
- Memory 2개: auto-dm GSD-2 방법론
- 전체 raw 파일: 152개

## [2026-04-06] github-ingest | GitHub 11개 레포 md 인제스트
- 대상 레포: kychr.space, auto-email, eywa-queue, tobe-mcp-improvement, akaxa.space, claude-team, weekly-inventory, tbe.kr, tbnws-ai, tbnws-ad, gfa-auto
- GitHub API로 전체 md 파일 목록 조회 → 중복 제거 후 40개 신규 파일 다운로드
- 새 Entity 3개: eywa-queue, tobe-mcp-improvement, weekly-inventory
- 기존 Entity 4개 업데이트: auto-email(Claude Team 에이전트 구성, 모바일 개편, v2.8-2.9), akaxa-space(활용 사례), tobe-mcp(개선 로드맵), eywa(분리 프로젝트)
- 전체 raw/ 347개, 위키 52개 페이지
- index.md 최종 갱신

## [2026-04-06] bulk-ingest-2 | 155개 추가 소스 인제스트
- collected-md/(186), tobe-macmini/(29), wooho-studio/(67) 총 282개 md 파일 스캔
- 중복 제거: 이미 raw/에 있음 8개, 폴더 간 중복 119개 제거
- 고유 신규 155개 파일 raw/에 복사 (전체 raw/ 307개)
- 새 Entity 8개: tbnws-ad, sabang-dashboard, keychron-launcher, tobe-ai, tbnws-admin, gfa-ops-ai, pawswing, aiper
- 새 Concept 8개: gfa-funnel-framework, messy-middle, attribution-modeling, ad-decay-saturation, product-code-hierarchy, confidence-routing, sales-data-pipeline, telegram-integration
- 새 Summary 6개: auto-email-docs-overview, knowledge-base-overview, tbnws-ad-sessions-overview, gfa-intelligence-overview, tobe-ai-schema-overview, sabang-overview
- index.md 갱신 (49개 위키 페이지 카탈로그)

## [2026-04-06] ingest | 152개 소스 일괄 인제스트
- Entity pages 11개 생성: auto-email, claude-team, eywa, claude-assistant, keychron-supply-tracker, kychr-space, akaxa-space, claudex, tobe-mcp, rhkorea, auto-dm
- Concept pages 6개 생성: bridge-architecture, mailbox-architecture, claude-team-bible, ci-cd-pipeline, gsd2-methodology, testing-strategy
- Summary pages 10개 생성: 그룹별 통합 요약 (auto-email, claude-team, tester, eywa, claude-assistant, keychron, akaxa, plans, skills, misc)
- index.md 전체 갱신 (27개 위키 페이지 카탈로그)
- 총 27개 위키 페이지, 152개 소스 기반

## [2026-04-07] wiki-mcp | Wiki MCP 서버 구축
- wiki-mcp 서버 프로젝트 생성: /Users/yangwooho/wiki-mcp/
- Python 3.13 + MCP SDK 1.27, stdio + SSE 이중 모드
- MCP 도구 10개: user 5 (search, read, list, graph, contribute) + admin 5 (edit, delete, approve, reject, ingest)
- 역할 기반 권한: user(읽기/기여), admin(전체 CRUD), Bearer 토큰 인증
- 위키 인덱서: frontmatter 파싱, 키워드 검색, 태그 필터, [[backlink]] 그래프
- 기여 플로우: contribute → raw/contributions/ → admin approve → wiki/ 반영
- 조직별 문서 15개 생성: wiki/org/{cs,sales,marketing,as,md}/ 각 index/onboarding/faq
- 직원 셋팅 가이드: wiki-mcp/docs/setup-guide.md
- CLAUDE.md, index.md 갱신 (67개 위키 페이지)
