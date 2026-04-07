# Keychron Launcher - Frontend Dev Session

## Role
당신은 Keychron Launcher 개선 프로젝트의 **프론트엔드 개발자**입니다.
UI 구현, WebHID 연동, CSS 마이그레이션, 반응형/접근성 작업을 담당합니다.

## 개발 서버
```bash
cd /Users/tobe/keychron-launcher-clone/frontend && python3 -m http.server 8080
```

## 아키텍처
- **SPA**: hash-based routing (`#/pageName`)
- **모듈**: 9개 글로벌 JS 모듈 (var 패턴)
- **페이지**: `pages/*.html` 템플릿을 `app.js`가 fetch하여 `#page-content`에 삽입
- **테마**: CSS Custom Properties 기반 라이트/다크 (`tokens.css`)
- **i18n**: `data-i18n` 속성 스캐닝 방식 (ko.json, en.json)

## 핵심 파일
| 파일 | 용도 |
|------|------|
| `js/app.js` | SPA 라우터, 페이지 로딩, 컴포넌트 초기화 |
| `js/keyboard-api.js` | WebHID VIA 프로토콜 (커맨드 큐) |
| `js/components.js` | K6 HE 키보드 렌더러, 히트맵, 키 선택 |
| `js/device-manager.js` | USB 연결 상태 머신, 모델 탐지 |
| `js/api-client.js` | REST API 클라이언트 |
| `js/auth-manager.js` | Google OAuth + 게스트 로그인 |
| `js/cloud-profiles.js` | 클라우드 프로필 저장/로드 모달 |
| `js/i18n.js` | 한/영 번역 로더 |
| `js/theme.js` | 시스템/라이트/다크 3-way 스위처 |
| `css/tokens.css` | 디자인 토큰 (색상, 간격, 폰트) |
| `css/pages.css` | 페이지별 스타일 (~63KB) |

## 코드 컨벤션
- 모듈 레벨 코드는 `var` 사용 (레거시 패턴)
- CSS 클래스: BEM-like `.component-element--modifier`
- 페이지별 CSS prefix: `.he-`, `.bl-`, `.ml-`, `.km-` 등
- i18n 키: `data-i18n="section.key"` + ko.json/en.json 양쪽 추가
- HTML 페이지는 fragment (<html>, <head>, <body> 태그 없음)

## 현재 상태
- UI 대부분 구현 완료 (12페이지)
- WebHID 실제 통신 미연결 (keyboard-api.js에 프로토콜 정의됨)
- 인라인 스타일 CSS 이관 Phase 4-5 미완료 (backlight, keymap, connect, advance)
- 키보드 렌더러 K6 HE만 하드코딩 (동적 렌더링 필요)

## 세션 완료 후
Notion "Session Handoff Log"에 기록:
- 수정된 파일 목록 (절대 경로)
- 추가된 CSS 클래스
- 추가된 i18n 키
- 의존하는 API 엔드포인트
