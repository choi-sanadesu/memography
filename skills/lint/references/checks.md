# Lint 체크 상세 규칙

각 체크의 판정 알고리즘·실패 조건·리포트 항목 명세. 대상 볼트의 `CLAUDE.md §2`·`§5.4`에 정의된 규칙을 읽어 적용한다.

---

## Check 1: Frontmatter 스키마

### 목적

모든 관리 대상 페이지가 필수 frontmatter 필드를 포함하고, enum 필드(`type`·`media`)가 유효한 값을 가지며, `summary` 길이·`tags` 개수가 규정 범위 내인지 확인.

### 대상 파일

Phase 1에서 수집한 전체 관리 대상. 단 루트 `log.md`·`index.md`·`hot.md`·`CLAUDE.md`는 meta 페이지로 동일 규칙 적용 (단 `summary` 누락은 warn 강도로 완화).

### 판정 규칙

대상 볼트의 `CLAUDE.md §2`에서 다음을 읽어와 기준값으로 사용:

- **필수 필드**: `type`, `tags`, `updated`, `summary` (볼트 §2 기준)
- **`type` enum**: 예: `inbox | entity | source | case | reference | note | meta`
- **`media` enum** (있으면): 예: `article | video | image | podcast | paper`
- **`tags` 개수 규칙**: 3~5개 권장, 7개 상한
- **`summary` 길이 규칙**: 80~120자 한 줄

볼트 §2에 특정 enum이 정의돼 있지 않으면 해당 enum **값 검증은 skip**(다른 규칙 검증은 계속). 예: §2에 media 정의가 없으면 media enum 위반 fail은 발동하지 않음.

> `title` 필드는 검증하지 않는다. 볼트 §2 필수에 없고 ingest는 관행적으로 작성하지만 lint는 누락·길이 검사 모두 skip.

### fail (❌, 심각한 위반)

한 파일에서 다음 중 하나라도 해당되면 fail:

- YAML frontmatter 블록이 없음
- YAML 파싱 에러 (닫히지 않은 대괄호, 들여쓰기 오류 등)
- 필수 필드 누락 (`type`·`tags`·`updated`·`summary`)
- `type` 값이 enum 범위 밖 (볼트 §2에 type enum 정의된 경우)
- `type: inbox` + 위치 ≠ `inbox/` (위치-타입 불일치 — Phase 1 대상 파일은 inbox 제외이므로 이 fail이 inbox 외부에서 발견되면 사고)
- `updated` 필드가 `YYYY-MM-DD` 형식 아님
- `tags` 필드가 리스트 형태 아님
- `tags` 개수 > 7 (상한 초과)
- `media` 값이 enum 범위 밖 (볼트 §2에 media enum 정의된 경우에만, null은 제외)
- `type: source` 페이지인데 `source:` URL 필드 없음 (볼트 §3 필수)

### warn (⚠️, 처리 권장)

- `summary` 길이가 80~120자 범위 밖 (`len(summary)` 기준; 한국어 문자도 1자로 카운트, 멀티바이트 의식 안 함)
- `tags` 개수 < 3 (권장 미만)

### 리포트 항목

각 위반/경고에 대해:

```
- {path}: {위반 내용} [severity: fail|warn]
  현재: {현재값}
  기대: {허용 범위 또는 올바른 형식}
  제안: {가능하면 구체적 대체값}
```

### 예시

**fail:**

```
- entities/people/뉴진스.md: `summary` 필수 필드 누락 [severity: fail]
  현재: (없음)
  기대: 80~120자 한 줄 요약 (볼트 §2)
  제안: 첫 H2 섹션의 첫 문장을 압축
```

```
- sources/2026-04-spring.md: `type: source` 페이지인데 `source:` 누락 [severity: fail]
  현재: (없음)
  기대: 원본 URL (볼트 §3)
```

**warn:**

```
- notes/디자인-사고.md: `summary` 길이 위반 [severity: warn]
  현재: 32자
  기대: 80~120자
  제안: 핵심 논점 한 줄 추가로 확장
```

---

## Check 2: 깨진 위키링크

### 목적

본문의 모든 `[[...]]` 링크 + `related:` 구조화 YAML 버킷의 모든 링크가 실제 존재하는 파일을 가리키는지 확인.

### 탐지 방식

**본문**: 정규식으로 `[[...]]` 추출:

```
\[\[([^\[\]|]+)(\|[^\[\]]+)?\]\]
```

각 매치에서 `|` 앞부분을 target으로 추출.

**`related:` 필드**: YAML 파싱 후 entities/sources/cases/references/notes 5개 버킷 각각의 리스트에서 `"[[path|display]]"` 형태 문자열 파싱. 따옴표 제거 후 동일 정규식 적용.

### 링크 해석 규칙 (Obsidian 호환)

1. target에 `/`가 포함되면 **경로 기반**: 볼트 루트 기준 상대경로로 `{target}.md` 파일 존재 확인
2. target에 `/`가 없으면 **basename 기반**: 볼트 전체에서 `{target}.md` 파일명을 검색해 단일 매치 여부 확인
3. 확장자가 이미 포함된 경우(`.md`·`.png` 등)는 그대로 사용

### 판정 규칙

다음 중 하나 해당 시 **위반(fail)**:

- 경로 기반 링크가 대응 파일 없음
- basename 기반 링크가 볼트에 없음 (매치 0개)
- basename이 볼트에 여러 개 있어 모호함 (매치 2개 이상)

`.trash/`·`inbox/` 내 파일은 해석 대상에서 제외.

### 리포트 항목

```
- {source} → [[{target}]]
  위치: 본문 | related.{bucket}
  상태: unresolved | ambiguous ({매치 수})
  후보: {basename 동명 파일이 있으면 경로 목록}
```

---

## Check 3: 고립 페이지 (백링크 0)

### 목적

어느 페이지에서도 링크되지 않는 페이지를 식별. 진입 경로가 없으면 사실상 볼트에서 사라진 페이지.

### 탐지 방식

1. 전체 `[[...]]` 링크(본문 + `related:` 양쪽) 수집 → target set 구성 (basename 및 경로 양쪽 포함)
2. 관리 대상 페이지 각각에 대해:
   - 페이지의 경로·basename이 target set에 들어 있는지 확인
   - 포함되지 않으면 **고립**

### 제외 대상 (고립 판정에서 빠짐)

엔트리 포인트 성격이라 백링크 없는 게 정상:

- `CLAUDE.md`, `index.md`, `log.md`, `hot.md`
- 폴더별 `_index.md` (`entities/_index.md`, `sources/_index.md`, `cases/_index.md`, `references/_index.md`, `notes/_index.md`)
- `meta/tags.md`

### 리포트 항목

```
- {path} (backlinks: 0)
  type: {type}
  updated: {date}
  판단 힌트: 최근 추가됐는지, 다른 페이지에서 언급되는데 링크가 빠졌는지 등
```

### 주의

고립 페이지는 **반드시 문제**가 아니다. 아직 다른 페이지에서 참조할 기회가 없었을 수 있다. 특히 신규 classify 직후 자연스럽게 발생한다. 리포트는 "확인 권장" 수준으로 내고, 일괄 삭제 제안은 금지.

---

## Check 4: inbox 잔여

### 목적

`inbox/`는 "유예 공간"이지만 누적되면 분류 부담 증가 (`CLAUDE.md §5.4`).

### 탐지 방식

`inbox/` 하위의 파일 수와 mtime 계산. 디렉토리·hidden 파일(`.`로 시작)은 제외.

### 판정

- 파일 수 0개 + stale 0건: pass
- 파일 수 ≥ 1: 정보 표시
- 파일 수 > 10: ⚠️ warn (볼트 §5.4 기본 임계값)
- mtime 기준 3일+ 잔류 1건+: ⚠️ warn (핸드오프 D6 결정)

> 핸드오프 D6: stale 임계값 3일 채택. 볼트 §5.4의 "7일"보다 엄격. 추후 볼트 SOT와 동기화 시 7일로 조정 가능.

### 리포트 항목

```
잔여: {N}개
파일 목록:
- {filename} (크기: {size}, 수정: {mtime}, {경과일}일 경과)

⚠️ {threshold} 초과 항목:
- {filename} ({경과일}일)

⚠️ 총 {N}개 — 임계 10개 초과
```

---

## Check 5: stale `updated:`

### 목적

`updated:` 필드가 오래된 페이지 식별. 내용이 현실과 어긋났을 수 있는 후보.

### 판정 기준

기본 임계값: **90일**. 스킬 인자로 조정 가능 (`--stale-days 30`).

오늘 날짜 − `updated` 필드 > 임계값이면 stale.

### 제외 대상

- `log.md` — append-only 로그라 updated는 마지막 항목 추가일.
- 폴더별 `_index.md` — 카탈로그 페이지, 자주 갱신되지 않는 게 정상.
- `meta/tags.md` — 태그 카탈로그.

### 리포트 항목

```
- {path}
  updated: {date} ({N}일 경과)
  type: {type}
  판단 힌트: 내용 재검토 또는 updated 필드만 갱신
```

---

## Check 6: 태그 카탈로그 위반

### 목적

볼트 §4: 모든 태그는 `meta/tags.md` 카탈로그에 등록된 것만 사용. 등록 외 태그가 페이지 frontmatter에 들어가면 카탈로그 드리프트 발생.

### 탐지 방식

1. `meta/tags.md` 읽고 태그 목록 파싱
   - 마크다운 표·리스트 형태 모두 지원 (구체 형식은 볼트 컨벤션 따름)
   - 정규식 또는 YAML/Markdown 파싱으로 태그 이름 추출
2. Phase 1 대상 파일 각각의 frontmatter `tags:` 리스트 추출
3. 각 태그가 카탈로그에 있는지 확인. 없으면 위반.

### 판정

- 모든 태그 카탈로그 내: pass
- 카탈로그 외 태그 1건+: ⚠️ warn

> fail이 아닌 warn — 신규 태그가 자연스럽게 추가되는 워크플로우 (사용자가 카탈로그 갱신을 잊는 경우 잡아줌).

### 리포트 항목

```
- {path}
  미등록 태그: [{tag1}, {tag2}, ...]
  현재 태그: [{전체 tags}]
  제안: meta/tags.md에 추가 또는 기존 태그로 대체
```

### 카탈로그 갱신 제안 섹션

리포트 말미에 미등록 태그 빈도순 집계 추가:

```
## 신규 태그 후보 (빈도순)

- `브랜딩`: 3개 페이지에서 사용 → meta/tags.md 추가 권장
- `메서드론`: 1개 페이지 → 단발성, 기존 `방법론` 권장
```

### 주의

`meta/tags.md` 자체가 누락된 경우: Check 6 전체를 skip + 메인 리포트에 "⚠️ meta/tags.md 부재 — Check 6 skip" 한 줄.
