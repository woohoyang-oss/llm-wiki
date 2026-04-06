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

## [2026-04-06] ingest | 152개 소스 일괄 인제스트
- Entity pages 11개 생성: auto-email, claude-team, eywa, claude-assistant, keychron-supply-tracker, kychr-space, akaxa-space, claudex, tobe-mcp, rhkorea, auto-dm
- Concept pages 6개 생성: bridge-architecture, mailbox-architecture, claude-team-bible, ci-cd-pipeline, gsd2-methodology, testing-strategy
- Summary pages 10개 생성: 그룹별 통합 요약 (auto-email, claude-team, tester, eywa, claude-assistant, keychron, akaxa, plans, skills, misc)
- index.md 전체 갱신 (27개 위키 페이지 카탈로그)
- 총 27개 위키 페이지, 152개 소스 기반
