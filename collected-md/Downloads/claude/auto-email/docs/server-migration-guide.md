# 서버 마이그레이션 가이드

> 현재 서버(3.34.66.166)에서 새 서버로 전체 이전하는 절차

## 현재 서버 현황

| 항목 | 값 |
|------|------|
| 서버 | AWS EC2 t3.small (ap-northeast-2) |
| IP | 3.34.66.166 |
| OS | Amazon Linux 2023 |
| 디스크 | 50GB (gp3), 사용 37GB |
| 메모리 | 2GB |
| DB 크기 | ~12MB |
| 프로젝트 경로 | `~/auto-email/` |
| Docker 볼륨 | `auto-email_pgdata` (PostgreSQL 데이터) |

## 마이그레이션 대상

| 항목 | 경로/이름 | 필수 | 비고 |
|------|-----------|------|------|
| 소스 코드 | `~/auto-email/` | O | GitHub에서 clone 가능 |
| .env | `~/auto-email/.env` | O | 비밀키, API 키 포함 |
| credentials | `~/auto-email/credentials/` | O | Gmail OAuth 토큰 |
| PostgreSQL 전체 | Docker 볼륨 `auto-email_pgdata` | O | 36 테이블, 12MB |
| knowledge | `~/auto-email/knowledge/` | O | GitHub에 포함 |
| Docker 이미지 | 빌드 시 자동 생성 | X | 새 서버에서 재빌드 |
| Redis | 캐시/큐 데이터 | X | 휘발성, 이전 불필요 |

---

## 방법 1: 마이그레이션 스크립트 (권장)

프로젝트에 포함된 스크립트로 원클릭 백업/복원:

### Step 1. 구서버에서 백업

```bash
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 \
  "cd ~/auto-email && bash scripts/migrate-export.sh"
```

### Step 2. 로컬로 다운로드

```bash
scp -i ~/.ssh/auto-email-poc.pem \
  ec2-user@3.34.66.166:~/auto-email-backup.tar.gz \
  ./auto-email-backup.tar.gz
```

### Step 3. 새 서버에 업로드 + 복원

```bash
# 업로드
scp -i ~/.ssh/NEW-KEY.pem \
  ./auto-email-backup.tar.gz \
  NEW-USER@NEW-IP:~/auto-email-backup.tar.gz

# 복원 실행
ssh -i ~/.ssh/NEW-KEY.pem NEW-USER@NEW-IP \
  "cd ~/auto-email && bash scripts/migrate-import.sh"
```

### Step 4. 검증

```bash
ssh -i ~/.ssh/NEW-KEY.pem NEW-USER@NEW-IP \
  "cd ~/auto-email && bash scripts/migrate-verify.sh"
```

---

## 방법 2: 수동 마이그레이션

### 2-1. 구서버: DB 덤프

```bash
# pg_dump (custom format, 압축)
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 << 'EOF'
docker exec esbot-db pg_dump -U bot -d email_support_bot \
  --no-owner --no-privileges -F custom -f /tmp/esbot-full.dump
docker cp esbot-db:/tmp/esbot-full.dump ~/esbot-full.dump
ls -lh ~/esbot-full.dump
EOF

# 로컬로 다운로드
scp -i ~/.ssh/auto-email-poc.pem \
  ec2-user@3.34.66.166:~/esbot-full.dump ./esbot-full.dump
```

### 2-2. 구서버: 설정 파일 백업

```bash
scp -i ~/.ssh/auto-email-poc.pem \
  ec2-user@3.34.66.166:~/auto-email/.env ./.env.backup

scp -i ~/.ssh/auto-email-poc.pem -r \
  ec2-user@3.34.66.166:~/auto-email/credentials/ ./credentials-backup/
```

### 2-3. 새 서버: 기본 환경 구성

```bash
# Amazon Linux 2023
sudo dnf update -y
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

# Docker Compose v2 plugin
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m) \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# 재로그인 (docker 그룹 반영)
exit
```

### 2-4. 새 서버: 프로젝트 배포

```bash
git clone https://github.com/woohoyang-oss/auto-email.git ~/auto-email
cd ~/auto-email
git checkout v2-phase0-implementation

# .env 복원 + 필요 시 수정 (IP, 도메인 등)
cp /path/to/.env.backup ~/auto-email/.env
nano ~/auto-email/.env

# credentials 복원
cp -r /path/to/credentials-backup/ ~/auto-email/credentials/
```

### 2-5. 새 서버: DB 먼저 시작 + 복원

```bash
cd ~/auto-email

# DB + Redis 먼저 시작
docker compose up -d db redis

# healthy 대기
until docker exec esbot-db pg_isready -U bot -d email_support_bot 2>/dev/null; do
  echo "waiting for DB..."; sleep 2
done

# pgvector 확장 생성
docker exec esbot-db psql -U bot -d email_support_bot \
  -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 덤프 복원
docker cp ./esbot-full.dump esbot-db:/tmp/esbot-full.dump
docker exec esbot-db pg_restore -U bot -d email_support_bot \
  --no-owner --no-privileges --clean --if-exists \
  /tmp/esbot-full.dump

# 복원 결과 확인
docker exec esbot-db psql -U bot -d email_support_bot \
  -c "SELECT count(*) as tables FROM information_schema.tables WHERE table_schema='public';"
```

### 2-6. 새 서버: 전체 서비스 시작

```bash
docker compose build
docker compose up -d

# 상태 확인
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

---

## 방법 3: Docker 볼륨 통째 복사 (가장 빠름)

DB 덤프 없이 PostgreSQL 데이터 디렉토리를 직접 복사. 같은 pgvector:pg16 이미지 사용 시에만 안전.

```bash
# ── 구서버: 볼륨 tar ──
ssh -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166 << 'EOF'
cd ~/auto-email
docker compose stop db
docker run --rm \
  -v auto-email_pgdata:/data \
  -v ~/:/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /data .
docker compose start db
ls -lh ~/pgdata-backup.tar.gz
EOF

# ── 로컬 경유 전송 ──
scp -i ~/.ssh/auto-email-poc.pem ec2-user@3.34.66.166:~/pgdata-backup.tar.gz .
scp -i ~/.ssh/NEW-KEY.pem ./pgdata-backup.tar.gz NEW-USER@NEW-IP:~/

# ── 새 서버: 볼륨 복원 ──
ssh -i ~/.ssh/NEW-KEY.pem NEW-USER@NEW-IP << 'EOF'
cd ~/auto-email
docker compose up -d db          # 볼륨 자동 생성
docker compose stop db
docker run --rm \
  -v auto-email_pgdata:/data \
  -v ~/:/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/pgdata-backup.tar.gz -C /data"
docker compose up -d
EOF
```

---

## DNS / 도메인 전환

| 시나리오 | 작업 |
|----------|------|
| 같은 AWS 리전, Elastic IP 재연결 | EC2 콘솔에서 EIP를 새 인스턴스에 연결 → DNS 변경 불필요 |
| 새 IP 사용 | Route53 또는 DNS에서 `mail.tbe.kr` A 레코드 변경 |
| 도메인 변경 | .env의 `ADMIN_ALLOWED_ORIGINS`, `GOOGLE_OAUTH_REDIRECT_URI`, `CORS_ORIGINS` 수정 |
| Google OAuth | Google Cloud Console → OAuth 클라이언트 → Redirect URI 업데이트 |
| Gmail Pub/Sub | Push endpoint URL 업데이트 |

---

## 마이그레이션 체크리스트

### 사전 준비
- [ ] 새 서버 프로비저닝 (최소 t3.small 2GB, 50GB gp3)
- [ ] SSH 키 설정
- [ ] Docker + Docker Compose 설치
- [ ] 보안그룹: 80, 8000, 22 포트 오픈

### 데이터 이전
- [ ] PostgreSQL 덤프 생성 + 다운로드
- [ ] .env 파일 복사
- [ ] credentials/ 디렉토리 복사
- [ ] knowledge/ 파일 확인 (GitHub clone에 포함)

### 새 서버 설정
- [ ] GitHub clone + `v2-phase0-implementation` 브랜치 checkout
- [ ] .env 배치 및 수정 (IP/도메인 변경 사항)
- [ ] credentials/ 복원
- [ ] DB 복원 + 데이터 검증
- [ ] `docker compose build && docker compose up -d`

### 검증
- [ ] 6개 컨테이너 정상 가동 (esbot-db, esbot-redis, esbot-api, esbot-admin, esbot-worker, esbot-beat)
- [ ] `curl http://localhost:8000/health` 정상
- [ ] DB 테이블 수 (36개) + 주요 레코드 수 일치
- [ ] 프론트엔드 접속 + 로그인 확인
- [ ] Celery Beat 로그에서 이메일 폴링 동작 확인
- [ ] 텔레그램 봇 응답 확인

### DNS 전환
- [ ] Elastic IP 재연결 또는 DNS A 레코드 변경
- [ ] Google OAuth redirect URI 업데이트 (도메인 변경 시)
- [ ] SSL 인증서 설정 (HTTPS 필요 시)

### 구서버 정리
- [ ] 구서버 서비스 중지 (`docker compose down`)
- [ ] 최소 1주 대기 후 인스턴스 종료
- [ ] 필요 시 AMI 스냅샷 보관
