---
title: Auto Email - AI 이메일 자동응답 시스템
tags: [auto-email, AI, 이메일, 자동응답, FastAPI, Celery]
created: 2026-04-06
updated: 2026-04-06
sources: [auto-email-bugfix-2026-03-23.md, auto-email-ux-디자인-요구사항.md, b3-분석---via-포워딩-메일-오판-상세.md, 버그큐---2026-03-24-세션-보고.md, 메일박스-구조-분석--테스트-시나리오.md, 메일박스-운영-방침-확정.md, 프로덕션-점검-결과-critical-9건-2026-03-23.md, claude-team-배포-재발-방지-조치.md]
---
# Auto Email - AI 이메일 자동응답 시스템

## 개요

Auto Email은 ToBe Networks의 AI 기반 이메일 자동응답 시스템이다. 여러 브랜드(GTGear, Keychron 등)의 고객 문의 메일을 단일 통합 수신함으로 수집하고, AI가 초안을 생성한 뒤 담당자 승인을 거쳐 발송하는 구조로 운영된다. [[claude-team]]이 개발을 담당했으며, GitHub Actions를 통해 EC2에 자동 배포된다.

## 아키텍처: 포워딩 + Send As

시스템의 핵심은 Gmail 포워딩과 Send As 조합이다.

1. **수신**: 각 브랜드 메일(support@gtgear.co.kr 등)에 도착한 메일이 Gmail 포워딩을 통해 통합 수신함(ai@tbe.kr)으로 전달된다.
2. **분류**: MailboxResolver가 X-Forwarded-To 헤더를 읽어 원래 수신 주소를 감지하고, DB에서 브랜드/카테고리를 매칭한다.
3. **AI 처리**: pipeline.py가 메일을 분석하여 AI 초안을 생성한다. Celery 태스크 기반으로 비동기 처리된다.
4. **발송**: 승인된 초안은 Gmail Send As 기능으로 원래 브랜드 주소에서 발송된다.

새 브랜드 추가 시 코드 변경 없이 DB 등록 + Gmail 포워딩/Send As 설정만으로 확장 가능하다. OAuth 직접 연동은 브랜드 10개 이상 시 재검토 예정이다.

## 주요 버그 이력

2026년 3월 집중 운영 기간 중 다수의 Critical 버그가 발견되어 수정되었다.

### Critical 이슈 (해결 완료)
- **B1. 무차별 AI 반응**: 인증번호, 시스템 알림 등 모든 메일에 AI가 반응. 화이트리스트 및 ignored 처리로 해결.
- **B3. via 포워딩 메일 오판**: Gmail 포워딩 시 From 헤더가 재작성되어(예: "홍길동 via support@gtgear.co.kr") 내부 발신으로 오인. from_name의 via 패턴 감지 로직으로 수정.
- **B5. 스케줄 삭제 후 계속 실행**: DatabaseScheduler 통합으로 해결.
- **B6. 태스크 폭주 (12,679건)**: Redis seen-cache와 after:yesterday 필터 적용.
- **B9. process_email_task 연속 실패 (1,634회)**: 두 가지 버그 모두 수정 완료.

### 프로덕션 점검 (Critical 9건 완료)
embed_passage 연결, 지식문서 토큰 제한, Celery max_retries, 승인 큐 race condition(Redis 락), AI 빈 초안 방지, XSS 이스케이프 등 9건 전부 해결.

### 배포 재발 방지
GitHub Actions Deploy 시 rsync로 코드가 덮어씌워져 EC2 수동 수정이 원복되는 문제 발생. 원칙: EC2에서 수동 수정하지 않고 항상 main 코드를 수정하여 푸시.

## 현재 상태

- 프론트엔드: Next.js + shadcn/ui + Recharts 기반 대시보드. UX 개선 진행 중.
- 백엔드: FastAPI + Celery + Redis + PostgreSQL. EC2 운영.
- 시드 데이터: 배정 규칙 23건, 서명 3건, SLA 4건, 템플릿 9건, 매크로 10건, 태그 21건, 지식베이스 15건, 메일박스 4건 등록.
- 브랜치: v2-phase0-implementation

## Claude Team 에이전트 구성

5-agent 역할 기반 시스템으로 개발이 진행된다:

| 역할 | 담당 |
|------|------|
| Team Lead | 전체 조율 및 의사결정 |
| Project Leader | 태스크 분배 및 진행 관리 |
| Coder | 구현 담당 |
| Reviewer | 코드 리뷰 및 품질 관리 |
| Tester | 테스트 작성 및 실행 |

Slack **#claude-team** 채널에서 오케스트레이션되며, [[akaxa-space]]를 통해 세션 이관 패키지를 저장/로드한다. 현재 **v2.8-2.9** 버전으로 97% 프로덕션 준비 상태다.

주요 최근 작업:
- 모바일 뷰 개편: ResponsiveDialog 도입, 28페이지 감사 완료
- 운영 Runbook 정비: 디스크 관리, 컨테이너 헬스체크, 모니터링 절차 문서화

## 관련 시스템

- [[claude-team]] - 개발 팀 운영
- [[eywa]] - 팀 오케스트레이션 플랫폼

## 관련 소스

- [auto-email-bugfix-2026-03-23.md](../../raw/auto-email-bugfix-2026-03-23.md)
- [auto-email-ux-디자인-요구사항.md](../../raw/auto-email-ux-디자인-요구사항.md)
- [b3-분석---via-포워딩-메일-오판-상세.md](../../raw/b3-분석---via-포워딩-메일-오판-상세.md)
- [버그큐---2026-03-24-세션-보고.md](../../raw/버그큐---2026-03-24-세션-보고.md)
- [메일박스-구조-분석--테스트-시나리오.md](../../raw/메일박스-구조-분석--테스트-시나리오.md)
- [메일박스-운영-방침-확정.md](../../raw/메일박스-운영-방침-확정.md)
- [프로덕션-점검-결과-critical-9건-2026-03-23.md](../../raw/프로덕션-점검-결과-critical-9건-2026-03-23.md)
- [claude-team-배포-재발-방지-조치.md](../../raw/claude-team-배포-재발-방지-조치.md)
