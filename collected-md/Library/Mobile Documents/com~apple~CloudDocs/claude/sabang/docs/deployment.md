# 배포 가이드

## 현재 배포
- **URL**: https://tbe.kr/sb/
- **서버**: AWS EC2 (ap-northeast-2), Amazon Linux 2023
- **접속**: `ssh ec2-user@tbe.kr`
- **Flask**: 포트 5050, 경로 `/home/ec2-user/sabang-dashboard/`
- **Nginx**: `/sb/` → `127.0.0.1:5050` reverse proxy

## 자동 배포 (deploy.sh)

```bash
# 기본 (현재 서버)
./deploy.sh

# 커스텀 서버
./deploy.sh [호스트] [사용자] [비밀번호] [URL접두사]

# 예시
./deploy.sh tbe.kr ec2-user ec2-user sb
./deploy.sh new-server.com ubuntu mypass dashboard
```

### deploy.sh가 하는 일
1. 원격 디렉토리 생성
2. 파일 업로드 (app.py, db.py, notify.py, index.html, soldout.py)
3. 시작 스크립트 생성
4. Flask 앱 (재)시작
5. 접속 확인

### deploy.sh가 하지 않는 일
- Nginx 설정 추가 (최초 1회 수동 필요)
- SSL 인증서 설정

## Nginx 설정 (최초 1회)

서버의 Nginx 설정 파일에 아래 location 블록 추가:

```nginx
# /etc/nginx/conf.d/tbe.conf (또는 해당 서버 설정)
# 기존 설정 내 server { } 블록 안에 추가

    # Sabang Dashboard → port 5050
    location /sb/ {
        proxy_pass http://127.0.0.1:5050/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
        proxy_buffering off;
    }
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 서버 이전 체크리스트

1. **Python 3.9+ 설치 확인**: `python3 --version`
2. **pip 패키지 설치**: `pip3 install flask requests`
3. **Nginx 설치 및 설정**: 위 location 블록 추가
4. **SSL (선택)**: Let's Encrypt certbot
5. **배포 실행**: `./deploy.sh 새서버 사용자 비밀번호 sb`
6. **접속 확인**: `curl https://새서버/sb/`

## 서버 관리

```bash
# 로그 확인
ssh ec2-user@tbe.kr 'tail -50 /home/ec2-user/sabang-dashboard/app.log'

# 재시작
ssh ec2-user@tbe.kr 'bash /home/ec2-user/sabang-dashboard/start.sh'

# 프로세스 확인
ssh ec2-user@tbe.kr 'ps aux | grep sabang-dashboard'

# 중지
ssh ec2-user@tbe.kr 'pkill -f sabang-dashboard/app.py'
```

## 디렉토리 구조 (서버)

```
/home/ec2-user/sabang-dashboard/
├── app.py              # Flask 메인 서버
├── db.py               # SQLite DB 모듈
├── notify.py           # Telegram/Email 알림
├── sabangnet.db        # SQLite DB (자동 생성)
├── app.log             # 서버 로그
├── start.sh            # 시작 스크립트
├── skills/
│   └── soldout.py      # 사방넷 자동화 모듈
└── templates/
    └── index.html      # 프론트엔드
```

## 서브패스 배포 원리

- **Nginx**: `location /sb/` → `proxy_pass http://127.0.0.1:5050/;` (trailing slash로 /sb/ strip)
- **Frontend**: `API_BASE = window.location.pathname` 자동 감지 → `/sb/api/...`로 요청
- **Backend**: Flask는 `/api/...`로 수신 (Nginx가 /sb/ 제거)
- **결과**: 어떤 서브패스(/sb/, /dashboard/, /app/ 등)에서도 무설정 동작
