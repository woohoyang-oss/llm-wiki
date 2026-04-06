---
title: CI/CD 파이프라인 및 서버 운영
tags: [ci-cd, deploy, github-actions, ec2, docker, snapshot]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-team-배포-재발-방지-조치.md, 서버-정상유무-체크리스트-2026-03-26.md, ec2-snapshot-method.md]
---
# CI/CD 파이프라인 및 서버 운영

## 개요

[[auto-email]] 프로젝트의 배포 프로세스, 서버 점검 체크리스트, EC2 스냅샷 관리 방법을 정리한다. GitHub Actions 기반 자동 배포와 Docker 컨테이너 운영을 중심으로 구성되어 있다.

## 배포 프로세스 (GitHub Actions)

GitHub Actions가 새 코드를 rsync로 EC2에 동기화한 뒤 docker restart를 수행한다. 이 과정에서 EC2에서 수동으로 수정한 내용이 덮어씌워지는 문제가 발생한 바 있다.

### 재발 방지 원칙
- **EC2에서 수동 수정 금지** -- 반드시 main 코드를 수정하여 푸시
- 수동 수정은 다음 Deploy에서 원복되어 서버 다운 유발
- 주요 수정 사례: `logging.py` try/except 래핑, `main.py` sentry 설정 폴백, `encrypted_types.py` plaintext 폴백

## 서버 점검 체크리스트

배포 후 반드시 수행해야 하는 6단계 검증 절차이다.

1. **컨테이너 상태**: `docker ps`로 6개 컨테이너(admin, api, worker, beat, db, redis) 전부 Up 확인
2. **HTTP 응답**: mail.tbe.kr 200, /admin/dashboard 200 또는 401 확인
3. **프론트 렌더링**: HTML title 태그 정상 출력 확인
4. **Server Action 에러**: admin 로그에서 "Failed to find Server Action" 확인 -- 발견 시 `--no-cache` 리빌드
5. **API 에러**: api 로그에서 traceback/500 에러 확인
6. **메일 폴링**: beat 로그에서 "Sending due task" 정상 출력 확인

### 검증 주체
- [[claude-team]] Tester: 배포 후 1~6번 전체 검증
- Reviewer: 코드 리뷰 시 Server Action 호환성 확인
- 커맨드센터: /loop 폴링으로 HTTP 모니터링

## EC2 스냅샷

AWS CLI(`tbnws-ec2` 프로필, ap-northeast-2)를 사용하여 EBS 볼륨 스냅샷을 생성한다.

### 주요 인스턴스
| 이름 | 용도 |
|------|------|
| auto-email-poc | 메인 서비스 |
| akaxa-space | 아카샤 서비스 |
| claude | Claude 관련 서비스 |

스냅샷 생성 절차: 인스턴스/볼륨 조회 → `create-snapshot` 명령 실행 → 태그 지정으로 관리한다.

## 관련 소스
- [claude-team-배포-재발-방지-조치.md](../../raw/claude-team-배포-재발-방지-조치.md)
- [서버-정상유무-체크리스트-2026-03-26.md](../../raw/서버-정상유무-체크리스트-2026-03-26.md)
- [ec2-snapshot-method.md](../../raw/ec2-snapshot-method.md)
