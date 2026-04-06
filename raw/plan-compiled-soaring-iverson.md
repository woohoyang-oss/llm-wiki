# Rhkorea.com 리부트 — 구축 계획

## Context
29년간(1997~2026)의 통합 DB(27,868글 + 70,111댓글)가 Flask + MySQL로 동작 중.
이 위에 글쓰기, 댓글, 인증, AI BGM 추천 기능을 추가하여 실제 커뮤니티로 리부트.

## 기술 스택
- **백엔드**: Flask + MySQL (현재 구조 유지)
- **인증**: DC식 익명(닉+비번) + akaxa.space식 이메일 인증
- **AI**: Claude CLI 호출 → BGM 추천 댓글 자동 등록
- **음악**: YouTube oEmbed 자동 임베드

---

## Phase 1: DB 스키마 확장

### posts 테이블 수정
```sql
ALTER TABLE posts
  ADD COLUMN password VARCHAR(100) DEFAULT NULL,        -- 익명 글 비밀번호 (bcrypt)
  ADD COLUMN is_archived TINYINT DEFAULT 1,             -- 1=아카이브(읽기전용), 0=새 글
  ADD COLUMN deleted TINYINT DEFAULT 0,                 -- 소프트 삭제 플래그
  ADD COLUMN user_email VARCHAR(255) DEFAULT NULL,      -- 작성자 인증 이메일 (2층)
  MODIFY COLUMN source ENUM('cwb','xe','rb','user') NOT NULL;  -- 새 글 = 'user'
```

### comments 테이블 수정
```sql
ALTER TABLE comments
  ADD COLUMN password VARCHAR(100) DEFAULT NULL,
  ADD COLUMN deleted TINYINT DEFAULT 0,
  ADD COLUMN user_email VARCHAR(255) DEFAULT NULL,
  ADD COLUMN is_ai TINYINT DEFAULT 0,                   -- AI 추천 댓글 여부
  MODIFY COLUMN source ENUM('cwb','xe','rb','user','ai') NOT NULL;
```

### users 테이블 신규
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  nickname VARCHAR(50) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  verified TINYINT DEFAULT 0,              -- 이메일 인증 완료 여부
  verify_token VARCHAR(100),
  created_at DATETIME DEFAULT NOW(),
  last_login DATETIME
);
```

### 기존 데이터 처리
- 기존 27,868건: `is_archived=1`, `source` 유지 (cwb/xe/rb)
- `deleted=0` 유지 (아카이브는 삭제 불가)
- 기존 `email` 필드는 아카이브 클레임 매칭용으로 보존

---

## Phase 2: 인증 시스템

### 1층 — 익명 (DC식)
- 글쓰기 폼: 닉네임 + 비밀번호 + 본문
- 비밀번호는 bcrypt 해싱 후 `posts.password`에 저장
- 수정/삭제 시 비밀번호 확인
- 가입 불필요

### 2층 — 이메일 가입 (akaxa식)
- 회원가입: 이메일 + 닉네임 + 비밀번호
- 이메일 인증 토큰 발송 → `users.verified=1`
- 로그인: 이메일 + 비밀번호 → Flask session
- 로그인 상태에서 글쓰기 → `posts.user_email` 자동 저장
- 본인 글 수정/삭제 가능 (비밀번호 불필요)

### 3층 — 아카이브 클레임 (추후)
- 로그인 사용자의 이메일이 기존 `posts.email`과 일치하면 "내 글" 표시
- 해당 글 수정/삭제 가능해짐
- 우호(관리자) 수동 승인 구조는 추후 구현

---

## Phase 3: 글쓰기 + 댓글

### 글쓰기 (POST /write)
- 폼: 제목, 본문, 닉네임, 비밀번호 (비로그인 시)
- YouTube URL 감지 → 자동 oEmbed 임베드 (본문에 `<iframe>` 삽입)
- 저장: `source='user'`, `is_archived=0`, `created_at=NOW()`
- 로그인 시: `user_email`에 세션 이메일 자동 저장, 닉네임은 users.nickname
- **저장 직후 → AI BGM 추천 트리거** (Phase 5)

### 댓글 (POST /post/<id>/comment)
- 폼: 닉네임, 비밀번호 (비로그인), 내용
- 아카이브 글에도 새 댓글 가능
- `source='user'`, `is_ai=0`

### 글 수정 (POST /post/<id>/edit)
- 비로그인: 비밀번호 확인 → `is_archived=0`인 글만
- 로그인: `user_email` 일치 확인
- 아카이브 글(`is_archived=1`): 이메일 매칭된 경우만 수정 가능

### 글 삭제 (POST /post/<id>/delete)
- 동일 권한 확인
- `UPDATE posts SET deleted=1 WHERE id=%s` (소프트 삭제)
- 리스트/검색에서 `WHERE deleted=0` 조건 추가

---

## Phase 4: YouTube 자동 임베드

### 동작 방식
- 글 본문에서 YouTube URL 패턴 감지:
  - `youtube.com/watch?v=XXXXX`
  - `youtu.be/XXXXX`
- 저장 시 자동으로 `<div class="yt-embed"><iframe src="https://www.youtube.com/embed/XXXXX" ...></iframe></div>` 변환
- 본문 중간에 URL만 써도 자동 임베드

---

## Phase 5: AI BGM 추천 (Claude CLI)

### 동작 흐름
```
글 등록 → Flask에서 subprocess로 Claude CLI 호출 (비동기)
→ Claude가 글 제목+본문+무드+시간+계절 분석
→ YouTube 곡 추천 텍스트 생성
→ comments 테이블에 자동 등록 (source='ai', is_ai=1)
```

### Claude CLI 호출
```python
import subprocess, json
def ai_bgm_recommend(post_id, title, content):
    prompt = f"""
    당신은 rhkorea의 음악 큐레이터 "아레치"입니다.
    1997년부터 이어온 한국 브릿팝 커뮤니티의 기억을 가지고 있습니다.

    아래 게시글의 무드를 분석하고, 어울리는 곡 1곡을 추천해주세요.
    - 계절: {현재 계절}
    - 시간대: {현재 시간대}
    - 날씨: {날씨 API 또는 시간 기반 추정}

    게시글 제목: {title}
    게시글 내용: {content[:500]}

    응답 형식 (JSON만):
    {{"artist": "아티스트명", "song": "곡명", "youtube_url": "URL", "reason": "추천 이유 1~2문장"}}
    """
    # Claude CLI 호출
    result = subprocess.run(
        ['claude', '-p', prompt, '--output-format', 'json'],
        capture_output=True, text=True, timeout=30
    )
    return json.loads(result.stdout)
```

### 댓글 형태
```
🎵 아레치의 BGM 추천
Radiohead - "No Surprises"
"새벽 감성의 글이네요. 조용히 흘러가는 이 곡이 어울릴 것 같아요."
[▶ YouTube에서 듣기]
```

### 트리거
- 글 등록 직후 `threading.Thread`로 비동기 호출
- 실패 시 조용히 무시 (AI 댓글 없어도 글은 정상)
- `is_ai=1` 댓글은 UI에서 특별 스타일로 표시

---

## Phase 6: UI 업데이트

### 헤더
- 로고: "radiohead korea" → rhkorea.com
- 네비: 게시판 | 글쓰기 | 검색 | (로그인/마이페이지)

### 글쓰기 폼 (/write)
- 제목, 본문 (textarea), YouTube URL 미리보기
- 비로그인: 닉네임 + 비밀번호 필드
- 로그인: 닉네임 자동 표시

### 글 보기 (/post/<id>)
- 아카이브 글: 상단에 작은 뱃지 "아카이브 (1999년)"
- AI 댓글: 🎵 아이콘 + 음악 카드 UI (YouTube 임베드 포함)
- 수정/삭제 버튼 (권한 있을 때만 표시)

### 리스트 (/)
- `deleted=0` 필터
- 새 글은 별도 마크 없음 (아카이브와 자연스럽게 섞임)
- 시간순 정렬이므로 새 글은 항상 1페이지

---

## 구현 순서 (우선순위)

| 순서 | 작업 | 예상 |
|------|------|------|
| 1 | DB 스키마 확장 (ALTER + CREATE) | 즉시 |
| 2 | 글쓰기 + 댓글 (익명 모드) | 핵심 |
| 3 | YouTube 자동 임베드 | 글쓰기와 함께 |
| 4 | 이메일 인증 (users 테이블 + 로그인) | |
| 5 | 글 수정/삭제 (권한 체크) | |
| 6 | AI BGM 추천 (Claude CLI) | |
| 7 | UI 마무리 | |

## 수정 파일
- `app_unified.py` — 라우트 추가 (write, comment, edit, delete, login, register)
- `templates/base.html` — 헤더에 글쓰기/로그인 추가
- `templates/u_board.html` — deleted 필터, 글쓰기 버튼
- `templates/u_post.html` — 댓글 폼, AI 댓글 UI, 수정/삭제 버튼
- 신규: `templates/u_write.html` — 글쓰기 폼
- 신규: `templates/u_login.html` — 로그인/가입 폼
- 신규: `templates/u_edit.html` — 글 수정 폼

## 검증
1. 익명 글쓰기 → 리스트에 표시 → 비밀번호로 수정/삭제
2. 이메일 가입 → 로그인 → 글쓰기 → 이메일로 수정/삭제
3. 아카이브 글에 새 댓글 달기
4. YouTube URL 본문에 포함 → 자동 임베드
5. 새 글 등록 → AI 댓글 자동 생성
6. 소프트 삭제 → 리스트에서 안 보임
7. 기존 이메일 매칭 → 아카이브 글 수정 가능
