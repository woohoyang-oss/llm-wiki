---
title: 텔레그램 연동 패턴
tags: [telegram, integration, automation]
created: 2026-04-06
updated: 2026-04-06
sources: []
---

# 텔레그램 연동 패턴

[[keychron]] 업무 자동화에서 텔레그램 Bot API를 활용하여 알림, 명령, 대화형 인터페이스를 구현하는 패턴.

## Long Polling with Offset Tracking

텔레그램 Bot API에서 메시지를 수신하는 기본 방식.

### 동작 원리

```
1. getUpdates?offset={last_offset}&timeout=N 호출
2. 새 메시지가 있으면 응답 수신
3. 수신한 update_id + 1을 새 offset으로 저장
4. 처리 완료 후 1번으로 돌아감
```

### Offset 관리

- Offset은 파일(`~/.tg_offset`)에 영속 저장
- 세션 재시작 시 마지막 offset부터 이어서 폴링
- Offset 미지정 시 최근 100개 메시지를 다시 수신 (중복 처리 위험)

### Timeout 설정

- `timeout=5` ~ `timeout=30` 사이 권장
- 긴 timeout: 서버 부하 감소, 응답 지연 증가
- 짧은 timeout: 즉시 응답, API 호출 빈도 증가

### 충돌 방지

동일 봇 토큰으로 여러 프로세스가 동시에 `getUpdates`를 호출하면 409 Conflict 발생. 하나의 봇은 하나의 폴링 프로세스만 유지해야 한다.

## Claude Code x Telegram

Claude Code 세션에서 텔레그램을 통해 사용자와 상호작용하는 패턴.

### 세션 시작 플로우

1. Claude Code 세션 시작
2. 텔레그램 봇 연결 (BOT_TOKEN, CHAT_ID 설정)
3. 연결 알림 발송
4. Long Polling 루프 시작
5. 수신 메시지를 Claude가 분석·처리·응답

### 메시지 처리 파이프라인

```
수신 메시지
  → 명령어 필터링 (/로 시작하면 무시)
  → 의도 분석 (MCP 도구 필요 여부 판단)
  → 도구 실행 (MCP, Bash 등)
  → 응답 생성
  → sendMessage API로 발송
  → 4096자 초과 시 분할 발송
```

### 응답 분할 규칙

텔레그램 메시지 길이 제한은 4096자. 이를 초과하는 응답은:

- 논리적 단위(섹션, 테이블)로 분할
- 각 청크를 순차적으로 sendMessage 호출
- 마크다운 포맷 유지 (parse_mode: Markdown)

## 활용 사례

### GFA-Auto (광고 자동화)

- GFA 실시간 알림 수신 (예산 과다소진, ROAS 하락 등)
- 캠페인 예산 조정 승인 요청
- 일일 성과 리포트 자동 발송
- [[gfa-funnel-framework|퍼널 건강도]] 이상 감지 알림

### 업무비서

- 일정 알림, 회의록 요약 전달
- 재고 부족 경고 ([[product-code-hierarchy|G-code]] 기반)
- CS 미답변 문의 알림
- 발주 필요 상품 리스트 전달

### CS 에스컬레이션

- [[confidence-routing|Confidence Routing]]에서 에스컬레이션된 문의를 텔레그램으로 알림
- 담당자가 텔레그램에서 즉시 확인·지시 가능
- 승인 큐 항목에 대한 승인/반려 처리

## Webhook vs Long Polling

| 방식 | 장점 | 단점 |
|------|------|------|
| **Long Polling** | 서버 불필요, 방화벽 무관 | 지연, 봇당 1프로세스 |
| **Webhook** | 즉시 수신, 효율적 | HTTPS 서버 필요, SSL 인증서 |

현재 Claude Code 연동에서는 **Long Polling**을 사용한다. 별도 서버 없이 로컬 세션에서 바로 실행 가능하기 때문이다.

## 보안 고려사항

- BOT_TOKEN은 환경변수 또는 로컬 파일로 관리 (코드에 하드코딩 금지)
- CHAT_ID를 화이트리스트로 관리하여 무단 접근 차단
- 민감 정보(주문번호, 개인정보)는 텔레그램 메시지에 마스킹 처리
- 봇 명령어를 통한 위험 작업은 이중 확인 필요

## 관련 개념

- [[gfa-funnel-framework]] — GFA 알림 연동
- [[confidence-routing]] — CS 에스컬레이션 알림
- [[sales-data-pipeline]] — 매출 이상 감지 알림
- [[product-code-hierarchy]] — 재고 알림 기반 코드
