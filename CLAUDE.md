# Memography

Memography Obsidian 지식 볼트의 무결성과 구조를 유지하는 Claude Code 플러그인.

제공 스킬:

- `/memography:lint` — 볼트 헬스체크
- `/memography:ingest` — 새 소스를 inbox에 수집
- `/memography:classify` — inbox 항목을 최종 폴더로 분류
- `/memography:query` — 자연어 질의 → 파일 근거 조회

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

## Frontmatter 스키마 참조 (볼트 §2 mirror)

SOT는 대상 볼트의 `CLAUDE.md §2`. 플러그인 스킬은 해당 규칙을 **실행 시 동적으로 읽어서** 검증·생성한다 (enum 하드코딩 금지 — `skills/lint/SKILL.md` 동작 원칙). 아래는 운영 요약본.

### 필수 필드

| 필드 | 유효값 |
|------|--------|
| `type` | `inbox \| entity \| source \| case \| reference \| note \| meta` |
| `tags` | `meta/tags.md` 카탈로그 기준, 3~5개 권장, 7개 상한 |
| `updated` | `YYYY-MM-DD` |
| `summary` | 80~120자 한 줄 요약 |

**`type: inbox`**: ingest가 수집한 미분류 항목의 임시 마커. 인박스 폴더(`inbox/`)에만 존재. classify Step 3에서 최종 type(`source` | `note` | `entity` | `case` | `reference`)으로 확정.

### 선택 필드

| 필드 | 설명 |
|------|------|
| `title` | 사람 가독성 제목. lint는 검증 안 함, ingest는 작성 |
| `source` | 원본 URL (`type: source` 시 필수) |
| `media` | `article \| video \| image \| podcast \| paper`. null 허용 |
| `aliases` | entity 페이지 별칭 리스트 |
| `series` | 시리즈 소속 케이스 (예: `studio-sana`) |
| `related` | 구조화 위키링크 딕셔너리 |

### related 형식

```yaml
related:
  entities: []
  sources: []
  cases: []
  references: []
  notes: []
```

### 원칙

- inbox 단계 파일은 `type: inbox` (임시 마커). 폴더 위치(`inbox/`)와 type 값이 일치해야 함. classify Step 3에서 최종 type 확정
- 이미지는 `type: source` + `media: image`. 옛 `type: image`는 폐기
- enum 위반은 ❌ fail. 필수 필드 누락은 ❌ fail. summary 80~120자 위반은 ⚠️ warn
- `type: inbox` + 위치 ≠ `inbox/` → ❌ fail (위치-타입 불일치). lint Phase 1에서 inbox/ 제외이므로 이 fail은 inbox 외부에서만 발생

### 스킬별 역할 분담

| 스킬 | media·schema 관련 동작 |
|------|------------------------|
| `/ingest` | 자동 추론(URL host·확장자·MIME) + Step 4 inbox 페이지 작성 시 frontmatter에 기록. low confidence면 ✋ 확인 |
| `/classify` | inbox 항목 최종 폴더 배치. 신규 엔티티 생성 시 ✋. type 확정. `related` 구조화 링크 작성 |
| `/lint` Check 1 | 볼트 §2 enum 동적 읽기. enum 밖이면 fail. summary 길이 검증. tags 개수 검증 |
| `/query` | frontmatter → related → 본문 3단 검색 |

## Adding a skill

1. `skills/<skill-name>/SKILL.md` 생성
2. frontmatter에 `description:` 필수 (Claude가 스킬 호출 시점을 이 필드로 판단)
3. `/memography:<skill-name>`로 호출
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
