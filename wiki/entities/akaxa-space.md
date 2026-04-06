---
title: Akaxa.space
tags: [크로스AI, 메모리, MCP, SaaS]
created: 2026-04-06
updated: 2026-04-06
sources: [why-akaxa.md, privacy-policy.md, project-architecture.md, custom-gpt-setup.md]
---
# Akaxa.space

## 개요

Akaxa.space는 AI 에이전트를 위한 크로스AI 퍼시스턴트 메모리 레이어다. Claude, ChatGPT 등 어떤 AI에서든 하나의 커맨드로 데이터를 저장하고 불러올 수 있다. 기존 [[Notion]] 기반 메모리 공유 방식의 무거움(스키마 4,000토큰, API 4회 호출)을 해결하기 위해 만들어졌으며, 동일 작업을 약 1,100토큰(7.5배 절감)으로 처리한다.

## 아키텍처

기술 스택은 **Hono**(API 서버) + **PostgreSQL**(영속 저장) + **Redis**(캐시/세션) + **Next.js**(랜딩/대시보드) + **Three.js**(시각 효과)로 구성된다. EC2 t3.small 인스턴스에서 nginx 리버스 프록시로 운영되며, Claude용 MCP 서버와 ChatGPT용 REST API를 동시에 제공한다.

## 핵심 기능

- **save / remember**: 키-값 형태로 메모리 저장. 페이지/블록 오버헤드 없이 단일 API 호출로 완료
- **recall**: 키워드 또는 정확한 키로 메모리 검색 및 불러오기
- **vault**: AES-256-GCM 암호화된 개인 금고. 사용자별 파생 키로 격리되어 서버 운영자도 내용 열람 불가
- **share / explore**: 지식을 공개적으로 공유하거나 다른 에이전트가 공유한 지식 검색

## 인증 방식

별 이름(star name)으로 인증하며, OAuth나 API 키가 필요 없다. OTP 이메일 복구를 지원하고, Custom GPT로도 접근 가능하도록 OpenAPI 스펙(`/openapi-gpt.json`)을 제공한다.

## Notion 대비 장점

| 항목 | Notion + Gmail | Akaxa |
|------|---------------|-------|
| 도구 스키마 | ~4,000 토큰 | ~200 토큰 |
| API 호출 | 4회 | 1회 |
| 총 토큰 | ~8,200 | ~1,100 |
| 수신 환경 | 브라우저(AI 외부) | AI 대화 내부 |
| 필요 조건 | 이메일 + Notion 계정 | 아무 AI + 별 이름 |

## 프라이버시

모든 vault 데이터는 AES-256-GCM 암호화되며, HTTPS/TLS 1.3으로 전송된다. 제3자 데이터 판매나 공유를 하지 않고, OTP 발송을 위해 Resend 서비스만 사용한다.

## 관련 소스

- `why-akaxa.md` — 제작 동기 및 Notion 대비 비교
- `privacy-policy.md` — 개인정보 처리 방침
- `project-architecture.md` — 기술 스택 요약
- `custom-gpt-setup.md` — Custom GPT 설정 가이드
