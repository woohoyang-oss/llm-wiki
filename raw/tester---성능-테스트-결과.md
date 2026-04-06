# Auto Email 버그수정 로그 (2026-03-23)

브랜치: v2-phase0-implementation
GitHub: https://github.com/woohoyang-oss/auto-email

## 수정 항목

1. **api-client.ts 403 로그아웃 버그** — 403 응답 시 토큰 삭제+로그아웃 → 401만 로그아웃, 403은 에러 메시지만
2. **메일 가져오기 100건** — fetchGmail.mutate(20) → mutate(100) (emails/page.tsx, system/page.tsx)
3. **/gmail/fetch 권한 완화** — verify_admin_role(admin만) → verify_admin(모든 인증 사용자)
4. **Agent 공용메일함 빈 화면** — agent 로그인 시 ?assignee=userId 자동필터 제거
5. **HTML 이메일 렌더링 깨짐** — dangerouslySetInnerHTML → iframe srcDoc 격리 렌더링
6. **챗봇 관리 기능 확장** — 14개 액션 추가 (update_agent_role, create_agent, toggle_agent_active, create/delete/list_agent_group, list/run/toggle_schedule, list/toggle_rule, list/toggle_assignment_rule, list_mailboxes)
7. **Ruff Lint CI 수정** — 130개 자동수정 + ruff.toml 추가 (E712,E501,N806 등 무시)
8. **사이드바 트리구조 복원** — 내 메일함(7폴더), 공용 메일함(5폴더), 메일쓰기 버튼, Gmail 라벨 카운트

## 핵심 파일
- admin/src/lib/api-client.ts
- admin/src/app/emails/page.tsx
- admin/src/app/system/page.tsx
- admin/src/components/app-sidebar.tsx
- app/api/admin.py
- ruff.toml
