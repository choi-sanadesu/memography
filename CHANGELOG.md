# Changelog

All notable changes to memography will be documented in this file.

Format: [Semantic Versioning](https://semver.org/). Each release section lists changes under `Added`, `Changed`, `Fixed`, or `Removed`.

---

## [0.1.0] - 2026-04-23

### Added

- 초기 릴리스 (alexandria-keeper v2.0.0 → memography v0.1.0 fresh start).
- 4 스킬: `ingest`, `classify`, `lint`, `query`.
- 볼트 플랫 구조 대응 (`inbox/`, `entities/{people|brands|works|concepts|media}/`, `sources/`, `cases/`, `references/`, `notes/`, `meta/`, `raw/`).
- Ingest·Classify 2단계 분리.
- frontmatter `summary` 필수화, `domain` 축 제거.
- type enum에 `inbox` 추가 (수집 시점 임시 마커, classify Step 3에서 최종 type 확정).

### Removed

- `/promote` 스킬 (classify Step 3 B모드가 신규 엔티티 생성 흡수).
- `/retro` 스킬 (운영 흐름 분리, 플러그인 범위 밖).
- 도메인 축 (`domain` 필드, `_{domain}.md` 정의 파일).
- 도메인 기반 폴더 (`02. enterwiki/` ~ `06. studio-sana/`).
- type enum: `concept`, `retro`, `image` 제거.

---

## Versioning Policy

- **MAJOR** — 하위 호환 불가 변경 (스킬 인터페이스 재설계, 볼트 구조 요구사항 변경)
- **MINOR** — 하위 호환 기능 추가 (새 스킬, 기존 스킬에 새 모드/옵션 추가)
- **PATCH** — 버그 수정, 문서 보완, 인프라 변경 (사용자 동작 변경 없음)
