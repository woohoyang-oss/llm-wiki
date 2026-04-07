# Architecture & Key Files

## Directory Structure
```
openclaw-sabang/
├── dashboard/
│   ├── app.py              # Flask 메인 서버 (port 5050)
│   ├── db.py               # SQLite DB 모듈 (thread-safe, WAL mode)
│   ├── notify.py           # Telegram/Email 알림 모듈
│   ├── sabangnet.db        # SQLite DB (자동 생성, git 제외)
│   └── templates/
│       └── index.html      # 전체 프론트엔드 (싱글페이지)
├── skills/
│   └── soldout.py          # 사방넷 자동화 모듈
├── docs/                   # 프로젝트 문서
├── deploy.sh               # 원격 서버 배포 스크립트
└── CLAUDE.md               # 프로젝트 컨텍스트
```

## Backend: app.py
- Flask dev server: `0.0.0.0:5050`, debug=on
- 환경변수: `SABANG_PASS`, `GOG_KEYRING_PASSWORD`
- 스케줄러: 60초 간격 폴링
- SKILL_DIR: 상대경로 자동 감지 (`../skills` → `./skills` fallback)

### API Endpoints
- `GET /` - 메인 페이지
- `GET /api/batch/configs` - 배치 설정 목록
- `POST /api/batch/configs` - 배치 설정 생성
- `PUT /api/batch/configs/<id>` - 배치 설정 수정
- `DELETE /api/batch/configs/<id>` - 배치 설정 삭제
- `POST /api/batch/toggle/<id>` - 배치 토글
- `POST /api/batch/run/<id>` - 배치 실행
- `GET /api/batch/history` - 배치 실행 이력
- `GET /api/batch/history/<id>` - 배치 이력 상세
- `POST /api/batch/rollback/<id>` - 배치 롤백
- `POST /api/snapshot/refresh` - 스냅샷 새로고침
- `GET /api/snapshot/status` - 스냅샷 상태
- `GET /api/products` - 상품 목록 (페이지네이션, 검색, 필터)
  - params: page, page_size, q, status, shma, include_no_gcode
- `GET /api/products/stats` - 상품 상태 통계
- `GET /api/products/<gcode>` - G-코드별 상세
- `GET /api/preview/<bid>` - 배치 미리보기
- `POST /api/task` - 비동기 태스크 생성
- `GET /api/task/<id>` - 태스크 상태 폴링
- `GET /api/task/<id>/logs` - 태스크 로그 스트리밍
- `GET /api/tasks` - 전체 태스크 목록
- `POST /api/notify/test` - 알림 테스트
- `GET /api/telegram/chat_id` - Telegram chat_id 확인

## Frontend: index.html
- 싱글 HTML 파일 (CSS + JS 인라인)
- htmx 없이 순수 fetch API 사용
- Tailwind CSS CDN (유틸리티만) + CSS Variables (테마)
- **6탭 구조**: 사용 방법 | 소스데이터 관리 | 소스 미리보기 | 배치 관리 | 수동 관리 | 이력 조회
- **기본 탭**: `sources` (소스데이터 관리), URL `?tab=` 파라미터로 탭 직접 접근 가능
- **이력 조회 서브탭**: 📋 이력 조회 | ⚠ 송신실패 분석 (pill 토글)
- **API_BASE**: `window.location.pathname` 기반 자동 감지 (서브패스 배포 지원)

### Theme System
- CSS Variables: ~50개 (`:root` light, `[data-theme="dark"]` dark)
- FOUC 방지: `<script>` 태그가 CSS 앞에서 localStorage 확인
- localStorage key: `sabang-theme` (값: `dark` 또는 없음)
- 토글: 헤더 오른쪽 sun/moon SVG 아이콘

### Semantic CSS Classes
- `.card` - 카드 컨테이너
- `.btn-primary`, `.btn-ghost`, `.btn-danger`, `.btn-sm` - 버튼
- `.form-input`, `.form-label` - 폼 요소
- `.modal-overlay`, `.modal-panel` - 모달
- `.tab-btn` - 탭 네비게이션
- `.products-table` - 테이블 (table-layout: fixed)
- `.status-badge` + `.status-002/003/001/004` - 상태 배지
- `.page-btn` - 페이지네이션 버튼
- `.toggle-switch/.toggle-slider` - 토글 스위치

## Database: db.py
- SQLite WAL mode, thread-safe (threading.local() per-thread connections)
- Path: `dashboard/sabangnet.db` (자동 생성)

### Tables
- `products` - 상품 데이터
  - gcode, prd_nm, modl_nm, brnd_nm, orgpl_ntn, status_cd, status_nm, sepr, shma_nm, ...
- `product_status_summary` - 상태 요약
- `batch_configs` - 배치 설정 (2 rows: 재입고 복구, 품절 처리)
- `batch_history` - 배치 실행 이력
- `snapshot_meta` - 스냅샷 메타
- `tasks` - 비동기 태스크 상태

## Automation Core: skills/soldout.py
- **위치**: 저장소 `skills/` 디렉토리 (이전: `~/.openclaw/workspace/skills/sabang-soldout/`)
- 사방넷 API: `getMallProductUpdateLists` with pagination
  - `fetch_all_product_statuses()` — requests 기반 직접 API 호출 (Scrapling 불필요)
  - `find_gcodes_by_stock()` — 시트 기반 재고 조건 필터링
  - `run_with_scrapling()` — 상태 변경 실행 (Scrapling/Playwright 필요)
- 배치 실행(상태 변경)은 scrapling venv 필요, 스냅샷/조회는 requests만으로 동작

## Server Start Command
```bash
cd dashboard
SABANG_PASS='@tbnws1245' GOG_KEYRING_PASSWORD='mailbot2026' python3 app.py
```

## Remote Deployment
```bash
./deploy.sh tbe.kr ec2-user ec2-user sb
```
→ 상세: [deployment.md](deployment.md)
