---
title: Keychron Supply Tracker - 발주 추적 시스템
tags: [keychron, 발주추적, supply-chain, PI, FastAPI]
created: 2026-04-06
updated: 2026-04-06
sources: [keychron-supply-tracker-README-KR.md, keychron-supply-tracker-README-EN.md, PI-업데이트-가이드.md]
---
# Keychron Supply Tracker - 발주 추적 시스템

## 개요

Keychron Supply Tracker는 Keychron 본사의 발주(PO) 상태를 지역 파트너(ToBe Networks, 한국)에게 실시간으로 동기화하는 폴링 기반 추적 시스템이다. 모델 번호 단위로 생산/출하/도착 일정을 추적하며, 납기 지연이나 부분 출하 상황을 자동으로 감지하여 알림을 발송한다.

GitHub: github.com/woohoyang-oss/keychron-supply-tracker (MIT)

## 해결하려는 문제

현재 Keychron 본사 직원이 매일 수동으로 PO 상태를 업데이트하지만, 지역 파트너에게 자동으로 전달되는 수단이 없다. 이로 인해 납기 지연(ETP/ETD) 발견이 늦고, 재고 결정이 사후적이며, 양측 모두 수동 팔로업에 시간을 소모한다.

## 시스템 아키텍처

```
Keychron HQ PO System -> REST API
        |
        | HTTPS (폴링, 기본 4시간 주기)
        v
Partner Polling Service (ToBe)
  Sync Engine: Fetch -> Diff -> Store -> Alert
        |
        v
  PostgreSQL (po_headers, po_line_items, change_log)
        |
        v
  알림: Slack / KakaoTalk / Notion DB / Web Dashboard
```

## 핵심 기능

### PO 상태 추적
PO 헤더(발주 번호, 날짜, 전체 상태)와 라인 아이템(모델별 수량, 생산 상태, ETP/ETD/ETA)을 모델 번호 단위로 추적한다. 상태 흐름: Draft -> Confirmed -> Waiting Parts -> In Production -> QC -> Packed -> Shipped -> Delivered.

### 자동 알림
- **상태 변경**: 모든 상태 변경 시 알림
- **납기 지연**: ETP 또는 ETD가 3일 이상 밀릴 경우 DELAY ALERT
- **미갱신**: 3일 이상 업데이트 없는 PO에 STALE ALERT
- **부분 출하 제안**: 같은 PO 내 모델 간 ETP 차이가 10일 이상이면 선출하 제안

### 변경 이력 (Audit Trail)
모든 필드 변경을 change_log 테이블에 기록하여 완전한 이력 추적이 가능하다.

## PI(Proforma Invoice) 관리

PI 번호 단위로 이슈를 추적하는 별도 운영 가이드가 존재한다.

- **상태 4단계**: 완료 / 진행중 / 대기 / 이슈
- **필수 항목**: PI 번호, 브랜드, 모델명, 이슈 유형(발주수량/단가/납기/출하), 상태, 담당자, 예상 완료일
- **PI 미수령 건**: 모델명으로 임시 키를 사용하다가 PI 수령 시 소급 반영
- **주간 리뷰**: 진행중 건 경과 확인, 1주 이상 미수령 건 에스컬레이션, 완료 건 아카이빙

## 기술 스택

- Python 3.11+ / FastAPI
- APScheduler (폴링 스케줄링)
- PostgreSQL
- Slack Webhook, Notion API, KakaoTalk
- Docker on Tailscale 네트워크

## MVP 로드맵

- Phase 1 (1~2주): API 클라이언트, 스케줄러, DB 스키마, Diff 엔진, 변경 로그
- Phase 2 (2~3주): Slack 알림, 지연 감지, 부분 출하 제안
- Phase 3 (3~4주): 웹 대시보드, Notion 동기화, Gantt 타임라인

## 관련 시스템

- [[kychr-space]] - 키크론 마켓 인텔리전스 대시보드

## 관련 소스

- [keychron-supply-tracker-README-KR.md](../../raw/keychron-supply-tracker-README-KR.md)
- [keychron-supply-tracker-README-EN.md](../../raw/keychron-supply-tracker-README-EN.md)
- [PI-업데이트-가이드.md](../../raw/PI-업데이트-가이드.md)
