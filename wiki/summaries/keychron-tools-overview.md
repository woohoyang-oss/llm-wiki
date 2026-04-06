---
title: 키크론 관련 도구 모음 요약
tags: [키크론, 발주추적, 리뷰크롤러, 대시보드, kychr.space, 도구]
created: 2026-04-06
updated: 2026-04-06
sources: [keychron-supply-tracker-README-KR.md, keychron-supply-tracker-README-EN.md, PI-업데이트-가이드.md, naver-brandstore-review-crawler-ex75.md, kychr-space-project.md, kychr-space-CLAUDE.md]
---
# 키크론 관련 도구 모음 요약

[[ToBe Networks]]에서 [[Keychron]] 사업 운영을 위해 구축한 내부 도구 3종과 관련 가이드를 정리한다.

---

## 도구 목록

| 도구 | 목적 | 기술 스택 | 상태 |
|------|------|-----------|------|
| [[keychron-supply-tracker]] | PO(발주) 상태 추적 및 지연 감지 | Python + FastAPI + PostgreSQL | 기획 완료 |
| [[naver-review-crawler]] | 네이버 브랜드스토어 리뷰 수집 | Python + requests + openpyxl | 운영 중 |
| [[kychr.space]] | 시장 인텔리전스 대시보드 | Astro 5.9 SSR + React + Tailwind | 운영 중 |

---

## 1. Keychron Supply Tracker (발주 추적)

키크론 본사의 PO 상태를 폴링 방식으로 동기화하는 시스템이다. 모델 번호 단위로 ETP/ETD 변동을 감지하고, 지연 시 Slack/KakaoTalk으로 알림을 발송한다.

**핵심 기능:**
- 4시간 주기 폴링으로 본사 API에서 PO 상태 수집
- ETP/ETD 3일 이상 지연 시 자동 경고
- 동일 PO 내 모델 간 일정 격차 10일 이상 시 부분 출하 제안
- 변경 이력 전체 기록 (change_log)

**MVP 범위:** API 클라이언트 -> Diff 엔진 -> Slack 알림 -> 웹 대시보드 (4주 계획)

---

## 2. PI 업데이트 가이드

발주/납기/단가 이슈를 **PI(Proforma Invoice) 번호 단위**로 추적하는 운영 가이드다. 상태값은 4단계(완료/진행중/대기/이슈)로 통일하며, 브랜드별(Keychron, MOZA, Playseat, Aiper, TrackRacer) PI 미수령 건을 별도 관리한다. 일일 보고와 주간 리뷰에서 동일 포맷을 사용한다.

---

## 3. 네이버 브랜드스토어 리뷰 크롤러

[[Keychron eX75 HE 8K TMR]] 리뷰를 네이버 브랜드스토어 API에서 자동 수집하는 Python 스크립트다. merchantNo 자동 감지, API 엔드포인트 3종 자동 시도, Excel/JSON 출력을 지원한다. PRODUCT_NO와 BRAND_STORE_ID만 변경하면 다른 상품에도 적용 가능하다.

---

## 4. kychr.space (시장 인텔리전스 대시보드)

글로벌 디스트리뷰터가 시장 데이터를 업로드하면 Claude CLI가 자동 분석하여 영문 요약/태그/핵심 발견을 생성하는 대시보드다.

**주요 특징:**
- EC2 t3.micro(서울 리전) + nginx + pm2 배포
- 이중 인증: 쿠키(브라우저) + Bearer 토큰(API)
- 업로드 파이프라인: 파일 저장 -> Claude 분석(.analysis.md) -> index.json 업데이트
- 챗봇이 모든 .analysis.md를 지식 베이스로 활용
- 뉴스페이퍼 스타일 대시보드 레이아웃 (InsightCard + 페이지네이션)

**디자인 시스템:** 라이트 웜 배경(#fcfcf9), rounded-[1.5rem] 카드, Tailwind CSS, 다크 모드 미지원
