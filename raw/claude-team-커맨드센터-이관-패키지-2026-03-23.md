# 커맨드센터 이관 패키지 (2026-03-23)

## 프로젝트 개요
- **프로젝트**: auto-email — AI 기반 이메일 자동 응답 시스템
- **서버**: https://mail.tbe.kr (EC2: 3.34.66.166)
- **GitHub**: https://github.com/woohoyang-oss/auto-email (private)
- **버전**: v2.8.0
- **프로덕션 준비도**: 97%

## 인프라
| 항목 | 값 |
|------|------|
| EC2 | i-0745490e05c7e0259, t3.small, 50GB |
| Docker | 6개 컨테이너: api, admin, worker, beat, db, redis |
| admin 포트 | 3001:3000 (nginx 80 → 3001 프록시) |
| DB | PostgreSQL 16 + pgvector |
| AWS 프로필 | tbnws-ec2 (맥스튜 ~/.aws/credentials) |

## 팀 구조
| 역할 | 위치 | 소통 |
|------|------|------|
| TL (전략 어드바이저) | 맥북 | Slack |
| PL (프로젝트 리더) | 맥북 | Slack |
| Coder | 맥미니 | Slack |
| Reviewer | 맥미니 | Slack |
| Tester (Slack) | 맥미니 | Slack |
| Tester (아카샤) | 맥스튜 | 아카샤 폴링 |
| 커맨드센터 | 맥스튜 | 여기 |

## 배포 규칙
1. EC2 수동 수정 금지 → 항상 main 코드 수정 후 푸시
2. main push → CI 자동 → Deploy 자동 (GitHub Actions)
3. 새 패키지 → Dockerfile 확인 (torch는 CPU-only)
4. 새 환경변수 → ci.yml env + .env.example 동기화
5. docker-compose 서비스명 변경 → deploy.yml 동기화
6. 선택적 의존성 → try/except 감싸기

## 오늘 완료 (30+건)
- 버그픽스: api-client 403, pipeline Rule.id, RuleEngine 싱글턴, approvals reject_reason
- 보안: X-Admin-Key 제거, Rate Limiting (slowapi+Redis+폴백)
- 안정성: Celery time_limit, Docker CPU-only torch, EC2 디스크 50GB 정리
- CI/CD: ruff 수정, 테스트 101개, deploy.yml 서비스명, 자동배포 성공
- guide: 11개 섹션 완성 (챗봇/메일박스/지식베이스/자동화/배정규칙/SLA/스케줄)
- 문서: production-readiness, deployment-checklist, operations-runbook, deployment-process, ci-guide, lessons-learned
- PL 자율 수정 7건: Celery time_limit 전체, XSS, sent_at, .env.example 등

## 남은 과제
- EC2 Gmail 토큰 마이그레이션 검증
- 메일박스 기능 개선 (OAuth 연동 검토 중)
- 전체 25페이지 브라우저 테스트 완료
- EC2 IAM Role 설정 (스냅샷 자동화)
- gap 분석 문서 최신화

## 핵심 파일
- app/api/admin.py — 모든 API (5000+줄)
- app/main.py — FastAPI 앱
- app/core/config.py — Settings
- app/models/encrypted_types.py — Fernet 암호화
- .github/workflows/ci.yml — CI
- .github/workflows/deploy.yml — 자동 배포
- docker-compose.yml — 포트 3001:3000
- admin/src/app/guide/page.tsx — 사용설명서
