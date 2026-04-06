---
title: Auto-Email 프로젝트 종합 요약
tags: [auto-email, 버그, 메일박스, 챗봇, 시드데이터, 프로덕션]
created: 2026-04-06
updated: 2026-04-06
sources: [auto-email-bugfix-2026-03-23.md, auto-email-ux-디자인-요구사항.md, b3-분석---via-포워딩-메일-오판-상세.md, 메일박스-구조-분석--테스트-시나리오.md, 메일박스-운영-방침-확정.md, 버그큐---2026-03-24-세션-보고.md, 시드-gap-분석-중간-결과-2026-03-24.md, 챗봇-기능-gap-분석-결과-2026-03-24.md, 태스크-챗봇-기능-완성도-점검-2026-03-24.md, 프로덕션-점검-결과-critical-9건-2026-03-23.md]
---
# Auto-Email 프로젝트 종합 요약

## 시스템 개요

[[auto-email]]은 Gmail 포워딩 기반 AI 고객 응대 자동화 플랫폼이다. 여러 브랜드([[GTGear]], [[Keychron]], [[Aiper]] 등)의 CS 메일을 단일 통합 수신함(ai@tbe.kr)으로 집약하고, AI가 자동 초안을 생성하여 승인 후 발송하는 구조이다.

**기술 스택**: FastAPI(백엔드) + Next.js(프론트) + Celery/Redis(비동기) + PostgreSQL + Docker, EC2 배포. AI 모델은 [[claude-sonnet-4-6]]을 사용한다.

## 메일박스 구조

- 단일 통합 수신: 각 브랜드 Gmail에서 ai@tbe.kr로 포워딩
- `X-Forwarded-To` 헤더로 원래 수신 주소 감지 → `MailboxResolver`가 DB 조회하여 브랜드/카테고리 매칭
- 회신 시 Gmail "Send As"로 원래 브랜드 주소 발송
- 새 브랜드 추가 시 코드 변경 불필요 (DB 동적 조회)
- OAuth 직접 연동은 브랜드 10개 이상 시 재검토 예정

## 주요 버그 (B1-B10)

| 버그 | 심각도 | 내용 | 상태 |
|------|--------|------|------|
| B1 | Critical | 모든 메일에 AI 반응 (인증번호, 시스템 알림 포함) | 화이트리스트 적용 |
| B2 | Critical | 내부 직원 발신에 AI 반응 (@tbnws.com) | 필터 추가 |
| B3 | Critical | via 포워딩 메일 오판 (165건, 110건 방치) | via 패턴 감지 구현 |
| B4 | Major | 인증번호/시스템 메일 완료 표시 누락 | ignored 상태 도입 |
| B5 | Critical | 스케줄 삭제해도 실행 지속 | DatabaseScheduler 통합 |
| B6 | Critical | 태스크 12,679건 폭주 | Redis seen-cache 적용 |
| B7 | Major | Beat 스케줄러 25분 다운 무감지 | restart:unless-stopped (와치독 미구현) |
| B8 | Critical | 공용 메일함 새 메일 안 들어옴 | DB 저장 로직 수정 |
| B9 | Critical | process_email_task 1,634회 연속 실패 | 수정 완료 |
| B10 | Minor | 테스트 메일 제목 깨짐 | 실제 메일은 정상 |

B3 상세: Gmail 포워딩 시 From 헤더가 "고객명 via support@gtgear.co.kr"로 재작성되어 내부 발신으로 오판. 해결 방안으로 via 앞 이름이 시스템명이면 ignored, 개인이면 AI 처리하도록 구현.

## 프로덕션 점검 결과

Critical 9건 전부 완료: embed_passage 연결, 지식문서 토큰 제한, embed_query 동기 블로킹, LocalLLM 예외 처리, Celery max_retries, 승인 큐 race condition(Redis 락), AI 빈 초안 방지, XSS 이스케이프, sent_at 자동 설정. 추가로 담당자 삭제 FK 버그, 스케줄 UPDATE 500, 태그 API 미구현도 수정 배포 완료.

## 챗봇 기능 GAP

- admin.py 총 197개 엔드포인트 중 챗봇 연결 36종(18%) → 60종으로 확장
- 조회 커버율 90%, 실행/변경 커버율 15%로 GAP 큼
- 우선순위 높음: 이메일 회신/포워딩/상태변경, 지식베이스 CRUD, 템플릿/매크로 CRUD
- 아키텍처: ai-chatbot.tsx(957줄) + POST /admin/chat → Claude + [action] 태그 파싱

## 시드 데이터 현황

총 105건 등록 완료: 배정규칙 23, 지식문서 14~15, 템플릿 9~12, 매크로 10, 태그 21, 자동화 2~6, 서명 3, SLA 4, 규칙 12. GAP 영역으로 채널 파트너 CS, 세금계산서 B2B 처리, 해외 문의 전용 룰/템플릿이 부족하여 보충 필요.

## UX 디자인 과제

대시보드 차트가 회색/연회색으로 데이터가 비어 보이는 문제. shadcn/ui 공식 예제 수준으로 개선 필요 (파란색 gradient fill, interactive tooltip, hover 효과). Recharts + shadcn ChartContainer 사용 중.
