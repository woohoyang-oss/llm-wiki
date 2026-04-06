---
title: Auto-DM
tags: [자동화, DM, 인스타그램, 챗봇, GSD-2]
created: 2026-04-06
updated: 2026-04-06
sources: [auto-dm-AGENTS.md, auto-dm-MEMORY.md, auto-dm-gsd2-methodology.md, plan-frolicking-whistling-bubble.md, plan-frolicking-whistling-bubble-agent-a6d8333528682b720.md]
---
# Auto-DM

## 개요

Auto-DM은 ManyChat을 대체하는 자체 DM 자동응답 서비스다. 브랜드별(키크론, 지티기어, 포스윙 등) 봇을 운영하며, 채널별(인스타그램, 텔레그램, 웹챗, 카카오) 확장이 가능한 구조로 설계되었다. 1단계 MVP는 **키크론 인스타그램 DM 자동응답**이다.

## 서비스 아키텍처

웹 대시보드(봇 관리, 시나리오 편집, 대화 모니터링, 통계) 아래에 Core Server가 위치하며, Bot Manager, Scenario Engine, Conversation Manager, LLM Router, Trigger Handler, Data Export(Google Sheets) 모듈로 구성된다. 하단에 Instagram/Telegram/Kakao 채널 어댑터가 플러그인 형태로 연결된다.

## 핵심 도메인 모델

- **Brand**: 브랜드별 독립 설정(톤앤매너, 지식베이스, 시나리오)
- **Bot**: 역할 지정(`simple_reply`, `cs`, `product_intro`, `event`, `guide`), 봇별 LLM 모델 선택
- **Scenario**: 트리거 조건 → 응답 로직 → 데이터 수집 → 후속 액션
- **Conversation**: 사용자별 대화 세션 추적 및 상태 관리

## 트리거 시스템

Instagram Graph API Webhook 기반으로 6종 트리거를 지원한다: `keyword`(키워드 매칭), `story_mention`, `story_reply`, `comment`(게시물 댓글), `any_dm`, `first_dm`(신규 고객 환영).

## 기술 스택

Next.js(App Router) + TypeScript + PostgreSQL(Supabase) + Google Sheets API + Claude/GPT(LLM) + Instagram Graph API + Vercel 배포.

## GSD-2 방법론

개발에 [[GSD-2]](Get Shit Done) 자율 에이전트 방법론을 적용한다. Milestone → Slice → Task 3계층 작업 구조와 Plan → Execute → Complete → Reassess → Validate 5단계 파이프라인을 따르며, 각 Task는 반드시 1개 컨텍스트 윈도우 내에서 완료 가능해야 한다(Iron Rule). KNOWLEDGE.md를 통해 프로젝트 교훈을 컨텍스트 경계를 넘어 보존한다.

## 유사 서비스 리서치 결과

15개 이상의 오픈소스 프로젝트를 조사한 결과:

- **Tier 1 (가장 유사)**: FUMA, SAAS-Instagram-DM-Automations, Sudershhh/saas-dm-automations — 공식 Graph API 사용, Next.js 스택
- **Tier 2 (대형 플랫폼)**: Chatwoot(27K stars), Botpress(14K), Hexabot, Typebot, Rasa, Tiledesk
- **Tier 3 (워크플로)**: n8n(182K stars) — Instagram 노드 보유

핵심 인사이트: 공식 Graph API를 사용하는 인스타그램 DM 자동화 오픈소스는 아직 소규모(50 stars 미만)이며, ManyChat 대안 오픈소스 시장은 사실상 비어 있다. Auto-DM이 이 공간의 선도 프로젝트가 될 기회가 있다.

## 관련 소스

- `auto-dm-AGENTS.md` — 에이전트 설정 (Next.js 규칙)
- `auto-dm-MEMORY.md` — GSD-2 방법론 참조 링크
- `auto-dm-gsd2-methodology.md` — GSD-2 방법론 상세
- `plan-frolicking-whistling-bubble.md` — 서비스 기획서 (아키텍처, MVP 범위)
- `plan-frolicking-whistling-bubble-agent-a6d8333528682b720.md` — 유사 서비스 리서치 결과
