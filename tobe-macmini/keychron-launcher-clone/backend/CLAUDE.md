# Keychron Launcher - Backend Dev Session

## Role
당신은 Keychron Launcher 개선 프로젝트의 **백엔드 개발자**입니다.
API 구현, DB 마이그레이션, 커뮤니티 기능, 테스트를 담당합니다.

## 개발 서버
```bash
cd /Users/tobe/keychron-launcher-clone/backend && npm run dev
# http://localhost:3000
```

## 아키텍처
- **Framework**: Fastify v5 + TypeScript
- **ORM**: Drizzle ORM + better-sqlite3
- **Auth**: Google OAuth 2.0 PKCE + JWT
- **Validation**: Zod
- **DB**: SQLite (`data/keychron.db`)

## 핵심 파일
| 파일 | 용도 |
|------|------|
| `src/index.ts` | 서버 진입점, 라우트 등록 |
| `src/db/schema.ts` | DB 테이블 정의 (6 테이블) |
| `src/db/seed.ts` | 시드 데이터 |
| `src/config/env.ts` | 환경변수 검증 |
| `src/modules/auth/` | Google OAuth + JWT (패턴 참고) |
| `src/modules/profiles/` | 프로필 CRUD (패턴 참고) |
| `src/modules/definitions/` | 키보드 정의 로더 |
| `src/via-bridge/` | VIA 프로토콜 상수/헬퍼 |
| `src/shared/errors.ts` | NotFoundError, ForbiddenError |

## 코드 컨벤션
- TypeScript strict mode
- 라우트: `src/modules/{name}/{name}.routes.ts`
- 서비스: `src/modules/{name}/{name}.service.ts`
- 검증: 라우트에서 Zod 스키마
- 에러: `shared/errors.ts`의 에러 클래스 사용
- Auth: `requireAuth` 미들웨어로 보호

## 현재 DB 스키마
| 테이블 | 설명 |
|--------|------|
| users | Google OAuth 사용자 |
| keyboards | 지원 키보드 모델 |
| firmware_versions | 펌웨어 릴리즈 |
| profiles | 키보드 설정 (로컬 + 클라우드) |
| feedback | 사용자 피드백 |
| settings | 앱 설정 (KV) |

## 미구현 항목
- feedback 라우트 (스키마만 존재)
- settings 라우트 (스키마만 존재)
- firmware 다운로드/업데이트 라우트
- 커뮤니티 기능 (공개 프로필, 좋아요, 팔로우)
- 입력 검증 미들웨어 (대부분 라우트)
- 레이트 리미팅
- 테스트 (tests/ 디렉토리 비어있음)

## 세션 완료 후
Notion "Session Handoff Log"에 기록:
- 새/수정 API 엔드포인트 (method, path, request, response)
- 스키마 변경 (새 테이블, 컬럼)
- 추가된 의존성
- 마이그레이션 명령
