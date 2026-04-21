# Changelog

All notable changes to alexandria-keeper will be documented in this file.

Format: [Semantic Versioning](https://semver.org/). Each release section lists changes under `Added`, `Changed`, `Fixed`, or `Removed`.

---

## [1.2.0] - 2026-04-21

### Changed
- `/ingest` 이미지 파일 수용 확장 — `.png`/`.jpg`/`.jpeg`/`.webp`/`.gif` 입력 시 메타(파일명·포맷·해상도·생성일)·Claude 비전 캡션·선택 OCR을 추출해 `type: image` frontmatter의 소스 페이지 생성. 세부 절차는 `skills/ingest/references/images.md`.

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
