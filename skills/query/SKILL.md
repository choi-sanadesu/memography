---
name: query
description: Memography Obsidian 볼트에 자연어 질의를 던져 파일 근거와 함께 답을 받는다. 증거 수집 순서는 frontmatter → related → 본문 (sequential with fallback). 근거가 없으면 "모름"을 반환해 할루시네이션을 방지한다. 읽기 전용 — 볼트에 아무것도 쓰지 않는다. "질의", "물어봐", "찾아줘", "query", "/query", "볼트에서 찾아" 같은 표현이 나올 때 트리거.
---

# memography: query

자연어 질의에 대해 볼트 파일을 근거로 답한다. `frontmatter → related → 본문` 순으로 증거를 수집하고(sequential with fallback), 어느 레이어에서도 근거가 없으면 "모름"을 반환해 할루시네이션을 차단한다.

## 언제 쓰는가

- "Smerz가 뭐야?", "코펜하겐 씬 관련 페이지는?" 같이 볼트 내용을 직접 조회하고 싶을 때
- `/ingest` 이후 "방금 추가한 소스에 언급된 아티스트 다른 페이지는?" 같은 맥락 추적
- `/classify` 전에 "이 이름이 이미 볼트에 있나?" 확인
- 볼트 규모가 커져서 수동 grep·백링크 추적 비용이 부담될 때

## 입력

| 호출 | 동작 |
|---|---|
| `/query {자연어 질의}` | 질의 1건 처리 |
| `/query` | 질의 대화형 입력 유도 |

질의는 자유 자연어 문자열. 예: "Erika de Casier는 어떤 아티스트야?", "뉴진스 언급된 페이지".

## 동작 원칙

- **읽기 전용**: 볼트 파일을 절대 수정하지 않는다. `log.md`에도 기록하지 않는다.
- **File-back 필수**: 모든 답변 주장은 볼트 내 실제 파일 인용으로 뒷받침되어야 한다. 근거 없는 합성 금지.
- **Sequential with fallback**: Layer 1(frontmatter)에서 충분하면 거기서 멈춘다. 불확실할 때만 Layer 2(related), 그래도 모호하면 Layer 3(본문)으로 내려간다.
- **모름 우선**: 세 레이어 모두에서 증거가 없으면 *추측하지 말고* "모름"을 반환한다. 대신 질의 재구성 제안을 같이 낸다.
- **볼트 규칙은 볼트에 산다**: 대상 볼트의 `CLAUDE.md §2`(frontmatter 스키마)·`§3`(폴더별 역할)을 읽어와 매칭 기준으로 삼는다. 타입 enum 하드코딩 금지.

## 탐색 범위

포함:
- 루트 `CLAUDE.md`·`index.md`·`log.md`·`hot.md`
- `entities/**/*.md`, `sources/*.md`, `cases/*.md`, `references/*.md`, `notes/*.md`
- 폴더별 `_index.md`
- `meta/tags.md`

제외 (`/lint`와 동일):
- `inbox/**`, `raw/**`, `.obsidian/`, `.git/`, `.trash/`

## 4단계 워크플로우

상세 알고리즘은 [references/search-strategy.md](references/search-strategy.md) 참조.

| # | 단계 | 처리 |
|---|---|---|
| 1 | 질의 파싱 | 키워드·고유명사·폴더 힌트 추출 |
| 2 | 증거 수집 | Layer 1(frontmatter) → Layer 2(related) → Layer 3(본문), 각 레이어 confidence 기록 |
| 3 | 답변 합성 | quote-heavy 템플릿으로 발췌 + 요약 병기. 레이어별 confidence 표기 |
| 4 | 리포트 출력 | 확정 / 부분 / 모름 중 하나 |

## 증거 수집 전략

세 레이어의 판정 규칙·early-exit 조건·"모름" 조건은 [references/search-strategy.md](references/search-strategy.md).

요약:

- **Layer 1 (frontmatter)** — `title`·`tags`·`type`·`aliases` 매칭. 단일 엔티티·소스가 명확히 매칭되면 early-exit.
- **Layer 2 (related)** — Layer 1 후보의 구조화 `related:` 버킷 추적 (entities/sources/cases/references/notes 5개 버킷). 본문 `[[...]]` fallback. `entities/` 허브 중심으로 관계 해소.
- **Layer 3 (본문)** — 전 볼트 텍스트 grep + 문맥 발췌. 마지막 수단.

Layer 3까지 증거가 0이면 **"모름"**.

## 출력 포맷

quote-heavy 템플릿 3종(확정·부분·모름)은 [references/report-template.md](references/report-template.md).

## 한계 (v0.1)

- **의미 유사도·임베딩 없음** — 단순 키워드/고유명사 매칭. "비슷한 느낌의 아티스트" 같은 fuzzy 질의는 Layer 3에 의존하며 품질 낮음.
- **다중 턴 follow-up 없음** — 1 호출 = 1 질의. 후속 질문은 `/query`를 다시 호출.
- **벌크 질의 없음** — 여러 질의 동시 처리는 추후.
- **답변 캐시 없음** — 볼트는 가변이라 매 호출 fresh. 같은 질의 반복이면 사용자가 이전 답을 참고.
- **영상·이미지 내용 이해 out of scope** — 파일명·캡션·frontmatter까지만.
- **alias 매칭 단순**: `aliases:` frontmatter 필드만 확인. 동의어 사전 없음.
