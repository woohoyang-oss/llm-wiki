# Phase 1: 자동 파이프라인 + 스케줄 관리 — 상세 개발 명세서 v2

> 목표: 메일 수신 → AI 분석 → 자동발송/승인대기가 완전 자동으로 동작
> 갱신: 2026-03-13 — 코드 재점검 반영

---

## 0. 사용자 워크플로우 (Before → After)

### Before (현재)
```
1. Admin이 Admin UI에서 "공용 메일함" 페이지 열기
2. [메일 가져오기] 버튼 수동 클릭 → Gmail API 호출
3. 새 메일 목록 표시
4. 메일 선택 → [AI 자동작성] 버튼 수동 클릭
5. AI 응답 확인 → [발송] 버튼 수동 클릭
※ Celery Worker가 꺼져있으면 전혀 자동화 안됨
```

### After (구현 후)
```
1. Admin이 "스케줄 관리" 페이지에서 폴링 주기 설정 (예: 2분)
2. 시스템이 자동으로:
   a. 2분마다 각 메일박스 폴링
   b. 새 메일 → AI 분석 자동 실행
   c. confidence ≥ 0.85 → 자동 발송 + Telegram 알림
   d. 0.70~0.84 → 승인큐 + 담당자 알림
   e. < 0.70 → 에스컬레이션 + 관리자 알림
3. Admin은 승인 대기 알림 확인 → 승인/수정/거부만 처리
4. "스케줄 관리"에서 주기 변경, 일시정지, 즉시실행 가능
```

---

## 1. 현재 코드 정밀 분석

### 1.1 이미 구현된 것 (정확한 위치)

| 구성요소 | 파일:라인 | 현재 동작 |
|---------|----------|----------|
| 폴링 태스크 | `tasks.py:88-135` | `check_new_emails()` — 모든 메일박스 일괄, maxResults=100 |
| Pub/Sub 태스크 | `tasks.py:30-85` | `process_gmail_notification()` — History API 사용 |
| AI 처리 태스크 | `tasks.py:138-170` | `process_email_task()` → `pipeline.process_message()` |
| Beat 스케줄 | `celery_app.py:27-32` | `poll-new-emails` 1개, crontab(minute="*/5") |
| 파이프라인 | `pipeline.py:69-249` | 전체 플로우 구현 완료 |
| Confidence 분기 | `pipeline.py:214-244` | CONFIDENCE_AUTO_SEND=0.85, CONFIDENCE_APPROVAL=0.70 |

### 1.2 구현 시 반드시 해결해야 할 문제

#### 문제 1: GmailService가 멀티 메일박스 미지원

```python
# gmail_service.py:38 — 현재 생성자
class GmailService:
    def __init__(self):
        # settings.gmail_delegated_user 하나만 사용
```

→ 메일박스별 폴링하려면 GmailService가 주소 파라미터를 받아야 함

#### 문제 2: sync_session 미존재

```python
# database.py — 현재: async_session만 있음
# config.py:16 — database_url_sync 정의되어 있으나 사용 안됨
database_url_sync: str = "postgresql://bot:bot_password@localhost:5432/email_support_bot"
```

→ DatabaseScheduler가 Celery Beat (동기 프로세스)에서 동작하므로 sync 세션 필수

#### 문제 3: check_new_emails가 단일 Gmail 인스턴스 사용

```python
# tasks.py:108 — 현재
gmail = GmailService()  # 파라미터 없음, settings에서 단일 계정만
```

→ 메일박스별 독립 폴링 불가

---

## 2. 구현 태스크 목록 (작업 단위)

### Task 1-1: DB 스키마 + 모델 (병렬 가능: Backend)

**새 테이블: `celery_schedules`**

```sql
CREATE TABLE celery_schedules (
    id              SERIAL PRIMARY KEY,
    schedule_id     VARCHAR UNIQUE NOT NULL,
    name            VARCHAR NOT NULL,
    description     VARCHAR,
    task_name       VARCHAR NOT NULL,
    schedule_type   VARCHAR NOT NULL DEFAULT 'crontab',
    cron_expression VARCHAR,
    interval_seconds INTEGER,
    task_args       JSONB DEFAULT '[]'::jsonb,
    task_kwargs     JSONB DEFAULT '{}'::jsonb,
    queue           VARCHAR DEFAULT 'email_polling',
    is_active       BOOLEAN DEFAULT true,
    last_run_at     TIMESTAMPTZ,
    total_run_count INTEGER DEFAULT 0,
    last_result     VARCHAR,
    last_error      TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
```

**기존 테이블 변경: `mailbox_configs`**

```sql
ALTER TABLE mailbox_configs ADD COLUMN poll_interval_minutes INTEGER DEFAULT 5;
ALTER TABLE mailbox_configs ADD COLUMN last_polled_at TIMESTAMPTZ;
ALTER TABLE mailbox_configs ADD COLUMN last_history_id VARCHAR;
```

**모델 추가** — `app/models/base.py`에:

```python
class CelerySchedule(Base):
    __tablename__ = "celery_schedules"
    id: Mapped[int] = mapped_column(primary_key=True)
    schedule_id: Mapped[str] = mapped_column(String, unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String, nullable=False)
    description: Mapped[str | None] = mapped_column(String)
    task_name: Mapped[str] = mapped_column(String, nullable=False)
    schedule_type: Mapped[str] = mapped_column(String, default="crontab")
    cron_expression: Mapped[str | None] = mapped_column(String)
    interval_seconds: Mapped[int | None] = mapped_column(Integer)
    task_args: Mapped[list] = mapped_column(JSONB, default=list)
    task_kwargs: Mapped[dict] = mapped_column(JSONB, default=dict)
    queue: Mapped[str] = mapped_column(String, default="email_polling")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    last_run_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    total_run_count: Mapped[int] = mapped_column(Integer, default=0)
    last_result: Mapped[str | None] = mapped_column(String)
    last_error: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

**MailboxConfig 확장** — 기존 모델(base.py:74-89)에 추가:

```python
# 기존 필드 뒤에 추가
poll_interval_minutes: Mapped[int] = mapped_column(Integer, default=5)
last_polled_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
last_history_id: Mapped[str | None] = mapped_column(String)
```

**seed_data.py 수정** — SEED_MAP(line 33-42)에 추가:

```python
("celery_schedules.json", CelerySchedule, "schedule_id"),
```

---

### Task 1-2: sync_session 추가 (병렬 가능: Backend)

**파일**: `app/core/database.py` — 기존 코드 아래에 추가

```python
# ── 동기 세션 (Celery Beat DatabaseScheduler용) ──
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

sync_engine = create_engine(
    settings.database_url_sync,  # config.py:16에 이미 정의됨
    pool_size=2,
    max_overflow=0,
)

SyncSessionLocal = sessionmaker(sync_engine, class_=Session)

def get_sync_session():
    """동기 세션 컨텍스트"""
    session = SyncSessionLocal()
    try:
        yield session
    finally:
        session.close()
```

---

### Task 1-3: GmailService 멀티 메일박스 지원 (병렬 가능: Backend)

**파일**: `app/services/gmail_service.py` — 생성자 수정

```python
# 현재 (line 38):
class GmailService:
    def __init__(self):
        self._service = None
        self._delegated_user = settings.gmail_delegated_user

# 수정 후:
class GmailService:
    def __init__(self, delegated_user: str | None = None):
        self._service = None
        self._delegated_user = delegated_user or settings.gmail_delegated_user
```

**영향 범위**: GmailService를 생성하는 모든 곳 확인 필요
- `tasks.py:108` — `GmailService()` → `GmailService(mailbox_address)` 변경
- `pipeline.py` — 기존 호출 확인
- `admin.py` — `/gmail/fetch` 엔드포인트 확인

**주의**: Service Account의 Domain-wide Delegation이 모든 메일박스 주소에 대해 권한이 있어야 함. 없으면 단일 계정(ai@tbe.kr)으로 폴백하고 `original_to` 헤더로 메일박스 구분.

---

### Task 1-4: DatabaseScheduler (병렬 가능: Backend)

**파일**: `app/workers/db_scheduler.py` (신규)

```python
"""
DB 기반 Celery Beat 스케줄러
- celery_schedules 테이블에서 활성 스케줄 로드
- 30초마다 DB 체크 → 변경 감지 시 스케줄 갱신
- Celery Beat 프로세스에서 실행 (동기)
"""

import time
import logging

from celery.beat import Scheduler, ScheduleEntry
from celery.schedules import crontab as celery_crontab
from celery.schedules import schedule as celery_interval
from sqlalchemy import select

from app.core.database import SyncSessionLocal
from app.models.base import CelerySchedule

logger = logging.getLogger(__name__)


def parse_cron(expr: str) -> celery_crontab | None:
    """'*/5 * * * *' → celery crontab 객체"""
    parts = expr.strip().split()
    if len(parts) != 5:
        logger.warning(f"Invalid cron expression: {expr}")
        return None
    return celery_crontab(
        minute=parts[0],
        hour=parts[1],
        day_of_month=parts[2],
        month_of_year=parts[3],
        day_of_week=parts[4],
    )


class DatabaseScheduler(Scheduler):
    """DB에서 스케줄을 읽어 Beat에 동적 적용"""

    UPDATE_INTERVAL = 30  # DB 체크 주기 (초)

    def __init__(self, *args, **kwargs):
        self._last_update = 0
        self._schedule_dict = {}
        Scheduler.__init__(self, *args, **kwargs)
        self.max_interval = 10  # Beat 루프 최대 간격

    def setup_schedule(self):
        """초기 로딩 — Beat 시작 시 1회 호출"""
        self._update_from_db()
        logger.info(f"DatabaseScheduler: {len(self._schedule_dict)}개 스케줄 로드")

    def _update_from_db(self):
        """DB에서 활성 스케줄 조회 → ScheduleEntry 변환"""
        try:
            session = SyncSessionLocal()
            try:
                rows = session.execute(
                    select(CelerySchedule).where(CelerySchedule.is_active == True)
                ).scalars().all()
            finally:
                session.close()
        except Exception as e:
            logger.error(f"DB 조회 실패: {e}")
            return  # 기존 스케줄 유지

        new_schedule = {}
        for row in rows:
            # 스케줄 타입별 파싱
            if row.schedule_type == "crontab" and row.cron_expression:
                sched = parse_cron(row.cron_expression)
                if not sched:
                    continue
            elif row.schedule_type == "interval" and row.interval_seconds:
                sched = celery_interval(run_every=row.interval_seconds)
            else:
                logger.warning(f"스케줄 '{row.schedule_id}' 설정 불완전, 건너뜀")
                continue

            entry = self.Entry(
                name=row.schedule_id,
                task=row.task_name,
                schedule=sched,
                args=tuple(row.task_args or []),
                kwargs=row.task_kwargs or {},
                options={"queue": row.queue},
                app=self.app,
            )
            new_schedule[row.schedule_id] = entry

        if new_schedule != self._schedule_dict:
            added = set(new_schedule) - set(self._schedule_dict)
            removed = set(self._schedule_dict) - set(new_schedule)
            if added:
                logger.info(f"스케줄 추가: {added}")
            if removed:
                logger.info(f"스케줄 제거: {removed}")
            self._schedule_dict = new_schedule

    @property
    def schedule(self):
        return self._schedule_dict

    @schedule.setter
    def schedule(self, value):
        """Celery 내부에서 schedule을 set하려 할 때 무시"""
        pass

    def tick(self):
        """주기적으로 DB 변경 감지"""
        now = time.monotonic()
        if now - self._last_update > self.UPDATE_INTERVAL:
            self._update_from_db()
            self._last_update = now
        return super().tick()

    def close(self):
        """종료 시 정리"""
        self._schedule_dict = {}

    @property
    def info(self):
        return f"DatabaseScheduler: {len(self._schedule_dict)} schedules"
```

---

### Task 1-5: 메일박스별 폴링 태스크 (Task 1-3 의존)

**파일**: `app/workers/tasks.py` — 기존 check_new_emails 아래에 추가

```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def check_mailbox_emails(self, mailbox_address: str):
    """특정 메일박스의 새 메일 확인

    Args:
        mailbox_address: 메일박스 이메일 주소 (예: support@keychron.kr)
    """
    try:
        result = _run_async(_check_single_mailbox(mailbox_address))
        return result
    except Exception as exc:
        logger.exception(f"메일박스 폴링 실패: {mailbox_address}")
        self.retry(exc=exc)


async def _check_single_mailbox(mailbox_address: str) -> dict:
    """단일 메일박스 폴링"""
    async with async_session() as db:
        # 1. 메일박스 설정 조회
        result = await db.execute(
            select(MailboxConfig).where(
                MailboxConfig.address == mailbox_address,
                MailboxConfig.is_active == True,
            )
        )
        mailbox = result.scalar_one_or_none()
        if not mailbox:
            return {"status": "skip", "reason": f"inactive or not found: {mailbox_address}"}

        # 2. Gmail API — 해당 메일박스의 계정으로 폴링
        #    Service Account delegation으로 해당 주소의 메일에 접근
        #    또는 공용 계정(ai@tbe.kr)에서 to: 필터로 조회
        try:
            gmail = GmailService(delegated_user=mailbox_address)
            messages = gmail.list_unread_messages(max_results=100)
        except Exception:
            # delegation 실패 시 공용 계정 + to: 필터 폴백
            gmail = GmailService()  # 기본 계정
            messages = gmail.list_unread_messages(
                max_results=100,
                query=f"to:{mailbox_address}",
            )

        # 3. 이미 처리된 메일 제외 + 신규 큐잉
        new_count = 0
        for msg in messages:
            msg_id = msg["id"]
            existing = await db.execute(
                select(Email.id).where(Email.message_id == msg_id)
            )
            if existing.scalar():
                continue

            process_email_task.apply_async(
                args=[msg_id],
                queue="email_processing",
            )
            new_count += 1

        # 4. 폴링 시각 갱신
        from sqlalchemy import update
        await db.execute(
            update(MailboxConfig)
            .where(MailboxConfig.id == mailbox.id)
            .values(last_polled_at=func.now())
        )
        await db.commit()

        return {"mailbox": mailbox_address, "new_emails": new_count}
```

**GmailService.list_unread_messages 수정 필요** — `query` 파라미터 추가:

```python
# gmail_service.py — list_unread_messages 수정
def list_unread_messages(self, max_results: int = 100, query: str | None = None) -> list[dict]:
    service = self._get_service()
    params = {"userId": "me", "labelIds": ["INBOX"], "maxResults": max_results}
    if query:
        params["q"] = query
    results = service.users().messages().list(**params).execute()
    return results.get("messages", [])
```

**celery_app.py task_routes 추가** (line 21-25):

```python
task_routes={
    "app.workers.tasks.process_email_task": {"queue": "email_processing"},
    "app.workers.tasks.process_gmail_notification": {"queue": "email_processing"},
    "app.workers.tasks.check_new_emails": {"queue": "email_polling"},
    "app.workers.tasks.check_mailbox_emails": {"queue": "email_polling"},  # 추가
},
```

---

### Task 1-6: 스케줄 서비스 (Task 1-1 의존)

**파일**: `app/services/schedule_service.py` (신규)

```python
"""스케줄 관리 서비스"""

from sqlalchemy import select, update, func
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.base import CelerySchedule, MailboxConfig


class ScheduleService:

    @staticmethod
    async def create_mailbox_schedule(db: AsyncSession, mailbox: MailboxConfig) -> CelerySchedule | None:
        """메일박스 생성/활성화 시 폴링 스케줄 자동 생성"""
        schedule_id = f"poll-{mailbox.brand}-{mailbox.category}"
        interval = mailbox.poll_interval_minutes or 5

        # 중복 체크
        existing = await db.execute(
            select(CelerySchedule).where(CelerySchedule.schedule_id == schedule_id)
        )
        if existing.scalar_one_or_none():
            return None

        schedule = CelerySchedule(
            schedule_id=schedule_id,
            name=f"{mailbox.display_name} 메일 폴링",
            description=f"{mailbox.address} 메일함 {interval}분 주기 확인",
            task_name="app.workers.tasks.check_mailbox_emails",
            schedule_type="crontab",
            cron_expression=f"*/{interval} * * * *",
            task_args=[mailbox.address],
            queue="email_polling",
            is_active=mailbox.is_active,
        )
        db.add(schedule)
        return schedule

    @staticmethod
    async def sync_all_mailbox_schedules(db: AsyncSession) -> int:
        """모든 활성 메일박스에 대해 스케줄 동기화, 생성된 개수 반환"""
        result = await db.execute(
            select(MailboxConfig).where(MailboxConfig.is_active == True)
        )
        mailboxes = result.scalars().all()
        created = 0
        for mb in mailboxes:
            s = await ScheduleService.create_mailbox_schedule(db, mb)
            if s:
                created += 1
        return created

    @staticmethod
    async def update_schedule_interval(db: AsyncSession, schedule_id: str, cron_expr: str):
        """스케줄 주기 변경"""
        await db.execute(
            update(CelerySchedule)
            .where(CelerySchedule.schedule_id == schedule_id)
            .values(cron_expression=cron_expr, updated_at=func.now())
        )

    @staticmethod
    async def record_run(db: AsyncSession, schedule_id: str, success: bool, error: str | None = None):
        """실행 결과 기록"""
        await db.execute(
            update(CelerySchedule)
            .where(CelerySchedule.schedule_id == schedule_id)
            .values(
                last_run_at=func.now(),
                total_run_count=CelerySchedule.total_run_count + 1,
                last_result="success" if success else "error",
                last_error=error if not success else None,
            )
        )
```

---

### Task 1-7: Admin API 엔드포인트 (Task 1-1, 1-6 의존)

**파일**: `app/api/admin.py` — 기존 라우터에 추가

7개 엔드포인트:

| Method | Path | 기능 |
|--------|------|------|
| GET | `/admin/schedules` | 스케줄 목록 (is_active, last_run_at 포함) |
| POST | `/admin/schedules` | 스케줄 생성 (cron_expression 유효성 검증) |
| PUT | `/admin/schedules/{schedule_id}` | 스케줄 수정 |
| DELETE | `/admin/schedules/{schedule_id}` | 스케줄 삭제 |
| POST | `/admin/schedules/{schedule_id}/toggle` | 활성/비활성 토글 |
| POST | `/admin/schedules/{schedule_id}/run-now` | 즉시 실행 (Celery send_task) |
| POST | `/admin/schedules/sync-mailboxes` | 메일박스 기반 스케줄 일괄 생성 |

**Pydantic 모델**:

```python
class ScheduleCreateRequest(BaseModel):
    schedule_id: str
    name: str
    description: str | None = None
    task_name: str = "app.workers.tasks.check_mailbox_emails"
    schedule_type: str = "crontab"
    cron_expression: str | None = None
    interval_seconds: int | None = None
    task_args: list = []
    task_kwargs: dict = {}
    queue: str = "email_polling"
    is_active: bool = True

    @model_validator(mode="after")
    def validate_schedule(self):
        if self.schedule_type == "crontab" and not self.cron_expression:
            raise ValueError("crontab 타입은 cron_expression 필수")
        if self.schedule_type == "interval" and not self.interval_seconds:
            raise ValueError("interval 타입은 interval_seconds 필수")
        if self.cron_expression:
            parts = self.cron_expression.strip().split()
            if len(parts) != 5:
                raise ValueError("cron_expression은 5필드 형식이어야 합니다 (분 시 일 월 요일)")
        return self

class ScheduleUpdateRequest(BaseModel):
    name: str | None = None
    description: str | None = None
    cron_expression: str | None = None
    interval_seconds: int | None = None
    task_args: list | None = None
    task_kwargs: dict | None = None
    queue: str | None = None
    is_active: bool | None = None
```

**MailboxCreateRequest 확장** (기존 line 1023-1033에 추가):

```python
class MailboxCreateRequest(BaseModel):
    # ... 기존 필드 ...
    poll_interval_minutes: int = 5  # 추가
```

**메일박스 생성 시 스케줄 자동 생성** — `POST /admin/mailboxes` (line 1056) 수정:

```python
@router.post("/mailboxes")
async def create_mailbox(req: MailboxCreateRequest, db=Depends(get_db)):
    mailbox = MailboxConfig(**req.model_dump())
    db.add(mailbox)
    await db.flush()
    # 폴링 스케줄 자동 생성
    from app.services.schedule_service import ScheduleService
    await ScheduleService.create_mailbox_schedule(db, mailbox)
    await db.commit()
    return {"status": "created", "id": mailbox.id}
```

---

### Task 1-8: celery_app.py 수정 (Task 1-4 의존)

**파일**: `app/workers/celery_app.py`

변경사항:
1. `beat_schedule` 딕셔너리 **제거** (DB에서 관리)
2. `check_mailbox_emails` 태스크 라우팅 추가
3. 주석에 Beat 실행 명령 추가

```python
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Seoul",
    enable_utc=True,
    task_routes={
        "app.workers.tasks.process_email_task": {"queue": "email_processing"},
        "app.workers.tasks.process_gmail_notification": {"queue": "email_processing"},
        "app.workers.tasks.check_new_emails": {"queue": "email_polling"},
        "app.workers.tasks.check_mailbox_emails": {"queue": "email_polling"},
    },
    # beat_schedule 삭제됨 — DB 기반 스케줄러 사용
    # Beat 실행: celery -A app.workers.celery_app beat -S app.workers.db_scheduler.DatabaseScheduler
    worker_concurrency=4,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
)
```

---

### Task 1-9: Admin UI 스케줄 페이지 (병렬 가능: Frontend)

**파일**: `admin/src/app/schedules/page.tsx` (신규)

**사이드바 추가** — `admin/src/components/app-sidebar.tsx` (line 79 부근, "자동화 설정" 그룹에):

```typescript
{ label: "스케줄 관리", href: "/schedules", icon: Timer },
```

**페이지 기능**:

1. **스케줄 목록 테이블**
   - 컬럼: ID, 이름, 태스크, 주기(cron 표현식), 상태(활성/비활성/에러), 마지막 실행, 실행 횟수
   - 행 액션: [▶ 즉시실행] [⏸ 토글] [✏️ 수정] [🗑 삭제]

2. **생성/수정 모달**
   - schedule_id (생성 시만, 수정 불가)
   - 이름, 설명
   - 태스크 선택 드롭다운: check_new_emails, check_mailbox_emails
   - 스케줄 타입: Crontab / Interval 라디오
   - Cron 표현식 입력 + 프리셋 버튼: [2분] [5분] [10분] [30분] [1시간] [매일 9시]
   - 큐 이름 (기본: email_polling)
   - 태스크 인자 JSON 에디터

3. **[메일박스 동기화]** 버튼
   - POST /admin/schedules/sync-mailboxes 호출
   - 결과: "3개 스케줄 생성됨" 토스트

**TypeScript 타입**:

```typescript
interface CelerySchedule {
  id: number;
  schedule_id: string;
  name: string;
  description: string | null;
  task_name: string;
  schedule_type: "crontab" | "interval";
  cron_expression: string | null;
  interval_seconds: number | null;
  task_args: any[];
  task_kwargs: Record<string, any>;
  queue: string;
  is_active: boolean;
  last_run_at: string | null;
  total_run_count: number;
  last_result: "success" | "error" | "timeout" | null;
  last_error: string | null;
}
```

---

### Task 1-10: docker-compose.yml + dev.sh (병렬 가능: DevOps)

**docker-compose.yml** — beat 서비스 command 수정:

```yaml
beat:
  command: celery -A app.workers.celery_app beat -S app.workers.db_scheduler.DatabaseScheduler --loglevel=info
```

**scripts/dev.sh** (신규):

```bash
#!/bin/bash
set -e

echo "=== Email Support Bot 개발서버 ==="

docker compose up -d postgres redis
sleep 2

# DB 초기화 (첫 실행 시)
python -m scripts.seed_data 2>/dev/null || true

uvicorn app.main:app --reload --port 8000 &
celery -A app.workers.celery_app worker --loglevel=info -Q email_processing,email_polling --concurrency=2 &
celery -A app.workers.celery_app beat -S app.workers.db_scheduler.DatabaseScheduler --loglevel=info &

echo "API: http://localhost:8000 | Admin: cd admin && npm run dev"
trap "kill 0" EXIT
wait
```

**seed-data/celery_schedules.json** (신규):

```json
[
  {
    "schedule_id": "poll-all-mailboxes",
    "name": "전체 메일박스 폴링",
    "description": "모든 활성 메일박스 새 메일 확인 (5분)",
    "task_name": "app.workers.tasks.check_new_emails",
    "schedule_type": "crontab",
    "cron_expression": "*/5 * * * *",
    "queue": "email_polling",
    "is_active": true
  }
]
```

---

## 3. 의존성 그래프 + 병렬 개발 구조

```
Task 1-1 (모델+스키마) ──┬──→ Task 1-4 (DatabaseScheduler)
                         ├──→ Task 1-6 (ScheduleService)
                         └──→ Task 1-7 (Admin API) ──→ Task 1-8 (celery_app 수정)

Task 1-2 (sync_session) ──→ Task 1-4 (DatabaseScheduler)

Task 1-3 (GmailService) ──→ Task 1-5 (메일박스별 폴링 태스크)

Task 1-9 (Admin UI) ← Task 1-7 완료 후 연동 테스트
Task 1-10 (Docker+dev.sh) ← 독립

병렬 가능 그룹:
  A: Task 1-1, 1-2, 1-3, 1-10 (모두 독립)
  B: Task 1-4, 1-5, 1-6 (A 완료 후)
  C: Task 1-7, 1-8 (B 완료 후)
  D: Task 1-9 (C 완료 후, 또는 Mock API로 병렬)
```

---

## 4. 테스트 체크리스트

- [ ] celery_schedules 테이블 생성 확인
- [ ] mailbox_configs 3개 컬럼 추가 확인
- [ ] CelerySchedule 모델 ORM 동작 (CRUD)
- [ ] sync_session DB 연결 성공
- [ ] GmailService("support@keychron.kr") — delegation 인증 성공 (또는 폴백)
- [ ] GmailService().list_unread_messages(query="to:support@keychron.kr") 동작
- [ ] DatabaseScheduler — Beat 시작 시 DB에서 스케줄 로드
- [ ] DatabaseScheduler — DB 변경 30초 이내 감지
- [ ] check_mailbox_emails 태스크 — 새 메일 감지 + process_email_task 큐잉
- [ ] Admin API 7개 엔드포인트 동작
- [ ] Admin API cron_expression 유효성 검증 (5필드)
- [ ] Admin API 즉시실행 → Celery 태스크 큐잉 확인
- [ ] Admin API 메일박스 동기화 → 스케줄 자동 생성
- [ ] POST /admin/mailboxes → 폴링 스케줄 자동 생성
- [ ] Admin UI 스케줄 목록/생성/수정/삭제/토글/즉시실행
- [ ] dev.sh — Worker + Beat + API 동시 실행
- [ ] E2E: 메일 수신 → AI 처리 → 자동발송 (워커 실행 상태에서)
