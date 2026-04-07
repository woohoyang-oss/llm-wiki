# Claude Code x Telegram Bot 폴링 구현 매뉴얼

## 개요
Claude Code CLI 세션을 Telegram Bot과 연결하여, 텔레그램 메시지를 통해 Claude Code의 모든 기능을 원격으로 사용할 수 있는 구조입니다.

**핵심 원리**: Claude Code가 직접 Telegram Bot API를 Long Polling 방식으로 호출하여 메시지를 수신하고, 처리 후 응답을 전송합니다.

---

## 1. 사전 준비

### 1.1 Telegram Bot 생성
1. Telegram에서 `@BotFather` 검색
2. `/newbot` 명령어 전송
3. Bot 이름 및 username 설정
4. **BOT_TOKEN** 발급 (예: `8794280034:AAGxH9IbYkttyNFtmA3M__BFVuUJcwv4MSQ`)

### 1.2 Chat ID 확인
1. 생성한 봇에게 아무 메시지 전송
2. 브라우저에서 접속:
   ```
   https://api.telegram.org/bot{BOT_TOKEN}/getUpdates
   ```
3. JSON 응답에서 `result[0].message.chat.id` 값이 **CHAT_ID** (예: `8188602191`)

### 1.3 Offset 파일 생성
```bash
echo "0" > ~/.tg_offset
```
- 이 파일은 마지막으로 처리한 update_id를 저장합니다.
- 중복 처리 방지 및 세션 재시작 시 이어서 폴링하기 위한 용도입니다.

---

## 2. 핵심 구성 요소

### 2.1 환경 변수 (세션 내 상수)
```
BOT_TOKEN: 텔레그램 봇 토큰
CHAT_ID: 대화 상대의 chat_id
OFFSET_FILE: ~/.tg_offset (offset 저장 경로)
```

### 2.2 폴링 스크립트 (메시지 수신)
```python
python3 -c "
import requests
BOT = '{BOT_TOKEN}'
offset_file = '/Users/{username}/.tg_offset'
with open(offset_file) as f:
    offset = int(f.read().strip())
r = requests.get(f'https://api.telegram.org/bot{BOT}/getUpdates',
                params={'offset': offset, 'timeout': 30}, timeout=35)
data = r.json()
if data.get('ok') and data.get('result'):
    for update in data['result']:
        offset = update['update_id'] + 1
        if 'message' in update and 'text' in update['message']:
            text = update['message']['text']
            if not text.startswith('/'):
                print(text)
    with open(offset_file, 'w') as f:
        f.write(str(offset))
else:
    print('NO_MESSAGE')
"
```

**주요 파라미터:**
- `offset`: 마지막 처리한 update_id + 1 (중복 방지)
- `timeout=30`: Long Polling 대기 시간 (초)
- `timeout=35`: requests 라이브러리 타임아웃 (Long Polling보다 약간 길게)

**동작 방식:**
1. offset 파일에서 마지막 처리 ID 읽기
2. Telegram API에 Long Polling 요청 (최대 30초 대기)
3. 새 메시지가 있으면 텍스트 출력 + offset 파일 업데이트
4. 없으면 `NO_MESSAGE` 출력

### 2.3 메시지 전송 스크립트 (응답 발송)
```python
python3 -c "
import requests
BOT = '{BOT_TOKEN}'
CHAT_ID = {CHAT_ID}
msg = '''여기에 응답 메시지'''
r = requests.post(f'https://api.telegram.org/bot{BOT}/sendMessage',
    json={'chat_id': CHAT_ID, 'text': msg}, timeout=10)
print('sent:', r.json().get('ok'))
"
```

**제한사항:**
- Telegram 메시지 최대 길이: **4096자**
- 긴 메시지는 여러 파트로 분할 전송 필요

### 2.4 파일 전송 스크립트
```python
python3 -c "
import requests
BOT = '{BOT_TOKEN}'
CHAT_ID = {CHAT_ID}
with open('/path/to/file', 'rb') as f:
    r = requests.post(f'https://api.telegram.org/bot{BOT}/sendDocument',
        data={'chat_id': CHAT_ID},
        files={'document': ('filename.ext', f)},
        timeout=30)
print('sent:', r.json().get('ok'))
"
```

---

## 3. Claude Code 세션 구조

### 3.1 폴링 루프
Claude Code는 Bash 도구를 사용하여 반복적으로 폴링 스크립트를 실행합니다:

```
[폴링 시작]
    ↓
[getUpdates 호출 (30초 대기)]
    ↓
[메시지 있음?]
    ├─ YES → [메시지 처리] → [응답 전송] → [다음 폴링]
    └─ NO  → [NO_MESSAGE] → [다음 폴링]
```

### 3.2 메시지 처리 흐름
1. 폴링으로 사용자 메시지 수신
2. Claude Code가 메시지 내용 분석
3. 필요한 도구 호출 (MCP, Bash, Gmail, Notion 등)
4. 결과를 텔레그램 메시지로 전송
5. 다음 폴링 반복

### 3.3 세션 시작 시 초기화
세션 시작 또는 재시작 시:
1. offset 파일 존재 확인 및 읽기
2. 세션 시작 알림 전송 ("세션 재시작 완료!")
3. 폴링 루프 시작

---

## 4. CLAUDE.md 설정

`~/.claude/CLAUDE.md`에 다음 내용을 추가하여 세션 시작 시 자동으로 텔레그램 폴링을 시작하도록 설정할 수 있습니다:

```markdown
## Telegram Bot 자동 시작
- 세션 시작 시 텔레그램 Long Polling 자동 시작
- BOT_TOKEN: {your_token}
- CHAT_ID: {your_chat_id}
- OFFSET_FILE: ~/.tg_offset
- 사용자 질문 없이 자율적으로 처리
```

---

## 5. 활용 가능한 MCP 도구들

Claude Code에 연결된 MCP 서버들을 텔레그램을 통해 원격으로 사용:

### 5.1 tbe.kr Data API (커머스)
| 도구 | 설명 |
|------|------|
| `get_sales_overview` | 매출 현황 |
| `get_sales_daily` | 일별 매출 |
| `get_sales_by_channel` | 채널별 매출 |
| `get_sales_by_model` | 모델별 매출 |
| `get_inventory_current` | 현재 재고 |
| `get_smart_reorder` | 스마트 발주 알림 |
| `get_cs_overview` | CS 현황 |
| `get_unanswered_inquiries` | 미답변 문의 |
| `get_ad_overview` | 광고 현황 |
| `get_margin_analysis` | 마진 분석 |

### 5.2 Gmail
| 도구 | 설명 |
|------|------|
| `gmail_search_messages` | 이메일 검색 |
| `gmail_read_message` | 이메일 읽기 |
| `gmail_create_draft` | 초안 작성 |

### 5.3 Notion
| 도구 | 설명 |
|------|------|
| `notion-search` | 페이지 검색 |
| `notion-fetch` | 페이지 조회 |
| `notion-create-pages` | 페이지 생성 |
| `notion-update-page` | 페이지 수정 |

### 5.4 ERP API (직접 호출)
```
https://api.tbnws.co.kr/api/erp/getAnnualSalesList
https://api.tbnws.co.kr/api/erp/getSalesModelList?year=2026
```

---

## 6. 주의사항 및 제한사항

### 6.1 세션 지속성
- Claude Code 세션은 컨텍스트 윈도우 한계가 있음
- 장시간 폴링 시 자동으로 이전 대화 압축됨
- 세션 종료 시 자동 재시작 불가 (수동으로 다시 시작 필요)

### 6.2 메시지 처리
- `/` 로 시작하는 메시지는 봇 명령어로 간주하여 무시
- 한 번에 한 메시지만 처리 (동시 처리 불가)
- 긴 분석은 여러 메시지로 분할 전송

### 6.3 에러 핸들링
- MCP 도구 에러 시: 다른 API 경로로 대체 시도
- 네트워크 에러 시: 다음 폴링에서 자동 재시도
- offset 파일 손상 시: 수동으로 현재 update_id 설정 필요

### 6.4 보안
- BOT_TOKEN은 외부에 노출하지 않도록 주의
- offset 파일은 로컬에만 저장
- 봇은 지정된 CHAT_ID에만 응답하도록 구현 권장

---

## 7. 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| 메시지 중복 수신 | offset 미업데이트 | offset 파일 확인 및 수정 |
| 메시지 미수신 | offset이 너무 큼 | offset 파일을 0으로 리셋 |
| 긴 응답 잘림 | 4096자 제한 | 메시지 분할 로직 확인 |
| 세션 끊김 | 컨텍스트 오버플로우 | 새 세션 시작 후 offset 파일로 이어서 진행 |
| MCP 도구 에러 | API 파라미터 불일치 | 에러 메시지 확인 후 파라미터 조정 |

---

## 8. 빠른 시작 체크리스트

- [ ] BotFather로 봇 생성 및 토큰 발급
- [ ] Chat ID 확인
- [ ] `~/.tg_offset` 파일 생성 (`echo "0" > ~/.tg_offset`)
- [ ] Claude Code 세션 시작
- [ ] `/yolo` 입력 (모든 권한 허용)
- [ ] 폴링 시작 지시
- [ ] 텔레그램에서 메시지 전송하여 테스트

---

*마지막 업데이트: 2026-03-10*
*구현: Claude Code (Sonnet 4.5) + Telegram Bot API Long Polling*
