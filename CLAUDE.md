# template

Claude Code plugin template repository.

## Commands

Validate plugin:

```bash
claude plugin validate
```

Validate YAML:

```bash
python3 -c "import glob, yaml; [list(yaml.safe_load_all(open(f))) for f in glob.glob('**/*.y*ml', recursive=True, include_hidden=True) if '/.git/' not in f and '/node_modules/' not in f]"
```

## Structure

```text
.claude-plugin/plugin.json   # Plugin manifest (name, version, description, author)
skills/<name>/SKILL.md       # Skill definition — one folder per skill
hooks/                       # (optional) Hook definitions
commands/                    # (optional) Slash command definitions
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md`
2. Add frontmatter — `description:` is required
   (Claude uses it to decide when to invoke)
3. Invoke via `/template:<skill-name>`

## Secrets required

| Secret                    | Used by                                       |
| ------------------------- | --------------------------------------------- |
| `CLAUDE_CODE_OAUTH_TOKEN` | `claude.yml`, `claude-code-review.yml`        |
| `MY_RELEASE_PLEASE_TOKEN` | `CI.yml` `tag` job (PAT — contents:write)     |

## CI checks (`CI.yml`)

- **JSON validity**: all `.json` files must parse
- **YAML validity**: all YAML files must parse
- **Manifest**: `.claude-plugin/plugin.json` must exist
