---
title: 네이버 리뷰 알림 자동화
tags: [review, naver, automation, email, cs]
created: 2026-04-09
updated: 2026-04-09
sources: []
---

# 네이버 리뷰 알림 자동화

## 개요
네이버 스마트스토어 1~3점 저평점 리뷰를 자동 감시하여:
1. **미답변 리뷰**를 긴급도별로 분류 (urgent/warning/fresh)
2. **문제 상품 집중도**를 파악 (같은 상품 반복 저평점)
3. 액션 필요 건을 **이메일로 CS팀에 발송**

## 데이터 소스
- DB: `TBNWS_ADMIN.naver_reviews`
- 원본 수집: tbnws-admin 크롤러 → `naver_store_review_crawler` / `naver_store_review_item`
- 판매자 답변 판단: `replyContent IS NULL` → 미답변

## eywa MCP 도구 (4개)

| 도구 | 설명 |
|------|------|
| `get_low_rating_reviews` | 저평점(1~3점) 리뷰 목록 + 답변 유무 |
| `get_unresolved_reviews` | 미답변 저평점 리뷰 (urgent/warning/fresh 분류) |
| `get_low_rating_summary` | 브랜드별 저평점 요약 통계 |
| `get_product_review_hotspot` | 상품별 저평점 집중도 |

## 워크플로우
1. `get_low_rating_summary` → 전체 현황 파악
2. `get_unresolved_reviews` → 긴급 미답변 리스트업
3. `get_product_review_hotspot` → 문제 상품 식별
4. 이메일 발송 제안 (urgent 1건 이상 시)

## 연관 문서
- [[tobe-mcp]] — MCP 도구 등록
- [[auto-email]] — 이메일 자동화 시스템
- 가이드: `review-alert`
