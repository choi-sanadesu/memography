# CLAUDE.md — bookkeeper plugin

This file is for people maintaining the **plugin itself**, not for users running it. End-user docs live in [README.md](README.md). The full runtime behavior of the plugin is defined in [skills/bookkeeper/SKILL.md](skills/bookkeeper/SKILL.md) — do not duplicate those rules here.

## Layout

```text
.
├── .claude-plugin/
│   └── plugin.json              # manifest (name, version, description)
├── skills/
│   └── bookkeeper/
│       └── SKILL.md             # full ingest / query / lint behavior
└── commands/
    ├── ingest.md                # /bookkeeper:ingest
    ├── query.md                 # /bookkeeper:query
    └── lint.md                  # /bookkeeper:lint
```

The slash commands are thin wrappers that delegate to the skill. Add new behavior in `SKILL.md` first; only add a new slash command when there is a distinct user-facing entry point.

## Validate

```sh
claude plugin validate
```

## Editing rules

- Behavior changes (what bookkeeper does to a vault) → edit `skills/bookkeeper/SKILL.md`.
- New entry point → add a file under `commands/` and have its body refer to the skill.
- Manifest changes (version bump, description) → `.claude-plugin/plugin.json` and `CHANGELOG.md`.
- Never put runtime vault data (`raw/`, `wiki/`, `output/`, `index.md`, `log.md`) into this repo. Those belong in the user's vault folder.

## Resources

- [Claude Code plugin docs](https://code.claude.com/docs/en/plugins)
- [Plugin reference](https://code.claude.com/docs/en/plugins-reference)
