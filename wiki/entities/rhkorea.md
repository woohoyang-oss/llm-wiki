---
title: RH Korea 웹사이트 리부트
tags: [커뮤니티, 웹개발, Next.js, Flask, AI]
created: 2026-04-06
updated: 2026-04-06
sources: [rhkorea-web-AGENTS.md, plan-compiled-soaring-iverson.md, plan-happy-tumbling-crown.md]
---
# RH Korea 웹사이트 리부트

## 개요

rhkorea.com은 1997년부터 29년간 이어온 한국 브릿팝(Radiohead Korea) 커뮤니티다. 통합 DB에 27,868개 글과 70,111개 댓글이 축적되어 있으며, 현재 Flask + MySQL 뷰어로 동작 중이다. 이 리부트 프로젝트는 읽기 전용 아카이브를 실제 커뮤니티로 되살리는 것을 목표로 한다.

## 기술 스택 전환

초기 계획은 Flask 위에 기능을 추가하는 방식이었으나, 이후 **Next.js 15**(App Router) + TypeScript + Tailwind CSS v4 + MySQL(기존 DB 유지) 구조로 전환되었다. 배포 대상은 Vercel이며, 다크 모드를 기본으로 채택했다.

## 핵심 기능 (6개 Phase)

### Phase 1-2: 인증 시스템 (3층 구조)
- **1층 — 익명**: DC식 닉네임 + 비밀번호(bcrypt). 가입 불필요
- **2층 — 이메일 가입**: [[akaxa-space]] OTP 인증 연동. JWT 세션
- **3층 — 아카이브 클레임**: 기존 글의 이메일 매칭으로 소유권 주장. 관리자 승인 구조

### Phase 3: 글쓰기 + 댓글
새 글은 `source='user'`로 구분되며, 아카이브 글에도 새 댓글 가능. 소프트 삭제 방식 채택.

### Phase 4: YouTube 자동 임베드
본문 내 YouTube URL을 감지하여 oEmbed iframe으로 자동 변환.

### Phase 5: AI BGM 추천
글 등록 직후 Claude CLI를 비동기 호출하여 글의 무드를 분석하고, AI 큐레이터 "아레치"가 어울리는 곡을 추천 댓글로 자동 등록한다. `source='ai'`, `is_ai=1`로 표기.

### Phase 6: UI 업데이트
시대 배지(CWB=핑크, XE=블루, Rb=그린, NEW=화이트)로 아카이브와 새 글을 자연스럽게 구분. Pretendard 폰트, 미니멀 디자인.

## DB 확장

기존 `posts`/`comments` 테이블에 `is_archive`, `user_id`, `author_password` 컬럼을 추가하고, `users`, `archive_claims`, `shared_tracks` 테이블을 신규 생성한다. 기존 27,868건은 `is_archive=TRUE`로 마킹.

## 구현 순서

1. Next.js 프로젝트 초기화 + MySQL 커넥션 풀
2. DB 스키마 변경
3. 게시판 목록 + 글 보기 (읽기 전용)
4. 익명 글쓰기/댓글
5. 이메일 OTP 로그인
6. YouTube 임베드 + 플레이리스트
7. 아카이브 클레임

## 관련 소스

- `rhkorea-web-AGENTS.md` — 에이전트 설정 (Next.js 규칙)
- `plan-compiled-soaring-iverson.md` — 초기 Flask 기반 구축 계획 (6 Phase)
- `plan-happy-tumbling-crown.md` — Next.js 전환 후 Phase 1 구현 계획
