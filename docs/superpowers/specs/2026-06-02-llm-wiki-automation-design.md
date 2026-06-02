# LLM Wiki Automation System — Design Spec

**Date:** 2026-06-02
**Status:** Approved

---

## Overview

Sources 폴더에 파일이 추가/수정되면 GitHub Actions가 자동으로 Deepseek API를 호출해 Schema 규칙에 따라 Wiki를 생성/업데이트하는 시스템.

Karpathy의 LLM Wiki 패턴을 기반으로 설계.

---

## Folder Structure

```
0.Sources/              ← 원본 파일 (MD, PDF, DOC 등), 불변
1.Wiki/
  index.md              ← 전체 Wiki 카탈로그 (링크 + 한 줄 요약)
  log.md                ← 작업 이력 (append-only)
  [Category]/
    index.md            ← 카테고리 설명 + 하위 페이지 링크 목록
    [Subcategory]/
      index.md
      [Concept].md      ← 개념 페이지
  sources/
    index.md
    [SourceSummary].md  ← 소스별 요약 페이지
2.Schema/
  AGENTS.md             ← LLM 동작 규칙 (Ingest/Lint 워크플로우)
  wiki-format.md        ← Wiki 페이지 형식, frontmatter, 링크 규칙
```

### 폴더 규칙
- 계층 구조 최대 3depth (`1.Wiki/Category/Subcategory/Page.md`)
- 새 폴더 생성 시 반드시 `index.md` 함께 생성
- `index.md`는 해당 폴더 설명 + 하위 파일/폴더 위키링크 목록 포함
- 카테고리: LLM이 자율 판단, 기존 카테고리로 커버 안 될 때만 신규 생성

---

## GitHub Actions Pipeline

### Trigger
```yaml
on:
  push:
    branches: [main]
    paths: ['0.Sources/**']
```

### Steps

1. **변경 파일 감지** — `git diff HEAD~1 --name-only -- 0.Sources/`
2. **텍스트 추출**
   - `.md` → 그대로 사용
   - `.pdf` → `pdftotext` (poppler-utils)
   - `.doc/.docx` → `python-docx`
3. **1단계 Deepseek 호출** (분석/계획)
   - 입력: Schema(AGENTS.md + wiki-format.md) + 소스 텍스트 + 현재 Wiki index.md
   - 출력: 분류, 태그, 연관 Wiki 페이지 목록, 작업 계획 (JSON)
4. **2단계 Deepseek 호출** (실행)
   - 입력: 1단계 결과 + 관련 Wiki 페이지 현재 내용
   - 출력: 생성/수정할 파일 목록 (경로 + 내용)
5. **파일 write** — Actions에서 직접 파일 생성/수정
6. **index.md, log.md 갱신**
7. **자동 commit & push** → Quartz 빌드 트리거

### 모델
- **deepseek-chat** (DeepSeek V3 계열)
- 변경 필요 시 환경변수 `DEEPSEEK_MODEL`로 오버라이드

---

## Wiki Page Format (Quartz 호환)

### Frontmatter
```yaml
---
title: 페이지 제목
date: YYYY-MM-DD
tags:
  - tag1
  - tag2
publish: true
---
```

### 본문 규칙
- 내부 링크: `[[페이지명]]` (Obsidian 위키링크, shortest path)
- 백링크는 Quartz가 자동 처리 — 명시적 작성 불필요
- 개념 페이지: 정의 → 핵심 내용 → 관련 개념 링크 순서
- 소스 요약 페이지: 출처 → 요약 → 핵심 인사이트 → 관련 Wiki 페이지 링크

### index.md 형식
```markdown
---
title: [폴더명]
publish: true
---

[폴더 한 줄 설명]

## 페이지
- [[Page1]]
- [[Page2]]

## 하위 카테고리
- [[Subcategory/index|Subcategory]]
```

---

## Schema Files

### AGENTS.md — LLM 동작 규칙
- **Ingest 워크플로우**: 소스 분석 → 분류 → Wiki 업데이트 절차
- **Lint 규칙**: 고아 페이지, 모순, 오래된 내용 감지 기준
- **카테고리 생성 기준**: 기존 카테고리로 커버 불가할 때만
- **업데이트 원칙**: 기존 페이지 병합 우선, 모순 시 최신 소스 기준

### wiki-format.md — 형식 규칙
- Quartz frontmatter 스펙
- 폴더/파일 네이밍 규칙 (영문, kebab-case)
- 링크 방식, 태그 규칙
- index.md 작성 형식

---

## Processed File Tracking

`git diff HEAD~1 --name-only -- 0.Sources/` 로 변경된 파일만 처리.
- 새 파일: ingest
- 수정된 파일: 관련 Wiki 페이지 재검토 후 업데이트
- 삭제된 파일: 관련 Wiki 페이지에 소스 제거 표시 (내용은 유지)

---

## Secrets Required

| Secret | 용도 |
|--------|------|
| `DEEPSEEK_API_KEY` | Deepseek API 인증 |
| `QUARTZ_DISPATCH_TOKEN` | Quartz 빌드 트리거 (기존) |

---

## Out of Scope

- URL 수집/처리 (Obsidian에서 MD로 변환 후 저장)
- 이미지 파일 처리
- Wiki → Sources 역방향 추적
