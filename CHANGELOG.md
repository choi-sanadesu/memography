## 2.0.0 (2026-04-21)

# Changelog

All notable changes to alexandria-keeper will be documented in this file.

Format: [Semantic Versioning](https://semver.org/). Each release section lists changes under `Added`, `Changed`, `Fixed`, or `Removed`.

---

## [2.0.0] - 2026-04-21

### Breaking Changes

- `medium` → `media` 명명 변경. 기존 볼트의 `medium:` frontmatter를 `media:`로 업데이트 필요.
  `/lint`는 잔여 `medium:` 필드를 자동 감지하지 않으므로 텍스트 검색으로 직접 확인:
  `grep -r "^medium:" <vault-root>`
- `type` enum에서 `source` 제거. 기존 `type: source` 페이지는 `type: reference`로 업데이트 필요.
  v1.2.0 Deprecated 섹션에서 예고된 변경.

### Added

- `media` null 허용 — `media: null`(또는 값 없이 키만)을 유효 상태로 처리. fail/warn 없음.
  `/ingest` B모드에서 사용자가 분류 보류 시 `media: null`로 기록.

### Changed

- PR 템플릿 개선 — CHANGELOG 업데이트, 스킬 문서 업데이트, 스키마 변경 여부, Breaking change 여부 체크리스트 추가.
- `/lint` Check 1: `type: source` → `type: reference` 조건으로 전환.
- `/ingest` Step 5 템플릿: `type: source` → `type: reference`, `medium:` → `media:`.

### Migration Guide

1. 볼트 내 `type: source` 페이지를 `type: reference`로 일괄 수정.
2. 볼트 내 `medium:` frontmatter 키를 `media:`로 일괄 수정.
3. `grep -r "^medium:" <vault-root>` 로 잔여 `medium:` 필드 확인 후 수동 수정.

---

## [1.2.0] - 2026-04-21

### Added

- `/query` — 자연어 질의 스킬. 볼트 내 엔티티·소스를 frontmatter → 백링크 → 본문 순(sequential with fallback)으로 탐색해 파일 근거와 함께 답한다. 읽기 전용, 근거 없으면 "모름" 반환으로 할루시네이션 방지.
- 소스 페이지(`type: source`)용 `medium` frontmatter 필드 — 소스의 미디어 유형 축(`article | video | image | podcast | paper`). 기존 `type`(페이지 역할)과는 별개 축. `note`는 사용자 작성 페이지로 `type` 쪽에 유지.
- `/ingest`의 medium 자동 추론 — URL host / 파일 확장자 / MIME 기반 판정. host-first 우선순위 (youtube·vimeo→video, spotify podcasts·apple podcast→podcast, arxiv·doi→paper). 상세 규칙은 `skills/ingest/references/classifier.md` 참조.
- `/lint` Check 1에 `medium` 검증 추가 — `type: source`에 누락 시 ⚠️ warn, enum 위반 시 ❌ fail (단 볼트 `CLAUDE.md §3.1`에 medium enum이 정의된 경우에만). `type≠source`인데 `medium` 있으면 stray warn.
- 플러그인 `CLAUDE.md`에 "Frontmatter 스키마 참조" 섹션 — 볼트 `CLAUDE.md §3.1`을 SOT로 유지하되 운영 요약본 제공.

### Changed

- `/ingest` 이미지 파일 수용 확장 — `.png`/`.jpg`/`.jpeg`/`.webp`/`.gif` 입력 시 메타(파일명·포맷·해상도·생성일)·Claude 비전 캡션·선택 OCR을 추출해 `type: image` frontmatter의 소스 페이지 생성. 세부 절차는 `skills/ingest/references/images.md`.
- `/ingest` Step 5 소스 페이지 frontmatter 템플릿에 `medium:` 키 추가. Step 3 B모드 확인 블록이 도메인과 함께 medium을 제시.
- `/promote` 엔티티 템플릿, `/retro` 프로젝트 retro 템플릿에 "medium 비대상" 주석 1행 — 엔티티·회고 페이지에는 `medium` 필드를 두지 않는다.

### Deprecated

- (v2.0.0 예고) `type: source` 페이지의 `medium` 누락은 v2.0.0에서 warning → fail로 승격 예정. v1.2.0은 단계적 마이그레이션을 위해 warning만 출력한다.

### Notes

- **Breaking 여부**: v1.2.0 자체는 **비-breaking**. 기존 `type: source` 페이지는 `/lint`가 warning만 출력하므로 릴리스 즉시 파괴되지 않는다. v2.0.0에서 fail로 전환되는 시점에 breaking으로 간주.
- 볼트 `CLAUDE.md §3.1`의 medium enum 정의가 없으면 `/lint`의 medium **enum 위반 검증(fail 경로)**은 skip. 누락 감지(warn)는 정상 작동. 볼트 §3.1 업데이트 시 자동 활성화 — 스킬 코드 수정 불필요.
- `plugin.json` 버전은 CI 태깅(`workflow_dispatch`) 시 자동 갱신되므로 본 PR에서 수동 편집하지 않는다.

---

## [1.1.2] - 2026-04-21

### Infrastructure
- 릴리스 레포 이관 및 템플릿 인프라 승계: `SECURITY.md`, `.editorconfig`, `.gitattributes`, 강화된 `.gitignore`, `.github/ISSUE_TEMPLATE/`, `dependabot.yml` 도입
- CI 전환: `markdownlint` + `yamllint` → pyyaml 기반 엄격 YAML 검증 + JSON 검증(`include_hidden=True`)
- PR 템플릿 영문 제네릭 버전으로 교체

---

## [1.1.1] - 2025-04-17

### Changed
- `/ingest` 보안 제약 섹션 문서화 (외부 소스 수집 시 허용 범위 명시)

### Infrastructure
- GitHub Actions 추가: PR 자동 라벨링, Claude PR Assistant, Claude Code Review

---

## [1.1.0] - 2025-04-17

### Added
- `/ingest` 배치 처리 모드 — 여러 소스를 한 번에 수집하는 워크플로우 지원

---

## [1.0.0] - 2025-04-17

### Added
- `/lint` — 볼트 헬스체크 (frontmatter 규칙, 깨진 위키링크, 고립 페이지, inbox 잔여, stale `updated:` 필드, 태그 일관성 6개 규칙)
- `/ingest` — 소스 추가 표준 워크플로우 (URL/파일 입력, Chrome MCP 연동)
- `/promote` — 엔티티 승격 + 언급된 소스 페이지에 백링크 자동 업데이트
- `/retro` — 회고 항목 기록 (`log.md` 전역 + 프로젝트별 `retros/`)

---

## Versioning Policy

- **MAJOR** — 하위 호환 불가 변경 (스킬 인터페이스 재설계, 볼트 구조 요구사항 변경)
- **MINOR** — 하위 호환 기능 추가 (새 스킬, 기존 스킬에 새 모드/옵션 추가)
- **PATCH** — 버그 수정, 문서 보완, 인프라 변경 (사용자 동작 변경 없음)
