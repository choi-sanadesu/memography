# CLAUDE.md — LLM Wiki

Automated personal knowledge base. The agent compiles and maintains this wiki. The user curates sources and asks questions.

## Layers

- **raw/** — immutable source documents. You read from these; you never modify them.
- **wiki/** — your workspace. You create, update, and maintain everything here.
- **output/** — generated artifacts (slides, charts, reports). Write here only when the user explicitly asks for a deliverable; never auto-generate.

Wiki subdirectories:

- `wiki/sources/` — one summary page per ingested source.
- `wiki/entities/` — pages for people, organizations, products, tools.
- `wiki/concepts/` — pages for ideas, frameworks, theories, patterns.
- `wiki/synthesis/` — comparisons, analyses, cross-cutting themes. **LYT MOC pattern (linking hub)** — synthesis 페이지는 주제별 표·리스트로 관련 페이지를 묶는 navigation hub다.

Special files:

- `index.md` — catalog of every wiki page, organized by category. Update on every ingest.
- `log.md` — append-only chronological record. Never edit existing entries.

## Scaffold (자가 초기화)

이 vault에 다음 중 하나라도 없으면, 첫 ingest/query/lint 작업을 시작하기 전에 자동으로 생성한다.

필수 디렉토리:

- `raw/`
- `wiki/sources/`
- `wiki/entities/`
- `wiki/concepts/`
- `wiki/synthesis/`
- `output/`

필수 파일:

- `index.md` — 4개 카테고리 헤더(`## Sources`, `## Entities`, `## Concepts`, `## Synthesis`)만 가진 빈 카탈로그.
- `log.md` — `# Log` 헤더만.

자동 생성 후 사용자에게 "스캐폴드 완료. 무엇을 ingest할까요?"로 안내한다.

## Operations

### Ingest

When I drop a source into `raw/` and ask you to ingest it:

1. Read the source completely.
2. Discuss key takeaways with me.
3. Write a summary page in `wiki/sources/`.
4. For each entity and concept mentioned: create or update the page in `wiki/entities/` or `wiki/concepts/`.
5. Add `[[wikilinks]]` between related pages.
6. Update `index.md` with any new pages.
7. Append to `log.md`: `## [YYYY-MM-DD] ingest | <Title>`.

A single source may touch 10–15 wiki pages. That is normal.

### Query

When I ask a question:

1. Read `index.md` to find relevant pages.
2. Read those pages.
3. Answer with `[[wikilink]]` citations.
4. If the answer is valuable (a comparison, an analysis, a new connection), offer to save it to `wiki/synthesis/`. synthesis 페이지는 LYT MOC 패턴(헤더 + `[[wikilink]]` 리스트)으로 관련 페이지를 묶는다.
5. If saved, update `index.md` and append to `log.md`: `## [YYYY-MM-DD] query | <Title>`.

### Lint

When I ask you to lint or health-check:

1. Find contradictions between pages.
2. Find stale claims newer sources have superseded.
3. Find orphan pages (no inbound links).
4. Find concepts mentioned but lacking their own page.
5. Find missing cross-references.
6. Suggest gaps that could be filled with external lookup. For library/SDK/API docs, prefer Context7 MCP over web search.
7. Report findings; offer to fix.
8. Append to `log.md`: `## [YYYY-MM-DD] lint | <one-line summary>`.

## Page format

Every wiki page has YAML frontmatter:

```yaml
---
tags: [tag1, tag2]
sources: [source-filename-1.md, source-filename-2.md]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

Use `[[Page Title]]` for all internal links. Page titles are Title Case; filenames are kebab-case (e.g., `wiki/entities/andrej-karpathy.md` with H1 `# Andrej Karpathy`).

Wiki pages are written in the user's preferred language. Default to the language the user is conversing in; if `wiki/` already contains pages, match the existing language for consistency unless the user asks to switch. Quote source-language passages verbatim when needed.

## Index format

One line per page under category headers (`## Sources`, `## Entities`, `## Concepts`, `## Synthesis`):

```
- [[Page Title]] — one-line summary (≤120 chars)
```

## Log format

Each entry begins with `## [YYYY-MM-DD] <op> | <Title>` followed by a one-line description. The consistent prefix makes the log grep-parseable:

```
grep "^## \[" log.md | tail -5
```

## Hard rules

1. Never modify files in `raw/`. They are immutable.
2. Always update `index.md` when you create or delete a wiki page.
3. Always append to `log.md` when you perform an operation.
4. Never edit existing entries in `log.md`. Only append.
5. Use `[[wikilinks]]` for all internal references. Never raw file paths.
6. Every wiki page has frontmatter (tags, sources, created, updated).
7. When a new source contradicts an existing claim, update the page AND note both sources. Don't silently overwrite.
8. Source pages stay factual. Interpretation goes in concept and synthesis pages.
9. When I ask a question, search the wiki first. Only fall back to `raw/` if the wiki doesn't have the answer.
10. Prefer updating existing pages over creating new ones.
11. 페이지는 evolve·accumulate한다. 새 정보가 들어오면 누적·진화시킨다. 과거 사실은 source 명기 후 보존한다 (=Evergreen 원칙).
12. 폴더는 2-level depth 초과 금지 (예: `wiki/entities/` OK, `wiki/entities/people/researchers/` 금지).
