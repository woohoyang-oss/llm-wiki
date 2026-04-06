# B3 via 포워딩 메일 오판 — 상세 분석

## 데이터
- via 메일 총 165건 (gtgear 137건, keychron 28건)
- 110건이 received 상태로 방치 (고객 A/S 요청 포함)

## 원인
Gmail 포워딩 시 From 헤더 재작성:
원본: 윤원호 <customer@naver.com>
포워딩 후: 윤원호 via support@gtgear.co.kr <gtgear@tbnws.com>

pipeline이 gtgear@tbnws.com → tbnws.com → 내부 → outbound 스킵

## via 패턴 분류
- 고객 문의 (AI 처리 대상): 개인 이름 via support@gtgear.co.kr
- 시스템 알림 (ignored): 스마트스토어, 네이버페이, Steam, Matrixify

## 수정 방향
1. from_name에 via가 있으면 외부 경유로 간주 (is_from_external=True)
2. via 앞 이름이 시스템명이면 ignored, 개인이면 AI 처리
3. X-Original-Sender / Reply-To에서 진짜 발신자 복원
