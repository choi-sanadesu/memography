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

## Frontmatter 스키마 참조 (볼트 §3.1 mirror)

SOT는 대상 볼트의 `CLAUDE.md §3.1`. 플러그인 스킬은 해당 규칙을 **실행 시 동적으로 읽어서** 검증·생성한다 (enum 하드코딩 금지 — `skills/lint/SKILL.md` 동작 원칙). 아래는 운영 요약본.

### 필수 필드

| 필드        | 타입          | 비고                       |
| ----------- | ------------- | -------------------------- |
| `title`     | string        | 정제 제목 (볼트 §3.3 규칙) |
| `type`      | enum          | 페이지 역할 축             |
| `domain`    | enum          | 볼트 도메인 축             |
| `tags`      | list          | 중간 입자도 2~4개          |
| `updated`   | `YYYY-MM-DD`  |                            |

### 축별 enum (예시 — 실제 값은 볼트 §3.1에서 읽음)

- `type` (페이지 역할): `entity | source | concept | case | retro | note | reference`
- `domain` (볼트 도메인): `enter | sana | archive | entity | project | meta`
- `medium` (소스 미디어 유형 — **`type: source`에만**): `article | video | image | podcast | paper`

### medium 적용 규칙 (v1.2.0)

- `type: source`인 페이지에만 `medium` 필드를 둔다. entity·retro·concept 등 다른 `type`에는 붙이지 않는다.
- `/ingest`는 URL host·파일 확장자·MIME 기반으로 medium을 자동 추론한다 (상세: `skills/ingest/references/classifier.md` "medium 추론" 섹션).
- v1.2.0: `medium` 누락은 ⚠️ warn(권장). enum 위반은 ❌ fail (볼트 §3.1에 medium enum 정의 시). v2.0.0에서 누락도 fail로 승격 예정.
- `note`는 사용자 작성 페이지로 `type: note`에 속한다 (미디어 유형 아님 — `medium` enum에서 제외).

### 스킬별 역할 분담

| 스킬              | medium 관련 동작                                                             |
| ----------------- | ---------------------------------------------------------------------------- |
| `/ingest`         | 자동 추론 + Step 5 템플릿에 기록 + low confidence면 Step 3 B모드 확인        |
| `/lint` Check 1   | type=source에 누락이면 warn, enum 밖이면 fail, type≠source에 있으면 stray warn |
| `/promote`        | 대상 아님 (엔티티 페이지에는 medium 없음)                                    |
| `/retro`          | 대상 아님 (회고 페이지에는 medium 없음)                                      |

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
