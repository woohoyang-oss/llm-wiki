---
title: Claude Team 타임라인 (2026-03-23 ~ 03-27)
tags: [claude-team, 타임라인, 커맨드센터, 이관패키지, auto-email]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-team---세션-로그-2026-03-23.md, claude-team---세션-이관-맥스튜.md, claude-team---일일-보고서-2026-03-23.md, claude-team---최종-보고서-2026-03-23.md, claude-team---최종-보고서-2026-03-23-v2.md, claude-team---커맨드센터-세션-보고-2026-03-24.md, claude-team---커맨드센터-이관-패키지-2026-03-24-1800.md, claude-team---커맨드센터-이관-패키지-2026-03-24.md, claude-team---커맨드센터-이관-패키지-2026-03-25-0920.md, claude-team-세션-종료-보고-2026-03-23.md, claude-team-커맨드센터-세션-보고-2026-03-23-오후.md, 커맨드센터-wave-진행-현황-2026-03-24.md]
---
# Claude Team 타임라인

## 팀 구성

[[Claude Team]]은 6명의 AI 에이전트를 [[Slack]] 브릿지로 연결한 병렬 개발 체계이다.

| 역할 | 머신 | 기능 |
|------|------|------|
| 커맨드센터 | 맥스튜 | 총괄 지휘, Claude Code VSCode Opus |
| TL (전략 어드바이저) | 맥북 | GAP 분석, 아키텍처 리뷰 |
| PL (프로젝트 리더) | 맥북 | CI/Deploy 관리, 태스크 배분 |
| Coder | 맥미니 | 코드 구현, 버그 수정 |
| Reviewer | 맥미니 | 코드 리뷰, 배포 판정 |
| Tester | 맥미니 + 맥스튜 | E2E 테스트, CRUD 점검, 시드 데이터 등록 |

통신: [[Slack]] Socket Mode (5봇) + [[Akaxa]] 이중 소통. [[bridge.py]]가 각 머신에서 Claude CLI subprocess를 구동.

---

## 2026-03-23 (Day 1) — 프로덕션 준비도 60% → 95%

### 새벽 세션 (01:30 완료)
- 긴급 5건 완료: config.py 시크릿 기본값 제거, Gmail 토큰 Fernet 암호화, 승인 DB 커밋 누락, 시드 텔레그램 토큰 분리, Gmail Rate Limit 처리
- 보강 8건: SLA 모니터링 배치, 룰 시드 5개, 승인 워크플로우 개선, LLM 라우팅 수정, AutomationExecutor 구현
- 중장기 4건: CI/CD GitHub Actions, Pub/Sub Watch 설계, 벡터 검색 기본 구현
- 테스트 62/62 전체 통과

### 오후 세션 (커맨드센터 맥스튜)
- 버그 수정 3건: 담당자 삭제 FK, 스케줄 UPDATE 500, 태그 API 404
- pre-commit hook 도입 (ruff check --fix + ruff-format)
- TL이 실제 메일 80건+ 분석 → 8유형 분류 (A/S, CS, 주문알림, 채널파트너, B2B, 시스템, 스팸, 해외)
- 시드 데이터 등록: 템플릿 +9, 매크로 +5, 태그 +21, SLA +1, 서명 +1 (총 105건)
- CRUD 전수 점검: 11메뉴 중 9 PASS, 2건 수정 후 PASS

### 세션 종료 보고
- CI 연속 성공 (#48~#51), Deploy 자동 배포 성공 (#33~#36)
- v2.8.0 배포 완료, 서버 성능 평균 35.8ms 응답
- API 테스트 16개 중 14개 통과 (batch-monitor, system 404)
- 브라우저 테스트 5/5 통과 (대시보드/메일함/룰/담당자/내메일함)

### 맥스튜 이관 시 배포 차단 3건
1. Gmail 토큰 마이그레이션 스크립트 (평문 → Fernet 암호화)
2. edit_and_send 발송 후 email.status 업데이트 누락
3. 벡터 검색 SQL 인젝션 (f-string → 바인드 파라미터)

---

## 2026-03-24 (Day 2) — Wave 1~4 병렬 실행

### 커맨드센터 세션 (이관 패키지 로드 → 4 Wave 병렬)

| Wave | 내용 | 결과 |
|------|------|------|
| Wave 1 | 버그 수정 + 시드 데이터 105건 | 배포 완료, changelog v2.8.1 |
| Wave 2 | 챗봇 CRUD 19종 추가 → 총 60종 | 배포 완료, event_based 자동화 연결 |
| Wave 3 | 대시보드/통계 shadcn v4 전면 개선 | 배포 완료 |
| Wave 4 | 메일 검색 챗봇 버그 수정 | 배포 완료 |

### 18:00 이관 패키지 (11커밋)
- asyncio event loop 충돌 해결, RFC 2822 날짜 파싱, 내부 발신 스킵
- 스킵 메일 DB 저장, 중복 메일 upsert, DatabaseScheduler 통합
- 화이트리스트 전환, restart:unless-stopped, Redis seen-cache
- 미해결: B3 via 포워딩, B4 ignored 상태, B7 와치독

---

## 2026-03-25 (Day 3) — via 패턴 해결 + status 체계 확립

### 09:20 이관 패키지 (약 15커밋)
- via 포워딩 처리 완료: from_name에 via가 있으면 외부 경유로 인식
- via 시스템(스마트스토어/Steam) → ignored, via 고객(개인명) → open
- status 5단계 체계 확립: processing / open / replied / done / ignored
- ignored 메일 UI 필터 + 사이드바 무시됨 폴더 추가
- 스팸 룰 매칭 → ignored 처리
- 현재 status 분포: received 351, done 112, ignored 89, open 4, replied 1
- 팀 태스크 배정 완료 (브릿지 미가동으로 응답 대기)

### 추가 산출물
- [[Claude Team Bible]] v1.0 → v2.0 작성
- [[bridge.py]] 상세 가이드 작성

---

## 이관 패키지 체계

커맨드센터 세션 전환 시 `/tmp/migration_package.md`를 로드하여 연속성 보장. 패키지에는 현재 상태, 미해결 버그, 팀 연결 정보(Slack User ID + 채널), 인프라 접속 정보(EC2, SSH, GitHub PAT), 다음 세션 할 일이 포함된다.

## 주요 성과 요약

- 프로덕션 준비도: 60% → 95%
- 커밋 수: 30건 이상 (Day 1) + 11건 (Day 2) + 15건 (Day 3)
- 챗봇 액션: 36종 → 60종
- 시드 데이터: 0 → 105건
- 버그 B1~B10 중 대부분 해결, B7 와치독만 미구현
