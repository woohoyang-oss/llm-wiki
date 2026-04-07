---
title: 키크론 런처 클론 프로젝트
tags: [keychron, launcher, webhid, via-protocol]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# 키크론 런처 클론 프로젝트

## 개요

키크론 런처(Keychron Launcher)의 클론 프로젝트로, 키크론 키보드의 펌웨어 설정을 웹 브라우저에서 직접 제어할 수 있는 도구다. WebHID API와 VIA 프로토콜을 사용하여 키맵 변경, HE(Hall Effect) 모드 설정 등을 지원한다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| 구조 | Monorepo |
| Frontend | Vanilla JS |
| Backend | Fastify (Node.js) |
| DB | SQLite |
| 프로토콜 | WebHID, VIA Protocol |

## 아키텍처

### Monorepo 구성
```
keychron-launcher/
  frontend/    # Vanilla JS 웹 앱
  backend/     # Fastify API 서버
  shared/      # 공유 타입 및 유틸리티
```

### WebHID 통신
- 브라우저의 WebHID API를 통해 USB HID 디바이스와 직접 통신
- VIA 프로토콜 기반 명령어 송수신
- 키맵 읽기/쓰기, 매크로 설정, 조명 제어

## 주요 기능

### 키맵 편집
- 레이어별 키 매핑 변경
- 매크로 설정 및 관리
- 키보드 레이아웃 시각화

### HE (Hall Effect) 모드
키크론의 홀 이펙트 스위치 키보드를 위한 고급 설정:
- **Actuation Point** — 키 입력 인식 깊이 조절
- **Rapid Trigger** — 키 릴리스 시점 기반 빠른 재입력
- **SOCD (Simultaneous Opposing Cardinal Directions)** — 좌우/상하 동시 입력 처리 방식

### VIA 프로토콜
- VIA 호환 키보드 자동 감지
- JSON 기반 키보드 정의 파일 로드
- 실시간 키맵 업데이트

## 역할별 CLAUDE.md

프로젝트 내 AI 페어 프로그래밍을 위해 역할별 지침 파일이 존재한다:
- **Analyst** — 요구사항 분석 및 스펙 작성
- **Backend** — Fastify API 및 SQLite 스키마 개발
- **Frontend** — UI 컴포넌트 및 WebHID 연동 개발
- **QA** — 테스트 시나리오 작성 및 검증

## 관련 시스템

- [[keychron]] — 키크론 브랜드 정보
- [[tbnws-admin]] — 제품 정보 관리 (ERP)
