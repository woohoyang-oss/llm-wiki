# 배포 시 서버 다운 재발 방지

## 문제
GitHub Actions Deploy가 새 코드를 rsync → docker restart 하면서 수동 수정이 덮어씌워짐.
EC2에서 수동으로 고친 것(logging.py, main.py sentry)이 다음 배포 시 원복 → 서버 다운.

## 근본 해결
main 코드 자체를 안전하게 수정:
1. app/core/logging.py — structlog configure를 try/except로
2. app/main.py — settings.sentry_dsn → getattr(settings, "sentry_dsn", "")
3. app/models/encrypted_types.py — ALLOW_PLAINTEXT_FALLBACK 폴백

## 원칙
EC2에서 수동 수정하지 말고, 항상 main 코드를 수정해서 푸시.
수동 수정은 다음 Deploy에서 덮어씌워짐.
