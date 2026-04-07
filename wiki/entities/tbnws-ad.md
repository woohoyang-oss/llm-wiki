---
title: TBNWS 네이버 광고 분석 대시보드
tags: [tbnws-ad, advertising, dashboard, flask]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# TBNWS 네이버 광고 분석 대시보드

## 개요

TBNWS AD는 네이버 SA(검색광고)와 GFA(성과형 디스플레이 광고), 스마트스토어 매출 데이터를 통합 분석하는 내부 대시보드다. Flask + Vanilla JS 기반으로 구축되었으며, [[tobe-mcp]]와 연동하여 MCP 도구로도 데이터를 조회할 수 있다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| Backend | Flask (Python), Blueprint 구조 |
| Frontend | Vanilla JS, Chart.js |
| DB | MySQL (TBNWS_ADMIN / TBNWS_AD) |
| 인프라 | AWS EC2 13.125.219.231 |
| 인증 | Google SSO |
| 스케줄링 | Cron 기반 자동 수집 |

## 접속 정보

- URL: `tbe.kr:8087`
- EC2: `13.125.219.231`

## 주요 기능

### SA (검색광고) 분석
- 캠페인/광고그룹/키워드별 성과 조회
- ROAS 상위 키워드, 낭비 키워드 자동 감지
- 일별 광고비/전환매출 추이

### GFA (성과형 디스플레이) 분석
- 캠페인별 광고비, 전환매출, ROAS
- 퍼널 분석 (TOF/MOF/BOF/ADVoost)
- 실시간 KPI 모니터링 (시간별 누적 추이)
- 소재 피로도 감지 및 낭비 소재 리포트
- AI 예산 재배분 제안 ([[gfa-ops-ai]] 연동)

### 스마트스토어 통합
- 광고비 vs 매출 상관관계 분석
- 비즈머니 잔액 및 잔여일수 예측

## 아키텍처

### Blueprint 구조
25개 이상의 Blueprint route로 모듈화되어 있다:
- SA 관련: 캠페인, 키워드, 일별 추이
- GFA 관련: 캠페인, 퍼널, 실시간, OPS AI
- 공통: 매출, 비즈머니, 광고 개요

### 데이터 파이프라인
1. Cron 스케줄러가 네이버 광고 API에서 데이터 수집
2. MySQL TBNWS_AD 데이터베이스에 적재
3. Flask API를 통해 대시보드 및 MCP에 서빙

### DB 스키마
- `TBNWS_ADMIN`: 관리 플랫폼 공용 DB
- `TBNWS_AD`: 광고 전용 DB (SA/GFA 캠페인, 키워드, 일별 데이터)

## 관련 시스템

- [[tobe-mcp]] — MCP 서버를 통한 광고 데이터 조회
- [[kychr-space]] — 키크론 스페이스 연동
- [[gfa-ops-ai]] — GFA 자동 최적화 AI 엔진
- [[tbnws-admin]] — 통합 관리 플랫폼 (공용 DB 공유)
