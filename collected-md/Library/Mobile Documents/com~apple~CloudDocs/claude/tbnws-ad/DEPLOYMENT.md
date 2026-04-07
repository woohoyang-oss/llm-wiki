# 배포 및 히스토리 관리 규칙

## 필수 절차

**모든 git commit + 서버 배포 시** 아래 절차를 반드시 수행합니다.

### 1. changelog.json 업데이트

`app/static/data/changelog.json` 배열 **맨 앞**에 새 항목을 추가합니다.

```json
{
  "version": "X.Y.Z",
  "date": "YYYY-MM-DD",
  "title": "사용자 관점의 변경 제목 (한글)",
  "type": "feature|fix|security|improvement|data|refactor",
  "commits": ["abc1234"],
  "changes": [
    "사용자가 이해할 수 있는 변경 내용 1",
    "변경 내용 2"
  ]
}
```

### 2. 버전 규칙 (Semver)

| 변경 유형 | 버전 증가 | 예시 |
|-----------|-----------|------|
| 새 기능 | minor (Y++) | 2.10.0 → 2.11.0 |
| 버그 수정 / 개선 | patch (Z++) | 2.11.0 → 2.11.1 |
| 대규모 변경 | major (X++) | 2.11.0 → 3.0.0 |

### 3. type 분류

| type | 사용 시점 |
|------|-----------|
| `feature` | 새 탭, 새 기능, 새 API |
| `fix` | 버그 수정, 레이아웃 수정 |
| `security` | 보안 관련 변경 |
| `improvement` | 기존 기능 개선, UX 향상 |
| `data` | 데이터 수집/배치/백필 |
| `refactor` | 코드 구조 변경 (기능 동일) |

### 4. changes 작성 규칙

- **사용자 관점**으로 작성 (개발 용어 지양)
- "~기능 추가", "~오류 수정", "~성능 개선" 형태
- 3~7개 항목 권장

---

## 배포 순서

```
1. 코드 변경 완료
2. changelog.json 업데이트 (위 규칙 준수)
3. git add + git commit
4. git push origin main
5. SCP로 EC2 서버에 파일 배포
6. (Python 변경 시) Flask 재시작
```

## 배포 명령어

```bash
# SSH 접속 패턴 (반드시 이 옵션들 사용!)
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr

# 정적 파일만 (JS/CSS/HTML)
SSH_AUTH_SOCK="" scp -o IdentitiesOnly=yes -i ~/Desktop/claude.pem app/static/ ec2-user@tbe.kr:/home/ec2-user/naver-ad/app/static/

# Python 파일 포함 시
SSH_AUTH_SOCK="" scp -o IdentitiesOnly=yes -i ~/Desktop/claude.pem <파일경로> ec2-user@tbe.kr:/home/ec2-user/naver-ad/<파일경로>

# Flask 재시작 (Python 변경 시에만) — systemd 사용!
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr "sudo systemctl restart tbnws-ad.service"

# 전체 서비스 상태 확인
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@tbe.kr "sudo systemctl is-active tbnws-ad sabang-dashboard tbnws-data-api tobe-ai-v2"
```

## systemd 서비스 목록

| 서비스 | 포트 | 재시작 명령 |
|--------|------|-------------|
| `tbnws-ad` | 8087 | `sudo systemctl restart tbnws-ad.service` |
| `sabang-dashboard` | 5050 | `sudo systemctl restart sabang-dashboard.service` |
| `tbnws-data-api` | 8100 | `sudo systemctl restart tbnws-data-api.service` |
| `tobe-ai-v2` | 3100 | `sudo systemctl restart tobe-ai-v2.service` |

> **주의**: 절대 `pkill -f python`이나 `kill $(pgrep -f ...)` 사용 금지 — 다른 서비스 죽일 수 있음
