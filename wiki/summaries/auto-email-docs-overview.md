---
title: Auto-Email 기술문서 종합 요약
tags: [auto-email, documentation, architecture]
created: 2026-04-06
updated: 2026-04-06
sources:
  - claude--auto-email--PROJECT_SPEC.md
  - auto-email--docs--implementation-plan.md
  - auto-email--docs--phase1-auto-pipeline.md
  - auto-email--docs--phase2-notification-assignment.md
  - auto-email--docs--phase3-rbac-permissions.md
  - auto-email--docs--phase4-personal-workspace.md
  - auto-email--docs--phase5-dashboard-analytics.md
  - auto-email--docs--deployment-guide.md
  - auto-email--docs--operations-guide.md
  - auto-email--docs--migration-guide.md
  - auto-email--docs--dev-framework.md
  - auto-email--docs--parallel-dev-guide.md
  - auto-email--docs--page-feature-specs.md
  - auto-email--docs--workflow-specs.md
  - auto-email--docs--ux-ui-guidelines.md
  - auto-email--docs--shared-infrastructure.md
  - auto-email--docs--data-integration-scenarios.md
  - auto-email--docs--server-migration-guide.md
  - auto-email--docs--database-dump.md
  - auto-email--docs--update-history.md
---

# Auto-Email 기술문서 종합 요약

## 개요

Auto-Email은 [[keychron]] / [[gtgear]] CS 팀의 이메일 업무를 자동화하는 시스템이다.
총 19개의 기술문서로 구성되며, 5단계 Phase로 나뉜 점진적 구현 계획을 따른다.

## PROJECT_SPEC (프로젝트 명세)

- 목표: 수동 이메일 처리 -> 자동 분류/배정/응답 파이프라인 구축
- 대상 채널: 네이버 스마트스토어, 자사몰, 해외 CS 이메일
- 핵심 기능: 자동 폴링, AI 분류, 담당자 배정, RBAC 권한 관리, 분석 대시보드

## Phase별 구현 계획

### Phase 1: 자동 폴링 파이프라인 (auto-pipeline)

- Gmail API / IMAP 기반 이메일 자동 수집
- AI 분류 엔진: 문의 유형(주문/배송/AS/반품/일반) 자동 태깅
- 파싱 파이프라인: 발신자 정보, 주문번호, 제품 코드 추출
- 큐 기반 비동기 처리 아키텍처
- 중복 메일 필터링 및 스레드 병합 로직

### Phase 2: 알림 및 배정 (notification-assignment)

- 담당자 자동 배정 규칙 엔진
- 문의 유형별/브랜드별/우선순위별 라우팅
- 실시간 알림: Slack, 텔레그램, 인앱 노티피케이션
- 에스컬레이션 정책: SLA 시간 초과 시 상위 담당자 자동 전달
- 배정 이력 추적 및 워크로드 밸런싱

### Phase 3: RBAC 권한 관리 (rbac-permissions)

- 역할 기반 접근 제어: Admin / Manager / Agent / Viewer
- 브랜드별 접근 범위 제한 (Keychron, GTGear, Aiper, PawSwing)
- 기능별 세분화 권한: 읽기, 응답, 배정 변경, 설정 관리
- JWT 기반 인증 및 세션 관리
- 감사 로그(Audit Log) 기록

### Phase 4: 개인 워크스페이스 (personal-workspace)

- 담당자별 메일함 뷰: 내 할당, 진행중, 완료
- 메일 필터/검색: 날짜, 유형, 브랜드, 상태별 필터링
- 메모/태그: 개인 메모 첨부 및 커스텀 태그
- 템플릿 응답: 자주 사용하는 답변 템플릿 관리
- 드래프트 저장 및 승인 워크플로

### Phase 5: 대시보드 및 분석 (dashboard-analytics)

- KPI 대시보드: 일별 처리량, 평균 응답 시간, SLA 준수율
- 브랜드별/유형별 문의 트렌드 차트
- 담당자 성과 분석: 처리 건수, 평균 응답 시간
- 이상 탐지: 문의량 급증, 미처리 건수 알림
- CSV/PDF 리포트 내보내기

## 인프라 및 운영 문서

### deployment-guide

- Docker Compose 기반 배포
- Nginx 리버스 프록시 설정
- SSL 인증서 관리 및 자동 갱신
- 환경별(dev/staging/prod) 구성

### operations-guide

- 모니터링: Prometheus + Grafana 메트릭
- 로그 관리: 구조화 로깅, 로그 로테이션
- 장애 대응 절차(Runbook)
- 백업 및 복구 프로세스

### migration-guide & server-migration-guide

- 기존 시스템에서의 데이터 마이그레이션 절차
- 서버 이전 시 무중단 전환 방법
- 데이터 정합성 검증 체크리스트

### database-dump

- DB 스키마 스냅샷 및 시드 데이터
- 테이블 구조: emails, assignments, users, roles, templates 등

## 개발 표준 문서

### dev-framework & parallel-dev-guide

- 기술 스택: Next.js + TypeScript + Prisma + PostgreSQL
- 코드 컨벤션 및 디렉토리 구조
- 병렬 개발 시 브랜치 전략 및 머지 규칙
- CI/CD 파이프라인 구성

### page-feature-specs & workflow-specs

- 각 페이지별 기능 상세 명세 (메일 목록, 상세, 설정 등)
- 워크플로 정의: 메일 수신 -> 분류 -> 배정 -> 응답 -> 완료

### ux-ui-guidelines

- 디자인 시스템: 색상, 타이포그래피, 컴포넌트 가이드
- 반응형 레이아웃 기준
- 접근성(a11y) 요구사항

### shared-infrastructure & data-integration-scenarios

- 공유 인프라: 인증, 로깅, 에러 핸들링 모듈
- 데이터 연동: [[naver-smartstore]] 주문 API, ERP 재고 연동 시나리오

### update-history

- 버전별 변경 이력 기록
- 주요 릴리즈 노트 및 핫픽스 내역

## 관련 페이지

- [[auto-email-overview]] - 기존 프로젝트 요약
- [[keychron]] - 키크론 브랜드
- [[naver-smartstore]] - 네이버 스마트스토어
