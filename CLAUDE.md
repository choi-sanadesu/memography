# Alexandria Keeper

Alexandria(구 memorial-vault) Obsidian 지식 볼트의 무결성과 구조를 유지하는 Claude Code 플러그인.

제공 스킬:

- `/alexandria-keeper:lint` — 볼트 헬스체크
- `/alexandria-keeper:ingest` — 새 소스 추가
- `/alexandria-keeper:promote` — 엔티티 허브 페이지 생성
- `/alexandria-keeper:retro` — 회고 기록

## Commands

플러그인 검증:

```bash
claude plugin validate
```

YAML 검증:

```bash
python3 -c "import glob, yaml; [list(yaml.safe_load_all(open(f))) for f in glob.glob('**/*.y*ml', recursive=True, include_hidden=True) if '/.git/' not in f and '/node_modules/' not in f]"
```

## Structure

```text
.claude-plugin/plugin.json       # 플러그인 매니페스트 (name, version, description, author)
skills/<name>/SKILL.md           # 스킬 정의 — 스킬 하나당 폴더 하나
skills/<name>/references/        # (선택) 스킬 참조 문서
```

## Adding a skill

1. `skills/<skill-name>/SKILL.md` 생성
2. frontmatter에 `description:` 필수 (Claude가 스킬 호출 시점을 이 필드로 판단)
3. `/alexandria-keeper:<skill-name>`로 호출
4. 릴리스 전 `CHANGELOG.md`에 항목 추가 (버전은 `workflow_dispatch` 태깅 시 CI가 자동 갱신)

## Secrets required

| Secret                    | Used by                                  |
| ------------------------- | ---------------------------------------- |
| `CLAUDE_CODE_OAUTH_TOKEN` | `claude.yml`, `claude-code-review.yml`   |
| `MY_RELEASE_PLEASE_TOKEN` | `CI.yml` `tag` 잡 (PAT — contents:write) |

## CI checks (`CI.yml`)

- **JSON validity**: 모든 `.json` 파일이 파싱 가능해야 함 (hidden 포함)
- **YAML validity**: 모든 YAML 파일이 pyyaml로 파싱 가능해야 함
- **Manifest**: `.claude-plugin/plugin.json` 존재 확인
