---
title: 사방넷 자동화 대시보드
tags: [sabang, e-commerce, automation, flask]
created: 2026-04-06
updated: 2026-04-06
sources: [claude--sabang--*.md, sabang--docs--*.md]
---

# 사방넷 자동화 대시보드

## 개요

사방넷(sabangnet) 자동화 대시보드는 20개 쇼핑몰의 상품 상태를 자동으로 동기화하고 관리하는 내부 도구다. Flask + SQLite 기반으로 경량하게 운영되며, Google Sheets 및 Scrapling을 연동하여 상품 데이터를 수집한다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| Backend | Flask (Python) |
| DB | SQLite |
| Frontend | 6-tab UI |
| 연동 | Google Sheets API, Scrapling |
| Port | 5050 |

## 주요 기능

### 상품 상태 동기화
- 20개 쇼핑몰의 상품 등록/수정/품절 상태를 일괄 관리
- 사방넷 API를 통한 자동 동기화

### 6-Tab UI 구성
대시보드는 6개 탭으로 구성되어 각 업무 영역을 분리한다:
1. 상품 현황 — 전체 상품 목록 및 상태
2. 동기화 관리 — 쇼핑몰별 동기화 상태
3. 가격 관리 — 채널별 가격 설정
4. 재고 연동 — 재고 수량 동기화
5. 주문 현황 — 주문 데이터 조회
6. 설정 — 쇼핑몰 연동 설정

### Google Sheets 연동
- 상품 마스터 데이터를 Google Sheets에서 관리
- Sheets API를 통한 실시간 읽기/쓰기

### Scrapling 연동
- 쇼핑몰 페이지에서 상품 정보를 스크래핑
- 가격, 재고, 상태 변경 감지

## 대상 쇼핑몰

사방넷을 통해 연동되는 주요 채널:
- 네이버 스마트스토어
- 지마켓 / 옥션
- 쿠팡
- 29CM
- 카카오 선물하기
- 기타 종합몰/오픈마켓

## 관련 시스템

- [[tbnws-admin]] — ERP 주문/재고 데이터 연동
- [[keychron]] — 키크론 상품 관리
- [[gtgear]] — 지티기어 상품 관리
