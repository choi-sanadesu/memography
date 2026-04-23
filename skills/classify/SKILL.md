---
name: classify
description: Memography 볼트의 inbox에 쌓인 항목들을 일괄로 최종 폴더(entities, sources, cases, references, notes 중 하나)에 배치하는 분류 워크플로우. 신규 수집은 하지 않으며 ingest로 이미 inbox에 들어온 페이지들만 처리. 도메인 축 없음 — 폴더 위치가 분류의 단위. entities는 5개 서브폴더(people, brands, works, concepts, media)로 추가 분기. 신규 엔티티 생성·서사 변경은 사나 승인 필수(B모드), 기존 엔티티에 링크 추가·사실 누적은 자동+사후 통보. inbox 임시 type 마커를 Step 3에서 최종 type으로 확정하고, index.md 갱신과 log.md classify 이벤트 기록까지 포함. "분류해", "classify", "/classify", "인박스 정리", "분류 워크플로우", "inbox 비우기" 같은 표현이 나올 때 트리거.
---

# memography: classify

볼트 inbox/의 미분류 항목을 최종 폴더로 배치. 수집(ingest)과 분류(classify)는 별도 단계.

## 언제 쓰는가

- ingest 누적 후 일괄 정리 시점
- `/lint` 의 inbox 잔여 경고 처리 시
- 주기적 정비 (볼트 §5.2)
- ingest 직후 단일 항목 즉시 분류

## 입력

| 호출 | 동작 |
|---|---|
| `/classify` | inbox 전체 스캔 → 항목별 제안 → 일괄 처리 |
| `/classify {path}` | 특정 inbox 파일 1건만 |

## 동작 원칙

- **신규 수집 안 함.** 이미 inbox에 있는 것만 처리.
- 대상 볼트의 `CLAUDE.md §1`(폴더 구조)·`§2`(스키마)·`§3`(폴더별 역할)·`§5.2`(classify 6단계)을 읽어서 기준으로 삼는다. 하드코딩 금지.
- **type 확정**: inbox 항목의 `type: inbox` 마커를 Step 3에서 최종 값(`source` | `note` | `entity` | `case` | `reference`)으로 갱신.
- **B모드 분기**:
  - 신규 엔티티 페이지 생성 = ✋ 사나 확인
  - **서사 변경** (대상 엔티티의 핵심 정체성·역할·평가·관계 재정의) = ✋ 사나 확인
  - **사실 누적** (`related` 버킷에 새 위키링크, 본문 사실 박스에 새 줄) = 자동 + 사후 통보
  - inbox/ → 최종 폴더 이동 = 자동 (R모드 단일 확인 후)
- **파괴적 복구 금지**: Step별 부분 성공 보존.

## 6단계 워크플로우 (개요)

[references/workflow.md](references/workflow.md)에 상세 분기·에러 처리 기술. 개요:

| # | 단계 | 처리 | B모드 |
|---|---|---|---|
| 1 | inbox 전체 스캔 | `inbox/*.md` 재귀 (플랫). `type: inbox` 마커 확인 | — |
| 2 | 최종 폴더 판단 | destination + (entities면 sub) + confidence ([category.md](references/category.md)) | low confidence → ✋ |
| 3 | 이동 + type 확정 | `mv inbox/{file} {destination}/`. frontmatter `type: inbox` → 최종 값 갱신. 신규 엔티티 페이지 생성은 ✋ | 신규 엔티티 / 서사 변경 ✋ |
| 4 | related 링크 연결 | 대상 페이지의 구조화 `related:` 버킷에 위키링크 삽입. 사실 누적 자동 / 서사 변경 ✋ | 사실 누적 자동 / 서사 변경 ✋ |
| 5 | index.md 갱신 | 해당 폴더 섹션에 새 항목 라인 추가 | 자동 |
| 6 | log.md classify 이벤트 | 항목별 또는 배치 단위 1건 append | 자동 |

## 카테고리

[references/category.md](references/category.md) 참조.

- **Tier 1 (top-level destination)**: `entities | sources | cases | references | notes` (볼트 §1)
- **Tier 2 (entities/ 서브)**: `people | brands | works | concepts | media` (볼트 §1)

## 사실 누적 vs 서사 변경 (B모드 경계)

볼트 §5.2 원칙: "신규 엔티티 생성·서사 변경은 사나 승인 후. 기존 엔티티에 링크 추가·사실 누적은 자동+사후 통보."

### 사실 누적 (자동 + 사후 통보)

- `related` 버킷(entities/sources/cases/references/notes)에 새 위키링크 추가
- 본문 사실 박스/연표/리스트에 새 줄 append
- 새 source URL 연결
- 부수적 정보 한 줄 추가 (활동 기록·인용·언급 등)

→ 처리 후 완료 보고에 포함. 사나 별도 확인 안 함.

### 서사 변경 (✋ 사나 승인 필수)

- 대상 엔티티의 `## 개요` 섹션 수정
- `summary` frontmatter 필드 갱신
- 직업·역할·포지셔닝 재정의 (예: "프로듀서 → 아트디렉터")
- 평가 톤 변경 (긍정 → 비판 등)
- `aliases` 추가
- `series` 소속 변경
- 인물·브랜드 간 관계의 핵심 재해석

→ Step 3 또는 Step 4 도중 발견 시 일시 중단 → 변경 제안 표시 → ✋ 후 진행.

> 모호한 경우(예: "이 인물의 대표작 추가는 사실 누적인가 서사 변경인가?")는 보수적으로 ✋ 처리. v0.1 한계.

## 신규 엔티티 vs 기존 엔티티

- inbox 항목이 새 엔티티를 시사 → 사나에게 confirm (✋), 승인 시 `entities/{sub}/{name}.md` 생성 (Step 3)
- 기존 엔티티에 새 사실/링크 추가 → 자동 처리 + 완료 보고 (Step 4)

## 완료 출력

```
✓ classify 완료

- 처리: {N}개 inbox 항목
- 이동:
  - sources/: {n1}건
  - cases/: {n2}건
  - references/: {n3}건
  - notes/: {n4}건
  - entities/{sub}/: {n5}건
- 신규 엔티티: {K}개 ({names})
- related 링크 추가: {L}건 (자동)
- 서사 변경: {S}건 (✋ 처리)
- 갱신된 페이지: [[index]], [[log]]
- 보류 항목: {M}개 (low confidence — 다음 호출 시 재시도)
```

## 한계 (v0.1)

- alias 매칭 미지원 (뉴진스/NewJeans 동의어는 사나가 사전 정규화 또는 수동 매핑)
- 다중 시리즈 자동 인식 없음 (cases의 `series:` 필드는 수동)
- inbox 외 위치 항목 처리 안 함 (사후 정리는 lint 권장)
- 서사 변경 vs 사실 누적 경계가 모호한 케이스는 보수적 ✋ — 운영 중 재조정 필요
- entity 서브폴더 자동 생성 없음 (`entities/{new-sub}/` 신규는 사나가 볼트 구조 갱신)
