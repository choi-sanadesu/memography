# Lint 체크 상세 규칙

각 체크의 판정 알고리즘·실패 조건·리포트 항목 명세. 대상 볼트의 `CLAUDE.md`에 정의된 규칙을 읽어 적용한다.

---

## Check 1: Frontmatter 스키마

### 목적

모든 관리 대상 페이지가 필수 frontmatter 필드를 포함하고, enum 필드(`type`·`domain`)가 유효한 값을 가지는지 확인.

### 대상 파일

Phase 1에서 수집한 전체 관리 대상. 단 루트 `log.md`·`index.md`·`CLAUDE.md`는 meta 페이지로 동일 규칙 적용.

### 판정 규칙

대상 볼트의 `CLAUDE.md §3.1`에서 다음을 읽어와 기준값으로 사용:

- **필수 필드**: `title`, `type`, `domain`, `tags`, `updated` (일반적으로 이 5개, 볼트 설정 기준)
- **`type` enum**: 예: `entity | source | concept | case | retro | note | reference`
- **`domain` enum**: 예: `enter | sana | archive | entity | project | meta`
- **`medium` enum** (있으면, `type: source`에만 적용): 예: `article | video | image | podcast | paper`
- **type별 조건부 필드**: 예: `type: source`는 `medium` 권장(v1.2.0 warn), v2.0.0에서 필수(fail 승격 예정)

볼트 §3.1에 특정 enum이 정의돼 있지 않으면 해당 enum **값 검증은 skip**(다른 규칙 검증은 계속). 예: §3.1에 medium 정의가 없으면 medium enum 위반 fail은 발동하지 않지만, medium 누락 warn은 `type: source` 조건부 규칙이 있으면 그대로 작동.

### fail (❌, 심각한 위반)

한 파일에서 다음 중 하나라도 해당되면 fail:

- YAML frontmatter 블록이 없음
- YAML 파싱 에러 (닫히지 않은 대괄호, 들여쓰기 오류 등)
- 필수 필드 누락
- `type` 값이 enum 범위 밖
- `domain` 값이 enum 범위 밖
- `updated` 필드가 `YYYY-MM-DD` 형식 아님
- `tags` 필드가 리스트 형태 아님
- `type: source`이면서 `medium` 값이 enum 범위 밖 (볼트 §3.1에 medium enum 정의된 경우에만)

### warn (⚠️, 권장 미준수 — 위반 아님)

- `type: source`이면서 `medium` 필드 없음 — v1.2.0 단계적 마이그레이션(v2.0.0에서 fail 승격 예정)
- `type ≠ source`이면서 `medium` 필드 있음 — stray field, 제거 권장

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
- 05. sanarchive/_sanarchive.md: `domain` enum 위반 [severity: fail]
  현재: sanarchive
  기대: enter | sana | archive | entity | project | meta
  제안: archive (폴더 경로 `05. sanarchive/` 기반)
```

**warn:**

```
- 02. enterwiki/wiki/음악/코펜하겐-씬.md: `medium` 권장 필드 누락 [severity: warn]
  현재: (없음)
  기대: article | video | image | podcast | paper (type: source 대상)
  제안: article (source URL host 추론)
```

---

## Check 2: 깨진 위키링크

### 목적

본문의 모든 `[[...]]` 링크가 실제 존재하는 파일을 가리키는지 확인.

### 탐지 방식

정규식으로 `[[...]]` 추출:

```
\[\[([^\[\]|]+)(\|[^\[\]]+)?\]\]
```

각 매치에서 `|` 앞부분을 target으로 추출.

### 링크 해석 규칙 (Obsidian 호환)

1. target에 `/`가 포함되면 **경로 기반**: 볼트 루트 기준 상대경로로 `{target}.md` 파일 존재 확인
2. target에 `/`가 없으면 **basename 기반**: 볼트 전체에서 `{target}.md` 파일명을 검색해 단일 매치 여부 확인
3. 확장자가 이미 포함된 경우(`.md`·`.png` 등)는 그대로 사용

### 판정 규칙

다음 중 하나 해당 시 **위반(fail)**:

- 경로 기반 링크가 대응 파일 없음
- basename 기반 링크가 볼트에 없음 (매치 0개)
- basename이 볼트에 여러 개 있어 모호함 (매치 2개 이상 — Obsidian이 경고를 띄우는 상태)

`.trash/` 내 파일은 해석 대상에서 제외(삭제 예정이므로 link target으로 간주하지 않음).

### 리포트 항목

```
- {source}: [[{target}]]
  상태: unresolved | ambiguous ({매치 수})
  후보: {basename 동명 파일이 있으면 경로 목록}
```

### 예시 (이번 세션 실사례)

```
- index.md: [[06. studio-sana/rams/MEMORY]]
  상태: unresolved
  후보: 없음
  제안: 파일 생성 또는 링크 제거 (핸드오프 처리 시점에 결정)
```

---

## Check 3: 고립 페이지 (백링크 0)

### 목적

어느 페이지에서도 링크되지 않는 페이지를 식별. 진입 경로가 없으면 사실상 볼트에서 사라진 페이지.

### 탐지 방식

1. 전체 `[[...]]` 링크 수집 → target set 구성 (basename 및 경로 양쪽 포함)
2. 관리 대상 페이지 각각에 대해:
   - 페이지의 경로·basename이 target set에 들어 있는지 확인
   - 포함되지 않으면 **고립**

### 제외 대상 (고립 판정에서 빠짐)

엔트리 포인트 성격이라 백링크 없는 게 정상:

- `CLAUDE.md`
- `index.md`
- `log.md`
- `_{domain}.md` 도메인 정의 파일들

### 리포트 항목

```
- {path} (backlinks: 0)
  domain: {domain}
  updated: {date}
  판단 힌트: 최근 추가됐는지, 다른 페이지에서 언급되는데 링크가 빠졌는지 등
```

### 주의

고립 페이지는 **반드시 문제**가 아니다. 아직 다른 페이지에서 참조할 기회가 없었을 수 있다. 특히 신규 ingest 직후 자연스럽게 발생한다. 리포트는 "확인 권장" 수준으로 내고, 일괄 삭제 제안은 금지.

---

## Check 4: inbox 잔여

### 목적

`01. inbox/`는 "항상 비우는 것을 목표"(`CLAUDE.md §5.1`). 잔여 항목이 있으면 리포트.

### 탐지 방식

`01. inbox/` 하위의 파일 수를 계산. 디렉토리·hidden 파일(`.`로 시작)은 제외.

### 판정

- 0개: pass
- 1개 이상: warn (위반은 아님, 처리 권장)

### 리포트 항목

```
잔여: {N}개
파일 목록:
- {filename} (크기: {size}, 수정: {mtime})
```

---

## Check 5: stale `updated:`

### 목적

`updated:` 필드가 오래된 페이지 식별. 내용이 현실과 어긋났을 수 있는 후보.

### 판정 기준

기본 임계값: **90일**. 스킬 인자로 조정 가능 (`--stale-days 30`).

오늘 날짜 − `updated` 필드 > 임계값이면 stale.

### 제외 대상

- `log.md` — append-only 로그라 updated는 마지막 항목 추가일. 의미 있지만 여기서는 스킵.
- 도메인 정의 `_*.md` — 규칙 문서라 자주 갱신되지 않는 게 정상. 별도 임계값(예: 180일)을 적용하거나 제외.

### 리포트 항목

```
- {path}
  updated: {date} ({N}일 경과)
  domain: {domain}
  판단 힌트: 내용 재검토 또는 updated 필드만 갱신
```

---

## Check 6: 엔티티 승격 후보

### 목적

같은 인물·브랜드·작품이 여러 페이지에서 반복 언급되는데 `04. entities/`에 독립 페이지가 없는 경우 식별. `CLAUDE.md §8`의 "단일 진실 소스" 원칙 유지.

### 탐지 방식 (LLM 휴리스틱)

이 체크는 결정적 규칙으로 완전히 자동화되지 않는다. 다음 휴리스틱 조합:

1. **볼드 표기 이름 추출**: 페이지 본문에서 `**이름**` 또는 `_이름_` 패턴의 고유명사 후보 수집
2. **반복 언급 카운트**: 같은 이름이 2개 이상의 서로 다른 도메인 페이지에 등장
3. **엔티티 페이지 부재 확인**: `04. entities/**/`에 해당 이름의 `.md` 파일이 없음
4. **단발성 제외**: 한 페이지 안에서만 언급되는 이름은 후보에서 제외

### 제외 대상

- 이미 `04. entities/`에 존재하는 이름
- 일반 명사·용어 (LLM이 판단)

### 리포트 항목

```
- {이름}
  등장 페이지: [{page1}, {page2}, ...]
  도메인 분포: {enter: N, sana: N, ...}
  제안: 04. entities/{카테고리}/{이름}.md 생성
```

### 주의

false positive 빈발 영역. 리포트는 "검토 권장 후보"로 표시하고, 자동 생성 제안은 금지. 사용자가 판단해 `promote` 스킬(v2 예정)로 처리.
