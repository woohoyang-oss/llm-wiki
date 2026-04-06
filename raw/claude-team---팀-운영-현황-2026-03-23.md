# Claude Team 운영 현황 (2026-03-23)

## 머신 배치
| 머신 | 역할 |
|------|------|
| 맥북 | TL (전략 어드바이저) + PL (프로젝트 리더) 브릿지 |
| 맥미니 | Coder + Reviewer + Tester 브릿지 |
| 맥스튜 | 커맨드센터 (총괄 지휘, 브릿지 안 돌림) |

## 역할 변경
- TL: 기존 팀장 → **전략 어드바이저 + 대시보드/상태 관리**
  - PL 계획 검증, 리스크 선제 발견, 사용자 영향도 분석
  - 대시보드 실행(localhost:5555) + 수정사항 발생 시 상태 업데이트
  - 부족한 기획/계획을 계속 고민해서 문서화

## 자율 운영 파이프라인
```
PL: 과제 발굴 → 팀원 지시 → 결과 확인 → 커맨드센터 보고
TL: PL 검증 + 리스크 분석 + 대시보드 관리 → 커맨드센터 보고
Coder/Reviewer/Tester: PL 지시 수행
커맨드센터: 보고 수신 + 의사결정
```

## 중요 발견: 봇 멘션 규칙
- TL 봇 토큰으로 TL에게 멘션하면 자기 메시지로 인식 → 무시됨
- TL에게 지시할 때는 PL 봇 토큰으로 보내야 함
- 반대로 PL에게 지시할 때는 TL 봇 토큰으로 보내면 됨

## 현재 진행 상태
- PL: 로드맵 작성 완료 (알파→베타→정식), 팀원 직접 지시 중
- TL: 대시보드 + 전략 분석 시작
- Coder: CI ruff 수정 완료 (e5b15d1), CI 재확인 중
- Reviewer: 머지 분석 완료 (48파일, +3631라인, 충돌 없음)
- Tester: pytest 실행 + 커버리지 분석 중

## 배포 차단 3건 → 모두 수정 완료
1. Gmail 토큰 Fernet 암호화 마이그레이션 스크립트
2. edit_and_send 후 email.status = replied
3. 벡터 검색 SQL 인젝션 → 바인드 파라미터

## 봇 토큰 정리
- TL: xoxb-...tie3 (user: U0AN3FEMS6N)
- PL: xoxb-...IhR (user: U0AN0EWHXJ9)
- Coder: xoxb-...PwzF (user: U0ANXPNJWAC)
- Reviewer: xoxb-...AKJe (user: U0AMN2KKML7)
- Tester: xoxb-...I6D9 (user: U0AN73VFGSY)
- Channel: C0AN03UNL0M
