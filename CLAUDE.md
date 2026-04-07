# LLM Wiki — Schema & Conventions

## 세션 시작 시 자동 실행
이 프로젝트 디렉토리에서 Claude 세션이 시작되면 반드시 `index.md`를 읽고 위키 전체 구조를 파악한 상태에서 대화를 시작하세요.

## Purpose
키크론/지티기어 업무 관련 지식을 축적하는 비즈니스 위키.
회의록, 프로젝트 문서, 아티클, 경쟁사 분석 등을 정리한다.

## Directory Structure

```
raw/            # 원본 소스 (불변, LLM은 읽기만)
  assets/       # 이미지, PDF 등 첨부파일
wiki/           # LLM이 생성·관리하는 마크다운 파일
  summaries/    # 소스별 요약 페이지
  entities/     # 엔티티 페이지 (사람, 회사, 제품 등)
  concepts/     # 개념/주제 페이지
index.md        # 위키 전체 카탈로그
log.md          # 작업 이력 (append-only)
```

## Page Conventions

### Frontmatter (YAML)
모든 위키 페이지 상단에 포함:
```yaml
---
title: 페이지 제목
tags: [태그1, 태그2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [관련 소스 파일명]
---
```

### Linking
- 위키 내부 링크: `[[페이지명]]` (Obsidian 호환)
- 소스 참조: `[출처](../raw/파일명.md)`

### Naming
- 파일명: kebab-case (예: `keychron-q1-pro.md`)
- 엔티티: 고유명사 그대로 (예: `keychron.md`, `naver-smartstore.md`)

## Workflows

### Ingest (소스 투입)
1. `raw/` 에 소스 파일 저장
2. LLM에게 "이 소스를 인제스트해줘" 요청
3. LLM이 수행하는 작업:
   - `wiki/summaries/`에 요약 페이지 생성
   - 관련 엔티티/개념 페이지 업데이트 또는 생성
   - `index.md` 갱신
   - `log.md`에 기록 추가

### Query (질의)
1. 질문하면 LLM이 `index.md` → 관련 페이지 읽기 → 답변
2. 유용한 답변은 새 위키 페이지로 저장

### Lint (점검)
- "위키 점검해줘" 요청 시 모순, 누락, 고아 페이지 등 체크

## Wiki MCP Server

사내 네트워크에서 전 직원이 Claude를 통해 위키에 접근할 수 있는 MCP 서버.

### 서버 위치
- 코드: `/Users/yangwooho/wiki-mcp/`
- Python: `.venv/bin/python3.13` (3.13+)
- 위키 경로: `.env`의 `WIKI_PATH`

### 실행
```bash
# stdio (로컬 관리자)
cd /Users/yangwooho/wiki-mcp && .venv/bin/python3.13 -m wiki_mcp.server

# SSE (사내 네트워크)
cd /Users/yangwooho/wiki-mcp && .venv/bin/python3.13 -m wiki_mcp.server --sse
```

### 도구 (10개)
- 일반 사용자: `wiki_search`, `wiki_read`, `wiki_list`, `wiki_graph`, `wiki_contribute`
- 관리자: `wiki_admin_edit`, `wiki_admin_delete`, `wiki_admin_approve`, `wiki_admin_reject`, `wiki_admin_ingest`

### 권한
- 일반 사용자: 검색, 읽기, 기여(제출만)
- 관리자: 전체 CRUD + 기여 승인/반려 + 인제스트
- 인증: Bearer 토큰 (`.env`에 `ADMIN_TOKENS`, `USER_TOKENS`)

### 조직별 문서
```
wiki/org/
├── cs/         # CS팀
├── sales/      # 영업팀
├── marketing/  # 마케팅팀
├── as/         # AS팀
└── md/         # MD(상품기획)팀
```
