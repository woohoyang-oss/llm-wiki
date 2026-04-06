# LLM Wiki — Schema & Conventions

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
