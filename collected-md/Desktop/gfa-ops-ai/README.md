# GFA Ops AI — POC

네이버 GFA 광고 운영 자동화 AI Proof of Concept.

## 구조

```
gfa-ops-ai/
├── gfa_session.py      # GFA 세션 관리 (쿠키 + XSRF)
├── gfa_manager.py      # 캠페인 관리 (예산/ON-OFF) — API 발견 후 구현
├── discover_api.py     # Phase 1: API 엔드포인트 탐색 도구
├── poc_test.py         # 세션 테스트 + 캠페인 목록 조회
└── data/
    └── sessions/       # 세션 쿠키 파일 (tbnws-ad 심볼릭 링크)
```

## 사용법

```bash
cd ~/Desktop/gfa-ops-ai
source venv/bin/activate

# 세션 테스트
python poc_test.py

# API 탐색
python discover_api.py
```

## 핵심 원칙

- tbnws-ad에 영향 없음 (세션 파일만 읽기 공유)
- GFA 내부 웹 API 사용 (gfa.naver.com/apis/gfa/v1/...)
- requests + session cookie + XSRF 토큰 방식
