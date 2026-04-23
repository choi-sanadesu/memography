# Classify 6단계 상세 워크플로우

볼트 `CLAUDE.md §5.2`와 1:1 대응. 각 단계의 입력·처리·출력·에러 처리.

---

## Step 1. inbox 전체 스캔

### 처리

`inbox/**/*.md` **재귀**. 플랫 구조가 SOT지만, 옛 데이터 잔재(`inbox/sources/`·`inbox/notes/` 서브폴더에 남은 파일) 안전망 차원에서 재귀로 모든 .md 포함.

| 제외 | 이유 |
|---|---|
| 숨김 파일 (`.`로 시작) | 시스템 파일 |
| `.obsidian/` 등 도구 디렉토리 | Obsidian 메타 |

### 서브폴더 발견 시

`inbox/sources/` 또는 `inbox/notes/` 같은 옛 서브폴더에 .md 파일 발견하면:

1. 해당 파일 포함 처리 (재귀 스캔)
2. Step 2~3에서 정상 분류 → 최종 폴더로 mv (서브폴더 비워짐)
3. 사나에게 사후 보고: "옛 inbox 서브폴더 잔재 N건 정리 — 서브폴더 자체는 빈 채로 남음, 수동 삭제 권장"

### type 마커 확인

각 inbox 파일의 frontmatter `type:` 필드 검사:

- `type: inbox` → 정상, Step 2로 진행
- 다른 type (옛 데이터 — `source`/`note`/...) → fail-fast 후 사나 통보:

```
⚠️ inbox/{file}.md: type 마커 불일치
- 현재: {type}
- 기대: inbox
- 원인 추정: 옛 ingest 결과 또는 수동 작성

이 파일을 어떻게 처리할까요?
1. type을 `inbox`로 강제 변경 후 분류 진행
2. 이 파일 건너뛰고 나머지 진행
3. 전체 중단 (수동 검토)
```

### 출력

```python
[
  {
    "path": "inbox/2026-04-23-newjeans-comeback.md",
    "type": "inbox",
    "summary": "NewJeans 컴백 인터뷰...",
    "source": "https://...",
    "media": "article",
    "tags": ["...", "..."]
  },
  ...
]
```

---

## Step 2. 최종 폴더 판단

### 처리

각 항목에 [category.md](category.md) 결정 트리 적용. 입력:

- frontmatter `type` (항상 `inbox`)
- `summary`
- 본문 키워드 (필요 시 본문 일부 read)
- `source` URL (있으면)
- `tags`
- `media` (Tier 1 판단 보조 — 예: `media: image` + 재사용 가치 → 보통 entities/media/ 후보 아님)

출력:

```python
{
  "destination": "sources",  # entities | sources | cases | references | notes
  "sub": null,               # entities일 때만 (people/brands/works/concepts/media)
  "confidence": "high",      # high | medium | low
  "basis": "외부 source URL + 인물 언급 → sources/"
}
```

### 배치 뷰 (N≥3)

```
## R1 분류 제안 ({N}개)

| # | inbox 파일 | summary | → 목적지 | confidence | 근거 |
|---|---|---|---|---|---|
| 1 | 2026-04-23-newjeans-comeback | NewJeans 컴백... | sources/ | high | source URL + 외부 인물 |
| 2 | 2026-04-23-design-method | 디자인 사고 방법... | notes/ | high | 방법론, source 없음 |
| 3 | 2026-04-23-spring-moodboard | 스프링 무드보드... | references/ | low | media:image, 큐레이션 vs sources |
| 4 | 2026-04-23-erika-de-casier | Erika de Casier 신곡... | entities/people/ | medium | 새 엔티티 시사 (✋ 신규 생성) |
| 5 | ... | ... | ... | ... | ... |

⚠️ low confidence: #3 — references vs sources, 사나 판단 필요
⚠️ 신규 엔티티 시사: #4 — 사나 승인 후 생성

---

**전체 진행 OK?** 개별 조정: "#3은 sources로", "#4 신규 생성 OK".
```

### 분기

| 사용자 응답 | 처리 |
|---|---|
| 전체 승인 (low 항목 명시 처리 후) | Step 3 진입 |
| 개별 override | 해당 항목 destination 변경 후 R1 재제시 |
| 개별 보류 | `{items}`에서 제거 (다음 classify 호출 시 재처리) |
| 전체 중단 | 아무 파일도 건드리지 않고 종료 |

---

## Step 3. 이동 + type 확정

각 항목에 대해 sequential 실행:

### 3-1. 파일 이동

```bash
# Bash mv inbox/{file}.md {destination}/{sub?}/{file}.md
```

destination에 따라:

| destination | 경로 |
|---|---|
| sources | `sources/{file}.md` |
| cases | `cases/{file}.md` |
| references | `references/{file}.md` |
| notes | `notes/{file}.md` |
| entities | `entities/{sub}/{file}.md` (sub: people/brands/works/concepts/media) |

파일명 충돌 시 ` (2)` suffix 회피.

### 3-2. type frontmatter 갱신

`Edit`로 frontmatter `type: inbox` → 최종 값:

| destination | 새 type |
|---|---|
| sources | `source` |
| cases | `case` |
| references | `reference` |
| notes | `note` |
| entities | `entity` |

### 3-3. 신규 엔티티 페이지 생성 (entities/ 직행 + 신규 엔티티)

inbox 항목이 새 엔티티 자체를 정의하는 경우 (entities/{sub}/{name}.md 생성):

**B모드 ✋**:

```
## 신규 엔티티 생성 제안

inbox/{file}.md → entities/{sub}/{name}.md

- 이름: {name}
- 카테고리: {sub} (people | brands | works | concepts | media)
- 근거: {summary 인용}
- aliases 후보: [{한글 표기}, {약칭}]
- summary: {80~120자}

생성해도 될까요?
```

confirm 후 inbox 파일을 entities/{sub}/{name}.md로 이동(또는 별도 작성), type = `entity`로 갱신, 아래 템플릿 적용:

```yaml
---
title: {정제된 이름}
type: entity
tags: [{2-3 from inbox 항목 컨텍스트}]
updated: {YYYY-MM-DD}
summary: {80-120자 — inbox 항목 컨텍스트로 합성}
aliases: [{한글 표기 등}]
related:
  entities: []
  sources: [{inbox 항목 출처면 그 링크}]
  cases: []
  references: []
  notes: []
---

# {name}

## 개요
{1-2문단 — inbox 항목 요약 기반}

## 상세
{비워둠 — 사나가 점진적으로 채움}
```

### 출력

각 항목 처리 결과:

```
✓ {original_path} → {new_path} (type: inbox → {new_type})
```

---

## Step 4. related 링크 연결

이동된 항목 각각에 대해 `related:` 구조화 필드 작성/갱신.

### 처리

1. 페이지 본문에서 위키링크 후보 추출 (`[[...]]`)
2. source URL이 있으면 그 source가 어떤 엔티티/케이스를 가리키는지 추정
3. 후보 각각에 대해:
   - 대상 파일이 어느 폴더(`entities/`/`sources/`/...)에 있는지 확인
   - 해당하는 `related.{bucket}` 리스트에 `"[[path|display]]"` 형태로 append (이미 있으면 skip)

### 본문 `## 관련` 섹션 vs 구조화 `related:` 필드

**볼트 §2 D12**: 그래프 데이터는 구조화 `related:` YAML 필드가 SOT. 본문 `## 관련` 섹션은 사람 가독용 보조 (선택).

→ classify는 **`related:` YAML에 작성**. 본문 `## 관련` 섹션은 건드리지 않음 (이미 있으면 유지, 없으면 만들지 않음).

### 사실 누적 vs 서사 변경 분기

이동된 항목이 다른 페이지(특히 entities)를 언급하면, 그 다른 페이지 쪽에도 역방향 링크 추가가 필요할 수 있음.

**사실 누적 (자동)**:
- 대상 엔티티 페이지의 `related.sources` (또는 적절한 버킷)에 새 위키링크 한 줄 append
- 본문에 사실 박스/연표 섹션 있으면 새 줄 한 개 append (예: "2026-04: {사건} ([[source]])")

**서사 변경 (✋)**:
- 대상 엔티티 페이지의 `## 개요` 섹션 수정 필요? → ✋
- `summary` 갱신? → ✋
- `aliases` 추가? → ✋

판단 모호 시 보수적으로 ✋.

### B모드 (서사 변경 시)

```
## 서사 변경 제안

대상: entities/people/{name}.md

inbox 항목 내용으로 다음 변경이 필요해 보임:
- {필드 또는 섹션}: 현재 → 제안

진행할까요?
```

### 자동 처리 결과 보고

```
[Auto] entities/people/{name}.md
  + related.sources: "[[sources/2026-04-23-newjeans-comeback]]"

[Auto] entities/brands/{label}.md
  + related.sources: "[[sources/2026-04-23-pdf-report]]"
  + 본문 활동 기록: "2026-04: {사건} ([[source]])"
```

### 출력

```python
{
  "auto_count": N,           # 사실 누적 자동 처리 건수
  "manual_count": M,         # 서사 변경 ✋ 처리 건수
  "skipped": [...]           # 사용자가 거부한 변경
}
```

---

## Step 5. index.md 갱신

이동된 각 항목에 대해 `index.md`의 해당 폴더 섹션에 한 줄 추가:

```markdown
- [[{path}|{title}]] — {summary 짧게}
```

### 섹션 위치

볼트 `index.md` 구조 가정:

```markdown
## entities
- [[entities/people/뉴진스|뉴진스]] — ...

## sources
- [[sources/2026-04-23-newjeans-comeback|NewJeans 컴백]] — ...

## cases
- ...
```

해당 섹션이 없으면: 알파벳 순으로 새 섹션 생성 + 사나에게 통보.

### 자동

확인 없이 `Edit`. 볼트 §6 규칙.

---

## Step 6. log.md classify 이벤트

### 단일 항목 처리

```markdown
## [{YYYY-MM-DD}] classify | {title}

- 이동: inbox/{file} → {destination}/{sub?}/{file}
- type: inbox → {new_type}
- related 추가: {N}건 (자동) / {M}건 (✋ 처리)
- 신규 엔티티: {name} (있으면)
```

### 배치 처리

```markdown
## [{YYYY-MM-DD}] classify | 배치 ({N}건)

- 처리: {N}개 inbox 항목
- 이동: sources({n1}), cases({n2}), references({n3}), notes({n4}), entities({n5})
- 신규 엔티티: {K}개 — {names}
- related 추가: {L}건 (자동) / {S}건 (✋)
- 보류: {M}개 (low confidence)
```

### 자동

확인 없이 `Edit`. append-only.

---

## 완료 출력

```
✓ classify 완료

- 처리: {N}개
- 이동: sources({n1}), cases({n2}), references({n3}), notes({n4}), entities({n5})
- 신규 엔티티: {K}개 ({names})
- related 자동: {L}건
- 서사 변경 ✋: {S}건
- 갱신된 페이지: [[index]], [[log]]
- 보류: {M}개 (low confidence — 다음 호출 시 재시도)
```

---

## 에러 복구 (파괴적 복구 금지)

| 실패 단계 | 상태 | 처리 |
|---|---|---|
| Step 1 | 변경 없음 | 에러 출력만 |
| Step 2 | 변경 없음 | 에러 출력만 |
| Step 3 (mv 실패) | inbox에 그대로 | 원인 보고, 다음 항목 진행 |
| Step 3 (type 갱신 실패) | mv는 됐음, type은 옛값 | 사나 통보, type 수동 갱신 안내 |
| Step 4 (related 갱신 실패) | 항목 이동·type은 됐음 | 사나 통보, 수동 갱신 안내. 다음 항목 진행 |
| Step 5 (index 갱신 실패) | 페이지는 이동됨 | 사나 통보, index 수동 추가 안내 |
| Step 6 (log 실패) | 모두 처리됨 | log.md 수동 추가 안내 |

원칙: **부분 성공 상태를 그대로 두고, 다음 재호출이 이어서 처리**할 수 있게 한다.
