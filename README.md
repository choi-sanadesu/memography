# bookkeeper

> Personal knowledge wiki builder for Claude Code

bookkeeper is a Claude Code plugin that turns any folder into a wikilinked knowledge vault. Drop sources into `raw/`, ask Claude to ingest them, and the plugin builds and maintains a network of pages on entities, concepts, and syntheses — with `[[wikilinks]]`, an index, and a chronological log.

## Install

```sh
# from the Claude Code plugin marketplace
claude plugin install bookkeeper

# or from a local clone
claude plugin install /path/to/bookkeeper
```

## Use

In any folder you want as a vault, open Claude Code and run one of:

| Command | What it does |
| --- | --- |
| `/bookkeeper:ingest <filename>` | Read a source from `raw/` and write/update wiki pages. |
| `/bookkeeper:query <question>` | Answer from the wiki with `[[wikilink]]` citations. |
| `/bookkeeper:lint` | Health-check the wiki (contradictions, stale claims, orphans). |

You can also just talk to Claude — "정리해줘", "이 PDF 읽고 위키에 정리해", "내 노트에서 X 찾아봐" — and the bookkeeper skill will trigger automatically.

The first time you invoke a command in an empty folder, bookkeeper sets up the vault layout for you. If the folder looks like a code project (has `.git/` plus a `package.json`/`pyproject.toml`/etc.), it stops and asks for confirmation first.

## Vault layout

```text
<your-vault>/
├── raw/               # source documents you drop in (immutable)
├── wiki/
│   ├── sources/       # one summary page per ingested source
│   ├── entities/      # people, organizations, products, tools
│   ├── concepts/      # ideas, frameworks, theories, patterns
│   └── synthesis/     # comparisons and cross-cutting analyses
├── output/            # generated artifacts (slides, charts) — explicit only
├── index.md           # catalog of every wiki page
└── log.md             # chronological append-only record
```

Every wiki page is plain Markdown with YAML frontmatter (`tags`, `sources`, `created`, `updated`) and uses `[[Page Title]]` for internal links. The vault is fully portable — back it up, sync it, version it with git, open it in Obsidian.

## How it works

The plugin ships:

- A skill (`skills/bookkeeper/SKILL.md`) that defines the full ingest/query/lint behavior and triggers on natural-language requests.
- Three slash commands (`commands/ingest.md`, `commands/query.md`, `commands/lint.md`) as explicit entry points.
- A manifest (`.claude-plugin/plugin.json`).

The skill operates on the **current working directory** (CWD). It never reaches outside the folder you opened.

## License

MIT — see [LICENSE](LICENSE).
