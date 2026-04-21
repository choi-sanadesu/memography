# Query 증거 수집 전략

`/query` 스킬의 3-레이어 sequential-with-fallback 탐색 알고리즘 상세.

핵심 원칙:

- **순차 조사·조기 종료** — 상위 레이어에서 충분한 증거가 나오면 하위 레이어로 내려가지 않는다. 비용·노이즈 모두 최소화.
- **Confidence 누적** — 각 레이어 결과에 `high / medium / low / none` 표기. 답변에 그대로 노출한다.
- **File-back 필수** — 모든 주장은 실제 파일 발췌로 지탱한다. 발췌 불가능하면 해당 증거는 버린다.

---

## 사전 준비: 질의 파싱

질의 문자열에서 다음을 뽑는다:

- **고유명사 / 엔티티명**: 대문자 토큰·한글 명사구 (예: `Smerz`, `Erika de Casier`, `뉴진스`)
- **도메인 힌트**: `아카이브`·`sana`·`enter` 같은 대상 볼트 `CLAUDE.md §3.1`의 domain enum 단서
- **타입 힌트**: "아티스트", "앨범", "회고" → 대상 볼트 type enum으로 매핑
- **관계 키워드**: "언급된", "같이", "관련", "영향을 준" → Layer 2 트리거 신호

추출 결과를 `{keywords, domain_hints, type_hints, relation_cues}`로 유지.

---

## Layer 1 — Frontmatter 매칭

### 목적

가장 구조적이고 신뢰도 높은 증거. 엔티티·소스 페이지 식별에 1순위.

### 대상

- `04. entities/**/*.md` (엔티티 허브 — 엔티티성 질의의 1차 후보)
- `*/wiki/**/*.md` (소스 페이지)
- `06. studio-sana/**/*.md` (프로젝트 페이지)
- 도메인 정의 `_*.md` (도메인 질의일 때)

### 매칭 규칙

각 파일의 YAML frontmatter를 파싱해 다음을 확인:

1. **`title` 일치/포함** — 추출된 고유명사가 title에 정확 또는 부분 매칭
2. **파일명 일치/포함** — 동일 기준을 파일 basename에도 적용 (title과 다를 수 있음)
3. **`tags` 포함** — 질의 키워드가 tag 리스트에 있는가
4. **`domain` 필터** — domain 힌트가 있으면 해당 domain만 통과
5. **`type` 필터** — type 힌트가 있으면 해당 type만 통과

### Confidence 판정

| 조건 | confidence |
|---|---|
| 단일 후보, title/파일명 정확 매칭 | high |
| 단일 후보, 부분 매칭 또는 tag 매칭 | medium |
| 2~5개 후보 | low (Layer 2로 내려갈 필요) |
| 6개 이상 또는 0개 | none (확장 필요) |

### Early-exit 조건

- confidence = high 이고 질의가 "X가 뭐야?"·"X에 대해" 형태의 단일 엔티티 질의 → Layer 1에서 종료, 답변 합성.
- confidence = medium 이지만 Layer 2 관계 추적이 불필요한 질의 → Layer 1에서 종료.

### Fallback 트리거

- confidence ≤ low
- 질의에 관계 키워드 포함 (관계 질의는 기본적으로 Layer 2로 확장)

---

## Layer 2 — 백링크 추적

### 목적

엔티티·소스 간 관계 기반 증거. "X와 같이 언급된 Y"·"X 관련 페이지" 류 질의의 본진.

### 대상

- Layer 1에서 수집된 후보의 in-links (누가 이 페이지를 wiki-link로 가리키는가)
- 후보의 `## 관련` 섹션 out-links
- `04. entities/**/*.md` 허브의 `## 참고 소스`·`## 언급 페이지` 섹션

### 매칭 규칙

1. **역방향 링크 추적**: 후보 페이지 경로·파일명이 포함된 `[[...]]` 위키링크를 전 볼트에서 검색 (제외 경로 제외)
2. **순방향 링크 추적**: 후보 페이지 본문의 `[[...]]` 링크 수집
3. **허브 섹션 파싱**: 엔티티 페이지라면 `## 참고 소스`·`## 관련` 섹션의 링크를 구조화해 수집

### Confidence 판정

| 조건 | confidence |
|---|---|
| 관계가 명확히 드러나는 섹션에서 매칭 | high |
| 본문 중 wiki-link로만 연결 | medium |
| 링크는 있지만 컨텍스트 모호 | low |
| 관계 증거 0 | none |

### Early-exit 조건

- confidence ≥ medium, 질의의 관계 의도가 해소됨 → Layer 2에서 종료.
- 관계 키워드가 없고 Layer 1에서 high였다면 Layer 2는 보강용으로만 쓰고 종료.

### Fallback 트리거

- confidence = none 이고 질의가 자유 키워드 탐색 성격 (관계 질의가 아님)
- Layer 1도 none 이었고 Layer 2도 none

---

## Layer 3 — 본문 텍스트 검색

### 목적

frontmatter·링크 어디에도 안 잡히는 자유 키워드 질의의 마지막 수단.

### 대상

탐색 범위 전부 (제외 경로 제외).

### 매칭 규칙

1. **Grep**: 추출된 키워드·고유명사 각각에 대해 정규식 검색 (case-insensitive 기본)
2. **문맥 수집**: 매치 주변 3~10줄을 발췌. 여러 매치가 한 문단에 모이면 합친다.
3. **노이즈 필터**: `##`·`---` 같은 구조 요소만 있는 매치는 제외. 실제 문장을 포함한 것만 증거로 채택.

### Confidence 판정

| 조건 | confidence |
|---|---|
| 키워드가 본문에 다수 등장 + 같은 페이지에 반복 | medium |
| 단발 매치 | low |
| 매치 0 | none |

Layer 3은 구조적 신뢰도가 낮아 **high는 기본적으로 부여하지 않는다.**

### "모름" 반환 조건

세 레이어 모두 confidence = none:

- Layer 1 frontmatter 매칭 0
- Layer 2 백링크 관계 0
- Layer 3 본문 매치 0

→ [references/report-template.md](report-template.md)의 "모름" 템플릿 사용. 질의 재구성 제안 1~3개를 함께 출력 (예: 동의어·부분어·도메인 한정 등).

---

## 레이어 간 누적 답변 합성

- **단일 레이어 해소**: 해당 레이어 발췌만 사용.
- **다중 레이어 해소**: 각 레이어 증거를 `report-template.md`의 `## 근거` 섹션에 레이어 라벨과 함께 병기 (`frontmatter 매칭, confidence: high` 등).
- **상충되는 증거**: 모순된 내용이 발견되면 두 발췌를 모두 인용하고 `## 주의` 섹션에 명시. 어느 쪽이 맞는지 단정 금지.

---

## 비용·한계 주의

- 볼트가 수백 페이지 이상이면 Layer 3은 비용이 커진다. 가능하면 Layer 1·2에서 끝낼 수 있는 질의 형태를 사용자에게 제안.
- 의미 유사도는 처리 안 함. "Smerz와 비슷한 아티스트"는 Layer 2 관계 증거 없으면 Layer 3까지 내려가도 low confidence가 한계.
- 영상 콘텐츠 내용 이해는 out of scope (ingest와 동일 한계).
