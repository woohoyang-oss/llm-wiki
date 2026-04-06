# Claude Team 구축 진행상황 (2026-03-23 11:00 업데이트)

## 현재 배치
| 봇 | 머신 | 계정 | 상태 |
|-----|------|------|------|
| team-lead | 맥북 (M1 16GB) | Max | ✅ 실행 중 |
| project-leader | 맥북 (M1 16GB) | Max (opus) | ✅ 실행 중 |
| coder | 맥미니 (M4 24GB) | 엔터프라이즈 | ✅ 실행 중 |
| reviewer | 맥미니 (M4 24GB) | 엔터프라이즈 | ✅ 실행 중 |
| tester | 맥미니 (M4 24GB) | 엔터프라이즈 | ✅ 실행 중 |

## 인프라
- Slack: 5개 App (Team Lead, Coder, Reviewer, Tester, Project Leader)
- 통신: Slack Socket Mode → 브릿지(bridge.py) → Claude CLI
- 대시보드: http://localhost:5555 (맥북)
- 세션 유지: --resume으로 스레드 단위 대화 맥락 유지
- 브랜치: feat/claude-team (GitHub 푸시 완료)

## 완료된 과제 (27건)

### 긴급(🔴) — 5건
1. ✅ config.py 시크릿 기본값 제거
2. ✅ Gmail 토큰 Fernet 암호화 (EncryptedText TypeDecorator)
3. ✅ 승인 DB 커밋 누락 수정 (admin.py 3곳)
4. ✅ 시드 파일 텔레그램 토큰 환경변수 분리
5. ✅ Gmail Rate Limit 처리 (exponential backoff 5회 재시도)

### 보강(🟡) — 8건
6. ✅ SLA 모니터링 배치 2개 (check_sla_breach, check_stale_pending)
7. ✅ 추가 룰 5개 시드 데이터 (B2B견적, 에이퍼, 포스윙, 배송조회, 재입고)
8. ✅ 승인 워크플로우 개선 (edit_and_send, reject_reason, edited 재승인)
9. ✅ 시드 데이터 보강 (브랜드 서명 4개, 영문 고지문, 에이퍼/포스윙 메일박스)
10. ✅ LLM 라우팅 수정 (claude-cli is_active=true)
11. ✅ 룰 액션 실행 (tag, set_priority 구현)
12. ✅ 메일 검색 다중 키워드 (공백 분리 AND 매칭)
13. ✅ AutomationExecutor 구현 (3 액션 + Celery Beat 5분)

### 중장기(🟢) — 4건
14. ✅ Email ScopeMixin 분석 → 적용 불필요 판단 (assignee 기반 권장)
15. ✅ CI/CD GitHub Actions (.github/workflows/ci.yml)
16. ✅ Pub/Sub Watch 자동 등록/갱신 설계
17. ✅ 벡터 검색 기본 구현 (embedding_service + pgvector search API)

### 테스트
- 62/62 전체 통과 (3회 실행, 매번 통과)
- test_rule_engine.py (40개), test_ai_processor.py (13개), test_email_parsing.py (9개)

### 분석/설계 (구현 보류)
- 템플릿 변수 치환 (render 함수 구현 완료)
- Pub/Sub Watch 자동 등록/갱신 (Celery 태스크 설계)
- 벡터 임베딩 자동 생성 (문서 저장 시 Celery 비동기 설계)

## 별도 터미널 버그수정 8건 (v2-phase0-implementation 브랜치)
1. ✅ 403 로그아웃 버그 → 401만 로그아웃
2. ✅ 메일 가져오기 20→100건
3. ✅ /gmail/fetch 권한 완화
4. ✅ Agent 공용메일함 빈 화면 수정
5. ✅ HTML 이메일 렌더링 → iframe 격리
6. ✅ 챗봇 관리 기능 14개 액션 추가
7. ✅ Ruff Lint CI 수정 (130개 자동수정)
8. ✅ 사이드바 트리구조 복원

## 미완료
- ⬜ git 히스토리 토큰 제거 (BFG — 양대표님 수동)
- ⬜ 태그/담당자 룰 추가
- ⬜ 견적 관련 레포/템플릿 분석
- ⬜ 모바일 UI (Stream J)
- ⬜ 모니터링 (Prometheus/Grafana)

## 배포
- EC2: auto-email-poc (i-0745490e05c7e0259, t3.small, 50GB)
- 배포 방식: main push → GitHub Actions → rsync → EC2
- 롤백: EC2 스냅샷 + git revert
- 스냅샷 가능 확인 완료 (vol-0dff8e880ea6b9d45)

## 코드 위치
- 브릿지: auto-email/claude-team/bridge.py
- 에이전트 MD: auto-email/.claude/agents/ (5개)
- 대시보드: auto-email/claude-team/dashboard/
- 환경변수: auto-email/.env.claude-team
- 진척도: auto-email/claude-team/PROGRESS.md