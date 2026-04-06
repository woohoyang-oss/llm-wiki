# 메일 검색 분석 결과

## 1. 공용 메일함 검색 (list_emails)
- 방식: ILIKE (대소문자 무시 LIKE)
- 패턴: %keyword% 부분 일치
- 검색 대상: subject, from_address, from_name, body_text
- 다중 키워드: 공백 분리 → 각 키워드별 OR(4컬럼) → 키워드간 AND
- 디바운스: 300ms

## 2. 내 메일함 검색 (personal-emails)
- 방식: Gmail API 직접 전달
- 단순 검색어 → prefix 매칭 (kw*)
- 고급 검색 → Gmail 문법 그대로
- 디바운스: 500ms

## 3. 공용 vs 내메일함 검색 차이
| 항목 | 공용 메일함 | 내 메일함 |
|------|-----------|----------|
| 검색 엔진 | PostgreSQL ILIKE | Gmail API |
| 페이지네이션 | offset/limit | page_token |

## 4. 개선 포인트
- 높음: 전문검색 부재, 하이라이팅 없음, 고급 검색 미지원
- 중간: 디바운스 불일치, 검색 히스토리 미지원
- 낮음: body_text 성능, to_address 검색 미지원
