# Sabang Dashboard - Project Context

## Project Overview
사방넷(Sabangnet) 상품상태 관리 자동화 대시보드. Flask 웹 UI.
- **목적**: 사방넷 쇼핑몰 상품 상태(공급중/일시중지/완전품절 등) 자동 변경
- **핵심 기능**: 배치(batch) 기반 상품상태 일괄 변경, 스냅샷, 상품 조회

## Quick Start (새 환경 셋업)
→ 상세: [docs/setup-guide.md](docs/setup-guide.md)

## Key Files & Architecture
→ 상세: [docs/architecture.md](docs/architecture.md)

## Credentials & Environment
→ 상세: [docs/credentials.md](docs/credentials.md)

## UI/UX Design System
→ 상세: [docs/ui-design.md](docs/ui-design.md)

## Database Schema
→ 상세: [docs/database.md](docs/database.md)

## Deployment
→ 상세: [docs/deployment.md](docs/deployment.md)

## Current State (2026-03-08)
- 라이트/다크 테마 시스템 완성 (CSS Variables, localStorage 저장)
- **6탭 UI**: 사용 방법 | 소스데이터 관리 | 소스 미리보기 | 배치 관리 | 수동 관리 | 이력 조회
- **기본 탭**: `sources` (소스데이터 관리)
- **글로벌 소스 상태바**: 헤더 아래에 활성 소스명 + 스냅샷 시간 표시
- **소스 미리보기 탭**: 활성 소스 재고 + 사방넷 스냅샷 교차 조회 (검색/필터/페이지네이션)
  - 상태 필터: `primary_status` (가장 많은 쇼핑몰의 상태) 기준
  - **재고 소스 선택**: `source_override` 파라미터로 활성 소스 또는 DB(ERP) 선택 가능
  - 행 클릭 → 상품 상세 모달
- **스냅샷 배치 스케줄러**: `batch_type=snapshot` 지원, 매일 09:00 자동 실행
  - 배치 카드 정렬: 일반 배치 상단, 스냅샷 배치 하단
  - 스냅샷 카드 전용 UI (📸 아이콘, 지금 실행 버튼)
- **이력 조회 탭**: 통합 이력 + 송신실패 분석 서브탭
  - **서브탭 구조**: 📋 이력 조회 | ⚠ 송신실패 분석 (뱃지 카운트)
  - **이력 테이블**: 구분 태그에 좌측 상태 색상 바 통합 (done=green, error=red)
  - **행 펼침**: 클릭 → 메타 정보 바 (유형·목표·시작·소요) + 결과 요약 + G-코드 상세 + 실행 로그
  - **결과 포맷**: 콤팩트 (`93/160`, `559/923`) + nowrap
  - **송신실패 분석**: 누적 방식 (limit=500 별도 API 호출), 쇼핑몰별 실패 카드 + 사유별 그룹핑
- **진행중인 작업 아이콘**: 헤더에 회전 화살표 아이콘 + 뱃지 (5초 폴링)
  - 클릭 → 모달: 진행중 작업 + 최근 완료 표시
  - 카드 클릭 → 로그 펼침 (실행중이면 2초 폴링)
  - 에러 메시지 인라인 표시 (예: "⚠ 사방넷 로그인 실패")
- **이메일 알림**: 모든 배치/수동 작업 완료 시 salesteam@tbnws.com 자동 발송
  - `gog` CLI (Gmail OAuth) 기반, 발신: ai.login@tbnws.com
  - Telegram 알림도 병행 가능
- **서버 시작 시 stuck 작업 정리**: queued/running → error 자동 전환
- DB: SQLite, 서버 시작 시 자동 생성 + 배치 설정 seed
  - `batch_type` 컬럼: 자동 마이그레이션 (batch_run / snapshot)
- 스냅샷 새로고침으로 상품 데이터 로드 (사방넷 API → DB)
- **서버 배포**: tbe.kr/sb/ (Nginx reverse proxy → Flask :5050)
- **서버 시작**: `start.sh` 사용 필수 (SABANG_PASS 환경변수 설정됨)
- **데이터 소스 이중 구조**: 배치는 ERP DB 사용(1,469 G-codes), 소스 미리보기는 활성 소스(시트/DB) 선택 가능

## Key Architecture Decisions
- `app.py` SKILL_DIR: 상대경로 자동 감지 (`../skills` 또는 `./skills`)
- `index.html` API_BASE: `window.location.pathname` 기반 자동 감지 (서브패스 배포 지원)
- `soldout.py`: 사방넷 API 직접 호출 (requests 기반, Scrapling 불필요)
- **작업 실행**: `start_task()` → 즉시 스레드 생성 (실제 큐 없음, "queued"는 과도 상태)
- **동시성**: 현재 Lock/Semaphore 없음 — 동시 실행 시 사방넷 세션 충돌 가능
- **stdout 캡처**: `_stdout_lock`으로 보호하지만 동시 실행 시 로그 교차 오염 가능

## Server Deployment
- **서버 경로**: `/home/ec2-user/sabang-dashboard/`
- **배포 커맨드**: `sshpass -p 'ec2-user' scp -o StrictHostKeyChecking=no <local> ec2-user@tbe.kr:<remote>`
- **app.py**: `/home/ec2-user/sabang-dashboard/app.py` (루트 레벨)
- **index.html**: `/home/ec2-user/sabang-dashboard/templates/index.html`
- **서버 시작/재시작**: `cd /home/ec2-user/sabang-dashboard && bash start.sh`
- **주의**: `nohup python3 app.py` 직접 실행 금지 → SABANG_PASS 누락으로 사방넷 로그인 실패

## Known Issues
- Chrome localhost 접속 불가 → `192.168.1.215:5050` 사용
- G-codes with leading spaces: 2건 (minor)
- 동시 배치 실행 시 사방넷 세션 충돌 위험 (Semaphore 미적용)
- `pkill -f 'app.py --port 5050'`는 SSH 세션도 종료시킴 → 별도 SSH 호출 필요
