# 챗봇 구현 가이드 (2026-03-24)

## 1. 시스템 프롬프트 위치 및 구조

### 위치: `app/api/admin.py` 라인 6441~6718
- 변수명: `CHATBOT_SYSTEM_PROMPT`
- 총 278줄의 상세 프롬프트
- 사용처: 라인 8524에서 `full_system = CHATBOT_SYSTEM_PROMPT + "\n\n" + user_context`

### 프롬프트 구조 (섹션 순서):
1. 역할 정의 + 할 수 있는 것 (6441~6458)
2. 담당자 배정 관리 — [action] 태그 사용법 (6460~6485)
3. 텔레그램 연동 — 상태 A/B/C 분기 (6487~6527)
4. 노션 연동 — 순서 제한 (6529~6563)
5. 개인 설정 변경 (6565~6572)
6. 알림 채널 관리 (6574~6583)
7. 지식 베이스 관리 (6585~6597)
8. 템플릿 & 매크로 관리 (6599~6612)
9. 룰 & 자동화 관리 + 자동응답 대화 플로우 (6614~6646)
10. 운영 현황 조회 (6650~6678)
11. 좌측 메뉴 기능 안내 — 네비게이션 (6680~6696)
12. 응답 규칙 (6698~6705)
13. 토큰/비밀정보 처리 규칙 (6706~6718)

## 2. 액션 실행 함수

### 위치: `_execute_chatbot_action()` — 라인 6822~7635
- 총 약 814줄
- 마지막 분기: `get_email_detail` (7618)
- 기본 반환: `"⚠️ 알 수 없는 액션: {action_type}"` (7635)

### 현재 액션 목록 (36종):

| 액션 | 라인 | 카테고리 | 권한 |
|------|------|---------|------|
| assign_responsibility | 6836 | 담당자 | admin |
| remove_responsibility | 6872 | 담당자 | admin |
| telegram_set_bot | 6899 | 텔레그램 | all |
| telegram_status | 6979 | 텔레그램 | all |
| telegram_generate_link | 7001 | 텔레그램 | all |
| telegram_settings | 7032 | 텔레그램 | all |
| telegram_disconnect | 7048 | 텔레그램 | all |
| notion_set_token | 7061 | 노션 | all |
| notion_test | 7093 | 노션 | all |
| notion_list_databases | 7110 | 노션 | all |
| notion_set_default_db | 7136 | 노션 | all |
| get_personal_settings | 7149 | 개인설정 | all |
| update_personal_settings | 7163 | 개인설정 | all |
| list_notification_channels | 7184 | 알림 | all |
| create_notification_channel | 7192 | 알림 | admin |
| delete_notification_channel | 7215 | 알림 | admin |
| send_telegram_test | 7228 | 텔레그램 | all |
| list_repositories | 7256 | 지식 | all |
| list_knowledge_docs | 7264 | 지식 | all |
| create_knowledge_doc | 7286 | 지식 | all |
| list_templates | 7317 | 템플릿 | all |
| create_template | 7334 | 템플릿 | all |
| list_macros | 7363 | 매크로 | all |
| create_macro | 7382 | 매크로 | all |
| list_rules | 7405 | 규칙 | all |
| toggle_rule | 7418 | 규칙 | all |
| list_assignment_rules | 7436 | 배정 | all |
| list_sla_policies | 7449 | SLA | all |
| list_business_hours | 7465 | 영업시간 | all |
| list_schedules | 7478 | 스케줄 | all |
| toggle_schedule | 7493 | 스케줄 | all |
| list_signatures | 7509 | 서명 | all |
| list_disclaimers | 7522 | 면책 | all |
| list_mailboxes | 7533 | 메일함 | all |
| search_emails | 7542 | 이메일 | all |
| list_agents | 7569 | 에이전트 | all |
| create_rule | 7579 | 규칙 | all |
| get_email_detail | 7618 | 이메일 | all |

## 3. 새 액션 추가 수정 포인트

### Step 1: 시스템 프롬프트에 설명 추가
- 위치: 라인 6614~6678 부근 (해당 카테고리 섹션에)
- 형식:
```
### 새기능 설명:
[action]{"type":"new_action","param1":"value1"}[/action]
```

### Step 2: `_execute_chatbot_action()`에 elif 분기 추가
- 위치: 라인 7635 직전 (마지막 `return` 직전)
- 형식:
```python
elif action_type == "new_action":
    param1 = action_data.get("param1")
    # 로직...
    return "결과 메시지"
```

### Step 3: admin 전용이면 권한 체크 추가
- 라인 6827의 `admin_only_actions` set에 추가

### 필요한 import/모델
- DB 모델: `from app.models.base import ...` (admin.py 상단에 이미 import됨)
- 외부 API 호출: `import httpx` (함수 내부에서 lazy import)

## 4. 커맨드 센터 (라인 7639~)
- `COMMAND_PATTERNS` dict: 사용자 메시지 → 커맨드 매핑
- 간단한 키워드 매칭으로 빠른 응답 제공
- 새 커맨드 추가 시 여기에도 매핑 추가 가능
