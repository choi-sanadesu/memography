---
name: bookkeeper
description: Use when the user asks to ingest a source, query a knowledge base, lint the wiki, or otherwise build or maintain a personal wikilinked knowledge vault. Treats the current working directory (CWD) as the vault.
---

# bookkeeper

Automated personal knowledge base. The agent compiles and maintains a wikilinked vault. The user curates sources and asks questions.

## Vault location

The **current working directory (CWD)** is the vault. All paths in this skill (`raw/`, `wiki/`, `output/`, `index.md`, `log.md`) are relative to CWD. Never operate on directories outside CWD.

## Layers

- **raw/** — immutable source documents. Read from these; never modify them.
- **wiki/** — your workspace. Create, update, and maintain everything here.
- **output/** — generated artifacts (slides, charts, reports). Write here only when the user explicitly asks for a deliverable; never auto-generate.

Wiki subdirectories:

- `wiki/sources/` — one summary page per ingested source.
- `wiki/entities/` — pages for people, organizations, products, tools.
- `wiki/concepts/` — pages for ideas, frameworks, theories, patterns.
- `wiki/synthesis/` — comparisons, analyses, cross-cutting themes. **LYT MOC pattern (linking hub)** — synthesis pages are navigation hubs that group related pages with topic-based tables and lists.

Special files:

- `index.md` — catalog of every wiki page, organized by category. Update on every ingest.
- `log.md` — append-only chronological record. Never edit existing entries.

## Scaffold (initialization)

Trigger scaffolding **only when the user explicitly invokes a bookkeeper operation** (a `/bookkeeper:*` slash command, or a natural-language request that triggers this skill — "ingest this PDF", "look up X in my notes", "lint the wiki"). Never scaffold just because the CWD is empty.

Before scaffolding, run this safety check:

1. If CWD already contains the required vault layout below, skip to the operation.
2. If CWD looks like a code project (`.git/` exists **and** any of `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, `pom.xml`, `build.gradle` is present), **stop and ask the user**: "This folder looks like a code project. Are you sure you want to use it as a bookkeeper vault?" Do not create any vault files until the user confirms.
3. Otherwise proceed.

Required directories (create if missing):

- `raw/`
- `wiki/sources/`
- `wiki/entities/`
- `wiki/concepts/`
- `wiki/synthesis/`
- `output/`

Required files (create if missing):

- `index.md` — empty catalog containing only the four category headers (`## Sources`, `## Entities`, `## Concepts`, `## Synthesis`).
- `log.md` — only the `# Log` header.

After scaffolding, prompt the user: "Scaffold complete. What would you like to ingest?"

## Operations

### Ingest

When the user drops a source into `raw/` and asks you to ingest it:

1. Read the source completely.
2. Discuss key takeaways with the user.
3. Write a summary page in `wiki/sources/`.
4. For each entity and concept mentioned: create or update the page in `wiki/entities/` or `wiki/concepts/`.
5. Add `[[wikilinks]]` between related pages.
6. Update `index.md` with any new pages.
7. Append to `log.md`: `## [YYYY-MM-DD] ingest | <Title>`.

A single source may touch 10–15 wiki pages. That is normal.

### Query

When the user asks a question:

1. Read `index.md` to find relevant pages.
2. Read those pages.
3. Answer with `[[wikilink]]` citations.
4. If the answer is valuable (a comparison, an analysis, a new connection), offer to save it to `wiki/synthesis/`. Synthesis pages group related pages using the LYT MOC pattern (headers + `[[wikilink]]` lists).
5. If saved, update `index.md` and append to `log.md`: `## [YYYY-MM-DD] query | <Title>`.

### Lint

When the user asks for a lint or health check:

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

```text
- [[Page Title]] — one-line summary (≤120 chars)
```

## Log format

Each entry begins with `## [YYYY-MM-DD] <op> | <Title>` followed by a one-line description. The consistent prefix makes the log grep-parseable:

```bash
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
9. When the user asks a question, search the wiki first. Only fall back to `raw/` if the wiki doesn't have the answer.
10. Prefer updating existing pages over creating new ones.
11. Pages evolve and accumulate. When new information arrives, integrate and evolve the page; preserve past facts with their source attributed (Evergreen principle).
12. No folder nesting beyond 2 levels (e.g., `wiki/entities/` OK; `wiki/entities/people/researchers/` not allowed).
