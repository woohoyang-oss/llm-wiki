---
title: eywa MCP 개발자 가이드
tags: [eywa, MCP, 개발, wiki, 통합서버]
created: 2026-04-07
updated: 2026-04-07
sources: [eywa-mcp-codebase]
---

# eywa MCP 개발자 가이드

eywa-mcp는 조직의 **유일한 MCP 서버**다. 위키(지식), 데이터(tobe-mcp), 스킬 가이드를 하나의 서버로 통합하여 Claude가 단일 연결로 모든 도구에 접근할 수 있게 한다.

---

## 1. 역할

```
┌──────────────────────────────────────────────┐
│              eywa-mcp (통합 MCP 서버)          │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Wiki     │ │ Guides   │ │ Data         │  │
│  │ 4 + 3    │ │ 2 도구    │ │ 51 도구      │  │
│  │ 도구      │ │          │ │ (tobe-mcp)   │  │
│  └──────────┘ └──────────┘ └──────────────┘  │
│         │           │             │           │
│    md 파일들     YAML 파일     aiomysql DB    │
└──────────────────────────────────────────────┘
         ↑
    Claude (stdio 또는 Streamable HTTP)
```

- **위키 도구 (7개)**: 검색, 읽기, 목록, 그래프 + 쓰기/수정/삭제
- **가이드 도구 (2개)**: 스킬 가이드 목록, 스킬 가이드 조회
- **데이터 도구 (51개)**: tobe-mcp-v1에서 임포트한 매출/재고/원가/CS/광고/GFA 도구

---

## 2. 프로젝트 구조

```
eywa-mcp/
├── eywa_mcp/
│   ├── server.py        # MCP 서버 (stdio + Streamable HTTP)
│   ├── wiki.py          # WikiIndex — md 파싱, 검색, [[wikilink]] 그래프
│   ├── guides.py        # 스킬 가이드 로더 (YAML)
│   └── config.py        # Settings (pydantic-settings)
├── guides/              # YAML 스킬 가이드 파일 (7개)
│   ├── morning-brief.yaml
│   ├── cs-urgency.yaml
│   └── ...
├── certs/               # mkcert HTTPS 인증서 (Streamable HTTP용)
│   ├── cert.pem
│   └── key.pem
├── .env                 # 환경변수 (서버마다 다름!)
├── pyproject.toml
└── README.md
```

### 핵심 파일 역할

| 파일 | 역할 |
|------|------|
| `server.py` | MCP 프로토콜 처리, 도구 등록, stdio/HTTP 분기 |
| `wiki.py` | WikiIndex 클래스 — 위키 인덱싱, 검색, 그래프 |
| `guides.py` | guides/ 디렉토리의 YAML 파일 로드 및 조회 |
| `config.py` | `.env` → Settings 객체 (WIKI_PATH, TOBE_MCP_PATH, SSL 등) |

---

## 3. 도구 구성

### EYWA_TOOLS (9개) — 위키 + 가이드

```python
EYWA_TOOLS = [
    # 위키 읽기 (4개)
    ("wiki_search",   "위키 페이지 검색",           {...}, [...], wiki_search),
    ("wiki_read",     "위키 페이지 읽기",           {...}, [...], wiki_read),
    ("wiki_list",     "위키 페이지 목록",           {...}, [],    wiki_list),
    ("wiki_graph",    "위키 링크 그래프 조회",       {...}, [...], wiki_graph),

    # 위키 쓰기 (3개, admin 전용)
    ("wiki_edit",     "위키 페이지 생성/수정",       {...}, [...], wiki_edit),
    ("wiki_delete",   "위키 페이지 삭제",           {...}, [...], wiki_delete),
    ("wiki_write",    "위키 새 페이지 작성",         {...}, [...], wiki_write),

    # 가이드 (2개)
    ("guide_list",    "스킬 가이드 목록",           {...}, [],    guide_list),
    ("guide_get",     "스킬 가이드 조회",           {...}, [...], guide_get),
]
```

### DATA_TOOLS (51개) — tobe-mcp-v1 임포트

```python
import sys
sys.path.insert(0, settings.TOBE_MCP_PATH)
from mcp_server import TOOLS as DATA_TOOLS
```

### 전체 도구

```python
ALL_TOOLS = EYWA_TOOLS + DATA_TOOLS  # 9 + 51 = 60개
```

---

## 4. tobe-mcp 통합 방식

eywa-mcp는 tobe-mcp-v1을 **별도 프로세스가 아닌 같은 프로세스에서 직접 임포트**한다.

```python
# eywa_mcp/server.py
import sys
from .config import settings

# tobe-mcp-v1 경로를 sys.path에 추가
sys.path.insert(0, settings.TOBE_MCP_PATH)

# TOOLS 리스트를 직접 임포트
from mcp_server import TOOLS as DATA_TOOLS
```

**동작 방식:**
1. eywa-mcp 시작 시 `TOBE_MCP_PATH` (예: `/home/deploy/tobe-mcp-v1`)를 sys.path에 삽입
2. tobe-mcp-v1의 `mcp_server.py`에서 `TOOLS` 리스트를 임포트
3. EYWA_TOOLS + DATA_TOOLS를 합쳐 ALL_TOOLS로 MCP에 등록
4. tobe-mcp-v1의 핸들러가 호출되면 같은 프로세스의 aiomysql 풀을 사용

**장점:**
- 네트워크 오버헤드 없음 (프로세스 내 함수 호출)
- 배포 단순화 (eywa-mcp 하나만 실행)
- 도구 추가 시 eywa 재시작만 하면 반영

---

## 5. 위키 인덱서 (WikiIndex)

`wiki.py`의 `WikiIndex` 클래스가 위키 파일 인덱싱과 검색을 담당한다.

### 핵심 메서드

```python
class WikiIndex:
    def __init__(self, wiki_path: str):
        self.wiki_path = Path(wiki_path)
        self.pages: dict[str, PageMeta] = {}
        self.graph: dict[str, set[str]] = {}  # outgoing links
        self.backlinks: dict[str, set[str]] = {}  # incoming links

    async def build(self):
        """전체 위키 인덱스 구축 (시작 시 1회)"""

    async def search(self, query: str, tags: list[str] = None) -> list[dict]:
        """키워드 + 태그 기반 검색"""

    async def read(self, page_name: str) -> str:
        """페이지 전체 내용 반환"""

    async def graph(self, page_name: str) -> dict:
        """페이지의 outgoing + incoming 링크"""

    async def list_pages(self, directory: str = None) -> list[dict]:
        """페이지 목록 (디렉토리 필터 가능)"""
```

### Frontmatter 파싱

```python
# gray-matter 스타일 Python 파싱
def _parse_frontmatter(content: str) -> tuple[dict, str]:
    """
    ---
    title: 페이지 제목
    tags: [tag1, tag2]
    ---
    본문 내용...
    """
    # YAML frontmatter를 dict로, 본문을 str로 분리
```

### [[wikilink]] 그래프

위키 본문에서 `[[페이지명]]` 패턴을 추출하여 방향 그래프를 구축한다.

```python
# outgoing: 이 페이지가 참조하는 페이지들
graph["eywa"] = {"tobe-mcp", "auto-email", "wiki-mcp"}

# incoming (backlinks): 이 페이지를 참조하는 페이지들
backlinks["tobe-mcp"] = {"eywa", "tobe-mcp-improvement", "skills-overview"}
```

### Watchdog 파일 감시 (SSE 모드)

Streamable HTTP 모드에서는 watchdog으로 위키 파일 변경을 감지한다.

```python
# 파일 변경 감지 → 인덱스 자동 갱신
# 0.5초 debounce로 연속 변경 시 과도한 리빌드 방지
class WikiWatcher(FileSystemEventHandler):
    def on_modified(self, event):
        if event.src_path.endswith(".md"):
            self._schedule_rebuild()  # 0.5s debounce
```

---

## 6. 스킬 가이드

### YAML 형식

```yaml
# guides/morning-brief.yaml
name: morning-brief
title: 모닝 브리프
description: 매일 아침 주요 지표 요약 가이드
steps:
  - action: get_daily_sales
    params:
      start_date: "{today}"
      end_date: "{today}"
    description: 오늘 매출 조회
  - action: get_stock_alerts
    description: 재고 알림 확인
  # ...
```

### 7개 가이드

| 가이드 | 설명 |
|--------|------|
| `morning-brief` | 아침 주요 지표 요약 |
| `cs-urgency` | CS 긴급 대응 프로세스 |
| `inventory-check` | 재고 점검 절차 |
| `ad-performance` | 광고 성과 분석 |
| `weekly-report` | 주간 리포트 생성 |
| `gfa-optimization` | GFA 최적화 가이드 |
| `cost-analysis` | 원가 분석 절차 |

### 도구 인터페이스

```python
# guide_list: 전체 가이드 목록 반환
# guide_get: 특정 가이드의 전체 YAML 내용 반환
```

---

## 7. 권한 모델

위키 쓰기/삭제 도구는 관리자 전용이다.

```python
# config.py
class Settings(BaseSettings):
    wiki_admins: list[str] = []  # .env에서 로드
    # WIKI_ADMINS=admin1@tobe.kr,admin2@tobe.kr

# server.py — wiki_edit 핸들러
async def wiki_edit(args: dict) -> str:
    email = args.get("admin_email", "")
    if email not in settings.wiki_admins:
        return _json({"error": "권한 없음. 관리자만 수정 가능합니다."})
    # ... 수정 로직
```

| 도구 | 권한 | 검증 |
|------|------|------|
| wiki_search, wiki_read, wiki_list, wiki_graph | 모든 사용자 | 없음 |
| wiki_edit, wiki_delete, wiki_write | 관리자 | admin_email 검증 |
| guide_list, guide_get | 모든 사용자 | 없음 |
| DATA_TOOLS (51개) | 모든 사용자 | 없음 |

---

## 8. Streamable HTTP

사내 네트워크에서 여러 클라이언트가 접속할 수 있도록 Streamable HTTP 모드를 지원한다.

```python
# server.py
from mcp.server.streamable_http import StreamableHTTPSessionManager

if args.mode == "http":
    manager = StreamableHTTPSessionManager(server)

    # /mcp — MCP 프로토콜 엔드포인트
    app.route("/mcp", methods=["GET", "POST"])(manager.handle)

    # /health — 헬스체크
    @app.route("/health")
    def health():
        return {"status": "ok", "tools": len(ALL_TOOLS)}

    # HTTPS (mkcert 인증서)
    app.run(
        host="0.0.0.0",
        port=settings.port,
        ssl_context=(settings.ssl_cert, settings.ssl_key)
    )
```

### 엔드포인트

| 경로 | 메서드 | 용도 |
|------|--------|------|
| `/mcp` | GET, POST | MCP Streamable HTTP 프로토콜 |
| `/health` | GET | 헬스체크 (도구 수 포함) |

### 접속 설정 (Claude Desktop)

```json
{
  "mcpServers": {
    "eywa": {
      "transport": "streamable-http",
      "url": "https://192.168.x.x:8443/mcp"
    }
  }
}
```

---

## 9. 배포

### rsync 배포

```bash
# 소스 서버 → 운영 서버
rsync -avz --delete \
  --exclude='.env' \
  --exclude='certs/' \
  --exclude='__pycache__/' \
  --exclude='.venv/' \
  ./eywa-mcp/ deploy@server:/home/deploy/eywa-mcp/
```

**중요:** `.env`는 서버마다 다르므로 반드시 제외한다.

### 서버별 .env 차이

| 변수 | 로컬 (Mac) | 운영 서버 |
|------|-----------|----------|
| `WIKI_PATH` | `/Users/.../llm-wiki` | `/home/deploy/llm-wiki` |
| `TOBE_MCP_PATH` | `/Users/.../tobe-mcp-v1` | `/home/deploy/tobe-mcp-v1` |
| `SSL_CERT` | `certs/cert.pem` | `/etc/ssl/eywa/cert.pem` |
| `DB_HOST` | `localhost` | `db.internal` |

### 실행

```bash
# stdio 모드 (로컬 Claude Code)
cd /home/deploy/eywa-mcp
.venv/bin/python -m eywa_mcp.server

# Streamable HTTP 모드 (사내 네트워크)
cd /home/deploy/eywa-mcp
.venv/bin/python -m eywa_mcp.server --http
```

---

## 10. 새 위키 도구 추가

### 순서

1. `server.py`의 `EYWA_TOOLS`에 튜플 추가
2. 핸들러 함수를 `wiki.py` 또는 별도 모듈에 구현
3. 테스트 후 배포

### 예시: 위키 통계 도구 추가

```python
# eywa_mcp/server.py
EYWA_TOOLS.append((
    "wiki_stats",
    "위키 전체 통계 (페이지 수, 태그 분포, 링크 수)",
    {},
    [],
    wiki_stats_handler
))

# eywa_mcp/wiki.py
async def wiki_stats_handler(args: dict) -> str:
    index = get_wiki_index()
    return _json({
        "total_pages": len(index.pages),
        "total_links": sum(len(v) for v in index.graph.values()),
        "tags": index.get_tag_distribution()
    })
```

---

## 11. 관련 문서

- [[eywa]] — Eywa 프로젝트 개요
- [[tobe-mcp]] — ToBe MCP 프로젝트 개요 및 도구 목록
- [[eywa-mcp-guide]] — eywa MCP 사용 가이드 (최종 사용자용)
- [[tobe-mcp-dev-guide]] — ToBe MCP 개발자 가이드
- [[telegram-integration]] — 텔레그램 연동 패턴
