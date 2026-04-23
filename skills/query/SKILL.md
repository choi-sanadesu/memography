---
name: query
description: Memography Obsidian 볼트에 자연어 질의를 던져 파일 근거와 함께 답을 받는다. 대상 볼트의 .claude/retrieval.md 프로토콜을 동적으로 읽어 Phase 0(hot.md)→Phase 1(질문 유형별 폴더 직행)→Phase 2(확장, hop·token 한도)→Phase 3(log.md 기록) 순으로 진행. 근거 없으면 "모름" 반환해 할루시네이션 방지. 본문은 읽기 전용이지만 touched 집계용으로 log.md에는 query 이벤트 1건 기록 (hot.md Auto 섹션 운영 의존). "질의", "물어봐", "찾아줘", "query", "/query", "볼트에서 찾아" 같은 표현이 나올 때 트리거.
---

# memography: query

자연어 질의에 대해 볼트 파일을 근거로 답한다. **볼트 retrieval.md가 SOT** — query 스킬은 그 프로토콜을 실행 시 동적으로 읽어 따른다.

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

- **retrieval.md 동적 로드**: 대상 볼트의 `.claude/retrieval.md` 파일을 매 호출 시 읽어 Phase 0~3 절차를 그대로 따른다. SKILL.md에 Phase 세부를 하드코딩하지 않는다 (lint이 볼트 §2 enum 동적 읽기하는 것과 동일 원칙).
- **본문 읽기 전용**: 볼트 페이지 본문은 절대 수정하지 않는다. 단 retrieval.md Phase 3에 따라 **`log.md`에 query 이벤트 1건 append**한다 (touched 집계 — `hot.md` Auto 섹션 퇴출 기준이 이 로그에 의존).
- **File-back 필수**: 모든 답변 주장은 볼트 내 실제 파일 인용으로 뒷받침되어야 한다. 근거 없는 합성 금지.
- **모름 우선**: retrieval.md 절차 종료까지 증거가 없으면 *추측하지 말고* "모름"을 반환한다. 대신 질의 재구성 제안을 같이 낸다.
- **볼트 스키마는 볼트에 산다**: 대상 볼트의 `CLAUDE.md §2`(frontmatter 스키마)·`§3`(폴더별 역할)도 읽어와 매칭 기준으로 삼는다. 타입 enum 하드코딩 금지.

## 4단계 워크플로우 (retrieval.md SOT)

상세는 볼트 `.claude/retrieval.md` 직접 참조. 스킬은 아래 매핑으로 실행:

| Phase | retrieval.md 정의 | 스킬 동작 |
|---|---|---|
| 0 | hot.md 로드 (사나가 특정 페이지·엔티티·폴더·태그·inbox 지정 시 직행) | 질의 파싱 후 직접 지정 여부 판단 → 아니면 hot.md 먼저 read |
| 1 | 질문 유형별 폴더 직행 (entities/{카테고리}/{이름}, cases/, sources/, notes/, references/. inbox는 직접 지정에서만, raw 제외) | 자연어 질의 → 질문 유형 매핑 → 해당 폴더 진입 ([references/search-strategy.md](references/search-strategy.md)) |
| 2 | hop 상한 2, L1(제목+frontmatter) → L2(본문) 승격, 누적 토큰 ~5k 초과 시 중단. 폴백: 폴더 `_index.md` → 루트 `index.md` | Phase 1 진입점에서 관련성 높은 페이지 따라가며 확장. 한도 도달 시 중단 |
| 3 | 참조한 페이지를 `log.md`에 `query` 이벤트로 기록 (touched 집계) | 답변 후 `log.md` append |

**자연어 질의 → 질문 유형 매핑** 휴리스틱은 [references/search-strategy.md](references/search-strategy.md).

## 출력 포맷

quote-heavy 템플릿 3종(확정·부분·모름)은 [references/report-template.md](references/report-template.md). 답변 끝에 "log.md 기록 완료" 한 줄 명시.

## log.md 기록 형식

retrieval.md Phase 3 준수. 단일 항목:

```markdown
## [{YYYY-MM-DD}] query | {질의 한 줄}

- 참조: [[{path1}]], [[{path2}]], [[{path3}]]
- 결과: 확정 | 부분 | 모름
```

배치 또는 다중 참조 시: 참조 페이지 전부 나열 (touched 집계 정확도).

## 한계 (v0.1)

- **의미 유사도·임베딩 없음** — 단순 키워드/고유명사 매칭 + 폴더 직행. "비슷한 느낌의 아티스트" 같은 fuzzy 질의는 Phase 2 본문 승격으로도 품질 낮음.
- **다중 턴 follow-up 없음** — 1 호출 = 1 질의. 후속 질문은 `/query`를 다시 호출.
- **벌크 질의 없음** — 여러 질의 동시 처리는 추후.
- **답변 캐시 없음** — 볼트는 가변이라 매 호출 fresh.
- **영상·이미지 내용 이해 out of scope** — 파일명·캡션·frontmatter까지만.
- **alias 매칭 단순**: entity 페이지의 `aliases:` 필드만 확인. 동의어 사전 없음.
- **retrieval.md 부재 시**: 볼트에 `.claude/retrieval.md`가 없으면 fallback으로 frontmatter → related → 본문 grep 순으로 진행 + 사나에게 "retrieval.md 부재로 fallback 모드" 보고.
