# GFA-Auto Telegram Bot 구현 매뉴얼

## 개요
Claude Code CLI 세션에서 Telegram Bot Long Polling을 통해 GFA(네이버 성과형 디스플레이 광고) 데이터를 조회하고, EC2 OPS AI를 통해 광고비를 원격으로 변경할 수 있는 시스템입니다.

---

## 1. GitHub 레포 클론

```bash
git clone https://github.com/woohoyang-oss/gfa-auto.git
cd gfa-auto
```

---

## 2. 사전 준비

### 2.1 Telegram Bot 생성
1. Telegram에서 `@BotFather` 검색
2. `/newbot` 명령어 전송
3. Bot 이름 및 username 설정
4. **BOT_TOKEN** 발급

### 2.2 Chat ID 확인
1. 봇에게 아무 메시지 전송
2. 브라우저에서 `https://api.telegram.org/bot{BOT_TOKEN}/getUpdates` 접속
3. `result[0].message.chat.id` 값이 **CHAT_ID**

### 2.3 Webhook 삭제 (중요)
기존 webhook이 있으면 getUpdates가 빈 결과를 반환합니다:
```bash
curl https://api.telegram.org/bot{BOT_TOKEN}/deleteWebhook
```

### 2.4 Offset 파일 생성
```bash
echo "0" > ~/.tg_offset
```

---

## 3. Python 환경 설정

```bash
python3 -m venv venv
source venv/bin/activate
pip install playwright requests
playwright install chromium
```

---

## 4. 텔레그램 폴링 스크립트

### 4.1 메시지 수신 (`tg_poll.py`)
- 표준 라이브러리만 사용 (urllib)
- Long Polling 30초 대기
- 그룹 채팅 + 개인 채팅 모두 지원
- 출력 형식: `chat_id|chat_type|text`
- 409 Conflict 에러 자동 재시도 (3회)

```python
# 핵심 파라미터
BOT = '{BOT_TOKEN}'
OFFSET_FILE = '~/.tg_offset'
ALLOWED_CHAT_IDS = [8188602191]  # 허용된 사용자
```

### 4.2 메시지 전송 (`tg_send.py`)
- 4096자 제한 자동 분할
- chat_id 동적 지원 (그룹/개인)

```bash
# 개인 채팅
python3 tg_send.py "메시지"

# 그룹 채팅
python3 tg_send.py "메시지" "-5109506868"
```

---

## 5. GFA 세션 관리

### 5.1 Playwright 로그인 (`login_wait.py`)
1. Chromium 브라우저 실행 (persistent context)
2. 네이버 로그인 페이지 자동 이동
3. 사용자가 로그인 완료하면 자동 감지
4. GFA 대시보드로 이동 → 쿠키 저장

```bash
source venv/bin/activate
python3 login_wait.py
```

### 5.2 저장되는 파일
- `session.json` — 전체 쿠키 (Playwright 형식)
- `api_session.json` — API 호출용 (cookie_header, XSRF-TOKEN, JSESSIONID)
- `chrome-profile/` — Playwright persistent context

### 5.3 필수 쿠키
| 쿠키 | 용도 |
|------|------|
| NID_AUT | 네이버 인증 |
| NID_SES | 네이버 세션 |
| XSRF-TOKEN | CSRF 방지 토큰 |
| JSESSIONID | GFA 세션 |

---

## 6. GFA API (읽기 전용)

### 6.1 엔드포인트
```
Base: https://gfa.naver.com/apis/gfa/v1/adAccounts/{accountId}/
```

| Method | Path | 설명 |
|--------|------|------|
| GET | campaigns | 캠페인 목록 |
| GET | adSets | 광고그룹 목록 |
| GET | adSets/{no} | 광고그룹 상세 |

### 6.2 인증 헤더
```
Cookie: {cookie_header}
X-XSRF-TOKEN: {xsrf_token}
Accept: application/json
```

### 6.3 제한사항
- **GET만 허용** (PUT/PATCH/POST 모두 405 에러)
- 예산 변경은 API로 불가 → EC2 OpsGfaApi 사용 필수

---

## 7. 예산 변경 (EC2 OPS AI)

### 7.1 SSH 접속
```bash
ssh -i ~/.ssh/ec2_temp ec2-user@tbe.kr
```

### 7.2 예산 변경 API
```python
cd /home/ec2-user/naver-ad
python3 -c "
from app.gfa_ops.ops_state import OpsGfaApi
api = OpsGfaApi()
result = api.change_budget(adset_no, new_budget)
print(result)
"
```

### 7.3 반환값
```python
(success: bool, old_budget: float, new_budget: float, [adset_detail])
# 예: (True, 10000.0, 15000.0, [{...}])
```

### 7.4 텔레그램에서 사용
사용자 메시지: "3883414 예산 15000으로 변경"
→ SSH로 EC2 접속 → change_budget(3883414, 15000) 실행 → 결과 텔레그램 전송

---

## 8. MCP 도구 연동

Claude Code에 연결된 MCP 서버 도구들을 텔레그램에서 원격 사용:

### 8.1 GFA 광고
| 도구 | 설명 |
|------|------|
| get_gfa_ops_status | OPS AI 엔진 상태/위임/실험 |
| get_gfa_ops_scores | 캠페인별 6차원 효율 스코어 |
| get_gfa_ops_proposals | AI 예산 재배분 제안 |
| get_gfa_performance | 기간 성과 (광고비/ROAS) |
| get_gfa_funnel | TOF/MOF/BOF 퍼널 분석 |
| get_gfa_realtime_kpi | 실시간 KPI |
| get_gfa_realtime_alerts | 실시간 이상감지 알림 |

### 8.2 매출/재고
| 도구 | 설명 |
|------|------|
| get_sales_overview | 전체 매출 현황 |
| get_sales_daily | 일별 매출 |
| get_inventory_current | 현재 재고 |
| get_smart_reorder | 스마트 발주 알림 |
| get_cs_overview | CS 현황 |

### 8.3 Gmail / Notion
| 도구 | 설명 |
|------|------|
| gmail_search_messages | 이메일 검색 |
| gmail_read_message | 이메일 읽기 |
| notion-search | 노션 검색 |
| notion-fetch | 노션 페이지 조회 |

---

## 9. Claude Code 세션 구조

### 9.1 폴링 루프
```
[폴링 시작]
    ↓
[getUpdates 호출 (30초 대기)]
    ↓
[메시지 있음?]
    ├─ YES → [메시지 분석]
    │         ├─ 데이터 조회 → MCP 도구 호출 → 텔레그램 응답
    │         ├─ 예산 변경 → SSH EC2 → change_budget → 텔레그램 응답
    │         └─ 일반 대화 → 텔레그램 응답
    └─ NO  → [다음 폴링]
```

### 9.2 세션 시작 시 초기화
```bash
# 1. offset 확인
cat ~/.tg_offset

# 2. 시작 알림
python3 tg_send.py "세션 시작!"

# 3. 폴링 루프 시작
python3 tg_poll.py
```

---

## 10. 운영 환경

| 항목 | 값 |
|------|-----|
| @tbcl_gfa_bot | MacBook Pro 14에서 동작 |
| @tbminipro_bot | Mac Mini에서 동작 |
| GFA Account ID | 26343 |
| EC2 서버 | tbe.kr (ssh -i ~/.ssh/ec2_temp ec2-user@tbe.kr) |
| 그룹 Chat ID | -5109506868 |
| 개인 Chat ID | 8188602191 |

---

## 11. 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| getUpdates 빈 결과 | webhook 설정됨 | `deleteWebhook` 호출 |
| 409 Conflict | 동시 폴링 | 자동 재시도 (tg_poll.py) |
| GFA API HTML 반환 | 잘못된 엔드포인트 | `/apis/gfa/v1/` 사용 |
| 예산 변경 405 | API 쓰기 불가 | EC2 OpsGfaApi 사용 |
| 세션 만료 | 쿠키 만료 | `login_wait.py` 재실행 |
| requests 미설치 | externally-managed | urllib 표준 라이브러리 사용 |
| 그룹 메시지 안 옴 | Group Privacy ON | BotFather → Turn off |

---

## 12. 파일 구조

```
gfa-auto/
├── tg_poll.py           # 텔레그램 폴링 (그룹+개인)
├── tg_send.py           # 텔레그램 전송 (chat_id 동적)
├── login_wait.py        # GFA 세션 로그인
├── gfa_budget.py        # GFA 예산 조회 (로컬 API)
├── gfa_download.py      # GFA 리포트 다운로드 (v11)
├── gfa_bot.py           # GFA 다운로드 봇 (/gfa 명령어)
├── session.json         # 전체 쿠키
├── api_session.json     # API 호출용 세션
├── chrome-profile/      # Playwright 브라우저 프로필
├── venv/                # Python 가상환경
└── TELEGRAM_BOT_MANUAL.md  # 이 매뉴얼
```

---

*마지막 업데이트: 2026-03-10*
*구현: Claude Code (Opus 4.6) + Telegram Bot API Long Polling + EC2 OPS AI*
