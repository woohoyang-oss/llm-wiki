# Claude Code × 텔레그램 실시간 연동 가이드

## 개요

Claude Code CLI 세션과 텔레그램 봇을 연동하여, 텔레그램 메시지로 Claude에게 작업을 지시하고 결과를 받을 수 있습니다. 터미널 앞에 앉아있지 않아도 모바일에서 개발 작업을 지시할 수 있습니다.

---

## 아키텍처

```
[사용자 모바일/PC]
    ↓ 텔레그램 메시지
[Telegram Bot API]
    ↓ Long Polling (getUpdates)
[Claude Code CLI 세션]
    ↓ 메시지 분석 + 도구 실행
    ↓ (파일 편집, 코드 검색, 서버 배포, MCP 도구 등)
    ↓ sendMessage API
[Telegram Bot API]
    ↓
[사용자 모바일/PC] ← 결과 수신
```

---

## 사전 준비

### 1. 텔레그램 봇 생성

1. 텔레그램에서 [@BotFather](https://t.me/BotFather) 검색
2. `/newbot` 명령으로 봇 생성
3. 봇 이름과 username 설정
4. **BOT_TOKEN** 발급받기 (형식: `1234567890:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)

### 2. Chat ID 확인

1. 생성한 봇에게 아무 메시지 전송
2. 브라우저에서 접속:
   ```
   https://api.telegram.org/bot{BOT_TOKEN}/getUpdates
   ```
3. 응답 JSON에서 `message.chat.id` 값이 **CHAT_ID**

### 3. Offset 파일 생성

```bash
echo "0" > ~/.tg_offset
```

---

## 설정 방법

### CLAUDE.md에 지시문 추가

프로젝트 CLAUDE.md 또는 글로벌 `~/.claude/CLAUDE.md`에 아래 내용을 추가합니다:

```markdown
## 텔레그램 실시간 연동 (세션 시작 시 자동 실행)

세션이 시작되면 아래 절차를 자동으로 수행하세요:

1. 텔레그램 봇 연결 시작
   - BOT_TOKEN: {발급받은 토큰}
   - CHAT_ID: {확인한 Chat ID}
   - Offset 파일: ~/.tg_offset

2. 연결 알림 발송
   ```
   curl -s -X POST "https://api.telegram.org/bot{BOT_TOKEN}/sendMessage" \
     -H "Content-Type: application/json" \
     -d '{"chat_id": {CHAT_ID}, "text": "✅ Claude Code 세션 시작. 텔레그램 연동 완료."}'
   ```

3. Long Polling 루프 시작
   - offset 파일에서 마지막 offset 읽기
   - getUpdates?offset={offset}&timeout=5 로 폴링
   - 메시지 수신 시 처리 후 응답
   - offset 업데이트 후 다시 폴링
   - /로 시작하는 명령은 무시

4. 메시지 처리
   - 사용자 메시지를 분석하고 적절한 도구(MCP, Bash 등) 활용하여 응답
   - 응답은 sendMessage API로 텔레그램에 전송
   - 4096자 초과 시 분할 발송

5. 주의사항
   - 동일 봇 토큰으로 다른 세션이 폴링하면 409 충돌 발생
   - 이 봇은 이 세션 전용으로 사용
```

---

## 동작 원리

### Long Polling

Claude Code가 Telegram Bot API의 `getUpdates`를 반복 호출하여 새 메시지를 감지합니다.

```bash
# 폴링 요청 (timeout=30초)
curl -s "https://api.telegram.org/bot{TOKEN}/getUpdates?offset={OFFSET}&timeout=30"
```

- **offset**: 마지막으로 처리한 메시지 ID+1. `~/.tg_offset`에 저장하여 중복 처리 방지
- **timeout**: Long Polling 대기 시간 (초). 메시지가 없으면 timeout까지 대기 후 빈 응답 반환

### 메시지 수신 → 처리 → 응답

1. `getUpdates` 응답에서 `update_id`와 `message.text` 추출
2. Claude가 메시지 내용을 분석하여 적절한 작업 수행
3. 결과를 `sendMessage` API로 텔레그램에 전송
4. offset 업데이트 (`update_id + 1`)

### 응답 전송

```bash
curl -s -X POST "https://api.telegram.org/bot{TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d '{"chat_id": {CHAT_ID}, "text": "응답 내용", "parse_mode": "Markdown"}'
```

- **4096자 제한**: 텔레그램 메시지 최대 길이. 초과 시 분할 발송 필요
- **parse_mode**: `Markdown` 또는 `HTML` 지원 (선택)

---

## 활용 예시

### 코드 작업 지시
```
사용자: "login.js에서 비밀번호 유효성 검사 추가해줘"
Claude: (파일 읽기 → 코드 수정 → 결과 전송)
```

### 서버 배포
```
사용자: "변경사항 서버에 배포해줘"
Claude: (scp로 파일 전송 → 서비스 재시작 → 결과 전송)
```

### 데이터 조회 (MCP 도구 활용)
```
사용자: "오늘 GFA 실시간 성과 알려줘"
Claude: (MCP 도구 호출 → 데이터 분석 → 요약 전송)
```

### Git 작업
```
사용자: "깃헙 푸시해줘"
Claude: (git add → commit → push → 결과 전송)
```

---

## 주의사항

1. **봇 토큰 보안**: BOT_TOKEN은 비밀키입니다. 공개 저장소에 커밋하지 마세요.
2. **단일 세션**: 같은 봇 토큰으로 여러 세션이 동시에 폴링하면 409 충돌이 발생합니다.
3. **세션 지속성**: Claude Code 세션이 종료되면 폴링도 중단됩니다. 새 세션에서 다시 시작해야 합니다.
4. **컨텍스트 제한**: 긴 대화가 이어지면 Claude의 컨텍스트 윈도우가 가득 차서 세션이 재시작될 수 있습니다.
5. **권한**: Claude Code의 도구 권한 설정에 따라 일부 작업은 터미널에서 승인이 필요할 수 있습니다.

---

## 파일 구조

```
~/.claude/CLAUDE.md          ← 텔레그램 연동 지시문 (글로벌)
~/.tg_offset                 ← 마지막 처리한 메시지 offset
프로젝트/CLAUDE.md            ← 프로젝트별 지시문 (선택)
```

---

## FAQ

**Q: 텔레그램 대신 Slack/Discord도 가능한가요?**
A: 동일한 원리로 가능합니다. 각 플랫폼의 Bot API를 사용하면 됩니다.

**Q: 여러 프로젝트에서 동시에 사용할 수 있나요?**
A: 봇을 여러 개 만들거나, 메시지에 프로젝트 식별자를 포함하는 방식으로 가능합니다.

**Q: 파일 전송도 되나요?**
A: `sendDocument` API를 사용하면 파일 전송도 가능합니다. 현재는 텍스트 메시지 위주입니다.
