# alexandria-keeper

Alexandria(구 memorial-vault) Obsidian 지식 볼트를 관리하는 Claude Code 플러그인. 볼트의 "사서" 역할을 담당.

## 목적

지식 볼트는 시간이 지날수록 무결성이 깨지기 쉽다. frontmatter 규칙 위반, 깨진 위키링크, 고립 페이지, 처리되지 않은 inbox 잔여 등이 누적된다. 이 플러그인은 그런 상태를 자동으로 감지·복원하고, 일상 운영(ingest·엔티티 승격·회고 기록)을 표준 워크플로우로 고정한다.

## 스킬

| 스킬 | 상태 | 역할 |
|---|---|---|
| `lint` | v1 | 볼트 헬스체크 (6개 규칙 기반 검사) |
| `ingest` | v1.1 | 소스 추가 표준 워크플로우 (URL/파일, Chrome MCP, 배치 처리) |
| `promote` | v1 | 엔티티 승격 + 백링크 업데이트 |
| `retro` | v1 | 회고 항목 기록 (log.md + 프로젝트 retros/) |

## 설치

플러그인 레지스트리에 등록되기 전 단계에서는 로컬 개발 모드로 연결:

```bash
# Claude Code 설정에서 플러그인 경로 추가
# 또는 ~/.claude/plugins/ 심볼릭 링크
```

## 설계 원칙

1. **리포트 우선, 자동 수정 금지** — v1 전 스킬은 제안만. 자동 수정은 C모드 전환 후.
2. **Obsidian 친화** — 파일 구조·YAML frontmatter·위키링크 기반. Obsidian 표준 준수.
3. **Claude in Chrome 연동** — 로그인 필요 소스(인스타·트위터 등)는 Chrome MCP 경유.
4. **볼트 규칙은 볼트에 산다** — 도메인 값·frontmatter 스키마 등은 대상 볼트의 `CLAUDE.md`에서 읽어온다. 플러그인은 규칙을 하드코딩하지 않는다.

## 관련

- 대상 볼트: [memorial-vault](https://github.com/choi-sanadesu/memorial-vault) (추후 alexandria로 리네임)
