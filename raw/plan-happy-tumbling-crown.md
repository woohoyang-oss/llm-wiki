# Rhkorea 리부트 Phase 1 구현 계획

## Context
97년부터 이어온 브릿팝 커뮤니티 rhkorea.com을 Next.js로 리부트.
현재 MySQL `rhk_unified` DB에 27,868글 + 70,111댓글이 있고 Flask 뷰어로 동작 중.
Phase 1은 AI/YouTube 채널 연동을 제외한 전체 사이트 기능 구현.

## 기술 스택
- **Next.js 15** (App Router) + TypeScript
- **MySQL** (기존 `rhk_unified` DB 그대로 사용 + 새 테이블 추가)
- **mysql2** (Node.js MySQL 드라이버)
- **Tailwind CSS v4** (다크 모드 기본)
- **2층 인증**: akaxa.space OTP (이메일 → OTP 인증)
- **1층**: 닉네임+비밀번호 익명 (bcrypt)
- **Vercel** 배포 대상

## DB 변경 (기존 테이블 유지 + 새 테이블 추가)

### 기존 유지
- `posts` — is_archive 컬럼 추가
- `comments` — is_archive 컬럼 추가
- `files` — 그대로

### 새 테이블
```sql
-- 가입 회원
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  nickname VARCHAR(100),
  bio TEXT,
  is_admin BOOLEAN DEFAULT FALSE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 아카이브 클레임
CREATE TABLE archive_claims (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL REFERENCES users(id),
  post_id INT NOT NULL REFERENCES posts(id),
  status ENUM('pending','approved','rejected') DEFAULT 'pending',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  reviewed_at DATETIME,
  reviewed_by INT REFERENCES users(id)
);

-- 공유된 음악 트랙 (플레이리스트용)
CREATE TABLE shared_tracks (
  id INT AUTO_INCREMENT PRIMARY KEY,
  post_id INT REFERENCES posts(id),
  youtube_url VARCHAR(500) NOT NULL,
  youtube_id VARCHAR(20) NOT NULL,
  title VARCHAR(500),
  shared_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- posts 테이블 변경
ALTER TABLE posts
  ADD COLUMN is_archive BOOLEAN DEFAULT FALSE,
  ADD COLUMN user_id INT REFERENCES users(id),
  ADD COLUMN author_password VARCHAR(255); -- bcrypt hash (익명용)

-- comments 테이블 변경
ALTER TABLE comments
  ADD COLUMN is_archive BOOLEAN DEFAULT FALSE,
  ADD COLUMN user_id INT REFERENCES users(id),
  ADD COLUMN author_password VARCHAR(255);

-- 기존 아카이브 데이터 마킹
UPDATE posts SET is_archive = TRUE;
UPDATE comments SET is_archive = TRUE;
```

## 프로젝트 구조
```
rhkorea/
├── web/                            # Next.js 앱 (새로 생성)
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx          # 루트 레이아웃 + 다크 테마
│   │   │   ├── page.tsx            # 메인 = 게시판 목록
│   │   │   ├── post/
│   │   │   │   └── [id]/page.tsx   # 글 보기
│   │   │   ├── write/page.tsx      # 글쓰기
│   │   │   ├── search/page.tsx     # 검색
│   │   │   ├── playlist/page.tsx   # 주간/월간 플레이리스트
│   │   │   ├── login/page.tsx      # 이메일 OTP 로그인
│   │   │   ├── profile/page.tsx    # 프로필 + 내 글 히스토리
│   │   │   └── api/
│   │   │       ├── posts/route.ts          # 글 CRUD
│   │   │       ├── comments/route.ts       # 댓글 CRUD
│   │   │       ├── auth/login/route.ts     # OTP 발송
│   │   │       ├── auth/verify/route.ts    # OTP 검증 + JWT 발급
│   │   │       ├── auth/me/route.ts        # 현재 유저
│   │   │       ├── claims/route.ts         # 아카이브 클레임
│   │   │       └── tracks/route.ts         # 공유 트랙
│   │   ├── components/
│   │   │   ├── Header.tsx
│   │   │   ├── PostList.tsx
│   │   │   ├── PostView.tsx
│   │   │   ├── CommentSection.tsx
│   │   │   ├── WriteForm.tsx
│   │   │   ├── YouTubeEmbed.tsx
│   │   │   ├── Pagination.tsx
│   │   │   ├── EraTag.tsx          # CWB/XE/Rb/NEW 시대 배지
│   │   │   └── RandomArchive.tsx   # 랜덤 옛날글
│   │   ├── lib/
│   │   │   ├── db.ts               # MySQL 커넥션 풀
│   │   │   ├── auth.ts             # JWT 발급/검증 + akaxa OTP
│   │   │   ├── youtube.ts          # URL 파싱/임베드
│   │   │   └── utils.ts
│   │   └── types/
│   │       └── index.ts
│   ├── sql/
│   │   └── 001_new_tables.sql      # 새 테이블 + ALTER
│   └── public/
```

## 구현 순서 (7단계)

### Step 1: Next.js 프로젝트 초기화
- `create-next-app` 셋업 (TypeScript + Tailwind + App Router)
- MySQL 커넥션 풀 (mysql2/promise)
- 기본 레이아웃 (다크 테마, Header)
- `.env.local` (DB 접속 정보)

### Step 2: DB 스키마 변경
- `sql/001_new_tables.sql` 작성
- 기존 posts/comments에 is_archive, user_id, author_password 추가
- users, archive_claims, shared_tracks 테이블 생성
- 기존 데이터 `is_archive = TRUE` 마킹

### Step 3: 게시판 목록 + 글 보기 (읽기 전용)
- 메인 페이지: 단일 타임라인, 30개씩 페이지네이션
- 글 보기: 본문 + 댓글 + 첨부 + 이전/다음글
- 시대 배지 (CWB=핑크, XE=블루, Rb=그린, NEW=화이트)
- 검색 (FULLTEXT + LIKE 폴백)
- 랜덤 아카이브 글 노출

### Step 4: 1층 — 익명 글쓰기/댓글
- 닉네임 + 비밀번호로 글쓰기 (가입 불필요)
- 댓글도 동일
- 비밀번호로 수정/삭제
- XSS 방지 (sanitize)

### Step 5: 2층 — 이메일 가입 (akaxa.space OTP)
- 로그인 페이지: 이메일 입력 → OTP 발송 → 검증
- JWT 세션 (httpOnly cookie)
- 프로필 페이지 (닉네임, 바이오 설정)
- 가입 회원 글쓰기 시 user_id 연동
- 내 글/댓글 히스토리

### Step 6: YouTube 임베드 + 플레이리스트
- 본문 내 YouTube URL 자동 감지 → iframe 임베드
- 글 저장 시 youtube_id 추출 → shared_tracks 저장
- 플레이리스트 페이지: 주간/월간별 공유곡 목록

### Step 7: 3층 — 아카이브 클레임
- 아카이브 글에 "이 글 내 거예요" 버튼 (로그인 필요)
- archive_claims 테이블에 요청 저장
- 관리자 승인 페이지
- 승인 시 posts.user_id 연결

## 디자인 방향
- **다크 모드 기본** — #0a0a0a 배경, 연한 그레이 텍스트
- 시대 배지로 아카이브와 새 글을 자연스럽게 구분
- 테이블 기반 게시판 (기존 Flask 뷰어 계승)
- Pretendard 폰트 + monospace 제목
- 미니멀 — 글과 대화 중심

## 검증
1. `npm run dev` → http://localhost:3000
2. 메인: 기존 아카이브 글 27,868개 목록 확인
3. 글 보기: 본문 렌더링, 댓글, 시대 배지
4. 익명 글쓰기 → 닉네임+비밀번호로 등록 확인
5. 이메일 OTP 로그인 → 프로필 설정
6. YouTube URL 글 → 자동 임베드
7. 플레이리스트 페이지 확인
8. 아카이브 클레임 요청 → 관리자 승인
