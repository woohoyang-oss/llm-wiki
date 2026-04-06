---
title: 메일박스 아키텍처
tags: [mailbox, gmail, forwarding, send-as, mail-resolver]
created: 2026-04-06
updated: 2026-04-06
sources: [메일박스-구조-분석--테스트-시나리오.md, 메일박스-운영-방침-확정.md, b3-분석---via-포워딩-메일-오판-상세.md]
---
# 메일박스 아키텍처

## 개요

[[auto-email]] 시스템의 메일 수신/발송 구조이다. 단일 통합 수신함(ai@tbe.kr)으로 모든 브랜드 메일을 집약하고, Gmail의 포워딩과 Send As 기능을 조합하여 멀티 브랜드 메일 처리를 구현한다.

## 포워딩 + Send As 구조

```
[브랜드 Gmail] → 포워딩 → [ai@tbe.kr 통합 수신함]
                              ↓
                    MailboxResolver (X-Forwarded-To 헤더 분석)
                              ↓
                    DB 조회 → 브랜드/카테고리 매칭
                              ↓
                    AI 처리 → Send As로 브랜드 주소 발송
```

### 수신 흐름
각 브랜드 Gmail에서 ai@tbe.kr로 포워딩을 설정한다. 수신된 메일의 `X-Forwarded-To` 헤더로 원래 수신 주소를 감지하고, `MailboxResolver`가 DB에서 동적으로 브랜드/카테고리를 매칭한다. 하드코딩 없이 DB 기반으로 동작하므로 코드 변경 없이 새 브랜드 추가가 가능하다.

### 발송 흐름
Gmail의 "다른 주소에서 메일 보내기(Send As)" 기능으로 원래 브랜드 주소로 회신한다. `GmailService`가 Send As 설정을 활용하여 발송하고, `pipeline.py`가 MailboxResolver → AI → 발송 전체 흐름을 자동 처리한다.

## via 패턴 문제

Gmail 포워딩 시 From 헤더가 재작성되는 문제가 발견되었다. 원본 `customer@naver.com`이 포워딩 후 `고객명 via support@gtgear.co.kr`로 변환되어, pipeline이 이를 내부 메일로 오판하고 처리를 스킵하는 버그가 발생했다. via 메일 총 165건(gtgear 137건, keychron 28건) 중 110건이 미처리 상태로 방치되었다.

### 해결 방향
1. `from_name`에 "via"가 포함되면 외부 경유 메일로 간주
2. via 앞 이름이 시스템명(스마트스토어, 네이버페이 등)이면 ignored, 개인명이면 AI 처리
3. `X-Original-Sender` / `Reply-To` 헤더에서 실제 발신자 복원

## 새 브랜드 추가 절차

1. mail.tbe.kr 관리자에서 메일박스 추가 (brand/category/address 입력)
2. 새 브랜드 Gmail에서 ai@tbe.kr로 포워딩 설정
3. ai@tbe.kr Gmail에서 Send As로 새 브랜드 주소 추가
4. MailboxResolver가 자동 매칭 시작

OAuth 직접 연동은 브랜드 10개 이상 시 재검토 예정이며, 현재는 포워딩+Send As 방식을 유지한다.

## 관련 소스
- [메일박스-구조-분석--테스트-시나리오.md](../../raw/메일박스-구조-분석--테스트-시나리오.md)
- [메일박스-운영-방침-확정.md](../../raw/메일박스-운영-방침-확정.md)
- [b3-분석---via-포워딩-메일-오판-상세.md](../../raw/b3-분석---via-포워딩-메일-오판-상세.md)
