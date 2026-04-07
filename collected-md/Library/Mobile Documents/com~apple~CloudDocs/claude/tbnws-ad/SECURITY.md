# TBNWS 대시보드 보안 점검 리포트

> 점검일: 2026-02-24
> 대상: tbnws-ad 전체 코드베이스 (Flask + Vanilla JS)
> GitHub 비공개 전제 — 키 재발급 제외

---

## 수정 가능 항목 (코드 변경만으로 해결)

### 1. SECRET_KEY 랜덤화

**현재 문제:** Flask 세션 서명 키가 예측 가능한 고정 문자열. 이 키를 알면 세션 쿠키 위조로 아무 사용자 로그인 가능.

**파일:** `app/config.py:101`
```python
# AS-IS
SECRET_KEY = os.getenv("SECRET_KEY", "tbnws-dashboard-secret-key-2026!")

# TO-BE
import secrets
SECRET_KEY = os.getenv("SECRET_KEY") or secrets.token_hex(32)
```

> 주의: 랜덤 생성 시 서버 재시작마다 기존 세션이 만료됨. 고정하려면 `.env`에 한번 생성해서 넣어두기.

---

### 2. auth_sso.py XSS 수정

**현재 문제:** OAuth 에러 페이지에서 `email`과 `str(e)`가 HTML 이스케이핑 없이 삽입되어 스크립트 주입 가능.

**파일:** `app/routes/auth_sso.py`

**2-1. 상단에 import 추가**
```python
# AS-IS (import 없음)

# TO-BE
from markupsafe import escape
```

**2-2. 라인 47 근처 — 접근 거부 페이지**
```python
# AS-IS
<p><strong>{email}</strong> 계정은 허용되지 않습니다.</p>

# TO-BE
<p><strong>{escape(email)}</strong> 계정은 허용되지 않습니다.</p>
```

**2-3. 라인 67 근처 — 에러 페이지**
```python
# AS-IS
<p>{str(e)}</p>

# TO-BE
<p>인증 처리 중 오류가 발생했습니다.</p>
```
> `str(e)` 내용은 `logger.exception()`으로 서버 로그에만 기록.

---

### 3. CORS origin 제한

**현재 문제:** 모든 도메인에서 credentials 포함 크로스 오리진 요청 허용. 악성 사이트에서 인증된 API 호출 가능.

**파일:** `app/__init__.py:11`
```python
# AS-IS
CORS(app, supports_credentials=True)

# TO-BE
CORS(app, supports_credentials=True, origins=["http://tbe.kr:8087"])
```

---

### 4. 보안 헤더 추가

**현재 문제:** CSP, X-Frame-Options 등 보안 헤더 없음. 클릭재킹, 콘텐츠 인젝션 취약.

**파일:** `app/__init__.py` — `create_app()` 함수 내부에 추가
```python
# TO-BE: create_app() 안에 추가
@app.after_request
def set_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    return response
```

---

### 5. 에러 메시지 내부정보 제거

**현재 문제:** `str(e)`를 그대로 클라이언트에 반환하여 DB 연결 정보, 파일 경로, SQL 쿼리 등 노출 가능.

**수정 대상 파일 및 위치:**

| 파일 | 라인 | 현재 코드 |
|------|------|-----------|
| `app/routes/gfa.py` | 383 | `return api_error(str(e), ...)` |
| `app/routes/settings.py` | 105 | `return jsonify({"ok": False, "error": str(e)})` |
| `app/routes/chatbot.py` | 558 | `error_msg = f"오류가 발생했습니다: {str(e)}"` |
| `app/routes/actions.py` | 149-150 | `result_msg = str(e)` |
| `app/routes/smartstore.py` | 다수 | `return api_error(str(e), ...)` |

**패턴:**
```python
# AS-IS
return api_error(str(e), endpoint="...")

# TO-BE
logger.exception("API error")  # 서버 로그에만 기록
return api_error("처리 중 오류가 발생했습니다", endpoint="...")
```

> actions.py의 `result_msg = str(e)`는 실행 결과 로그이므로 DB에는 기록하되, API 응답에서는 일반 메시지로 변환.

---

### 6. 파일 업로드 크기 제한

**현재 문제:** GFA CSV/ZIP 업로드에 크기 제한 없음. ZIP bomb으로 서버 메모리 고갈 가능.

**파일:** `app/__init__.py` — `create_app()` 안에 추가
```python
# TO-BE
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB
```

---

### 7. account 파라미터 화이트리스트

**현재 문제:** `account` 파라미터가 검증 없이 사용됨. 존재하지 않는 계정 조회 가능.

**방법 A: 미들웨어 방식** — `app/auth.py`에 추가
```python
# TO-BE: before_request_auth() 안에 추가
ALLOWED_ACCOUNTS = {'keychron', 'gtgear', 'aiper'}

account = request.args.get('account') or (request.get_json(silent=True) or {}).get('account')
if account and account not in ALLOWED_ACCOUNTS:
    return jsonify({'ok': False, 'error': 'invalid account'}), 400
```

**방법 B: config.py에 허용 목록 정의**
```python
# app/config.py
ALLOWED_ACCOUNTS = set(SA_ACCOUNTS.keys())  # {'keychron', 'gtgear'}
```

---

### 8. 챗봇 세션 소유자 검증

**현재 문제:** 인증된 사용자가 다른 사용자의 채팅 세션을 읽기/삭제/메시지 전송 가능.

**파일:** `app/routes/chatbot.py`

**8-1. 세션 생성 시 user_email 저장 (라인 373 근처)**
```python
# AS-IS
cur.execute("""
    INSERT INTO chat_sessions (session_id, title, account)
    VALUES (%s, %s, %s)
""", (session_id, title, account))

# TO-BE
user_email = session.get('user', {}).get('email', '')
cur.execute("""
    INSERT INTO chat_sessions (session_id, title, account, user_email)
    VALUES (%s, %s, %s, %s)
""", (session_id, title, account, user_email))
```

> chat_sessions 테이블에 `user_email VARCHAR(200)` 컬럼 추가 필요.

**8-2. 세션 조회/삭제/메시지 전송 시 소유자 체크**
```python
# 예시: delete_session()
user_email = session.get('user', {}).get('email', '')
cur.execute("DELETE FROM chat_sessions WHERE session_id=%s AND user_email=%s",
            (session_id, user_email))
```

---

### 9. flask-limiter 추가

**현재 문제:** Rate limiting 없어서 brute force, API 비용 공격 가능.

**9-1. 의존성 추가:** `requirements.txt`
```
flask-limiter>=3.5
```

**9-2. 초기화:** `app/__init__.py`
```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    get_remote_address,
    app=app,
    default_limits=["200 per minute"],
    storage_uri="memory://",
)
```

**9-3. 엔드포인트별 제한 적용:**

| 엔드포인트 | 제한 | 파일 |
|------------|------|------|
| `/api/chat/send` | `10/minute` | `chatbot.py` |
| `/auth/login` | `10/minute` | `auth_sso.py` |
| `/auth/callback` | `10/minute` | `auth_sso.py` |
| `/api/telegram/send_report` | `5/minute` | `telegram.py` |
| `/api/telegram/test` | `5/minute` | `telegram.py` |
| `/api/settings/credentials` POST | `10/minute` | `settings.py` |

```python
# 사용 예시 (chatbot.py)
from app import limiter  # 또는 Blueprint에서 import

@bp.route("/api/chat/send", methods=["POST"])
@limiter.limit("10/minute")
def send_message():
    ...
```

> `limiter` 객체를 Blueprint에서 쓰려면 `app/__init__.py`에서 export 하거나 `init_app()` 패턴 사용.

---

### 10. chatbot 마크다운 XSS 방어

**현재 문제:** AI 응답을 마크다운 렌더링 후 innerHTML로 삽입. 악의적 HTML이 포함될 경우 스크립트 실행 가능.

**파일:** `app/static/js/chatbot.js`

```javascript
// AS-IS: 마크다운 렌더링 결과를 그대로 삽입
el.innerHTML = this._renderMd(fullText);

// TO-BE: script/iframe/on* 이벤트 제거
_sanitize(html) {
    const div = document.createElement('div');
    div.innerHTML = html;
    div.querySelectorAll('script,iframe,object,embed,form').forEach(el => el.remove());
    div.querySelectorAll('*').forEach(el => {
        [...el.attributes].forEach(attr => {
            if (attr.name.startsWith('on') || attr.value.startsWith('javascript:')) {
                el.removeAttribute(attr.name);
            }
        });
    });
    return div.innerHTML;
}

// 사용
el.innerHTML = this._sanitize(this._renderMd(fullText));
```

---

## 참고: 제가 직접 하기 어려운 항목

| 항목 | 이유 |
|------|------|
| HTTPS 적용 | nginx + Let's Encrypt 설정, DNS 변경 필요 |
| CSRF 보호 | 프론트엔드 모든 POST 요청에 토큰 추가 (변경 범위 큼) |
| 시크릿 .env 분리 | 서버 환경변수 설정 + 배포 프로세스 변경 필요 |
| 키 재발급 | 각 서비스 콘솔에서 수동 작업 |

---

## 요약 체크리스트

- [x] 1. SECRET_KEY 랜덤화 (`config.py`)
- [x] 2. auth_sso.py XSS 수정 (`auth_sso.py`)
- [x] 3. CORS origin 제한 (`__init__.py`)
- [x] 4. 보안 헤더 추가 (`__init__.py`)
- [x] 5. 에러 메시지 내부정보 제거 (여러 파일)
- [x] 6. 파일 업로드 크기 제한 (`__init__.py`)
- [x] 7. account 파라미터 화이트리스트 (`auth.py`)
- [x] 8. 챗봇 세션 소유자 검증 (`chatbot.py`)
- [x] 9. flask-limiter 추가 (`requirements.txt` + 여러 파일)
- [x] 10. chatbot 마크다운 XSS 방어 (`chatbot.js`)
