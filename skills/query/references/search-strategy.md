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
- **폴더 힌트**: `엔티티`·`인물`·`소스`·`케이스` 같은 볼트 폴더 단서. `entities/people` 같은 명시적 경로도 인식
- **타입 힌트**: "엔티티", "소스", "노트" → 대상 볼트 type enum으로 매핑 (`inbox` 제외 — query는 inbox 탐색 안 함)
- **관계 키워드**: "언급된", "같이", "관련", "영향을 준" → Layer 2 트리거 신호

추출 결과를 `{keywords, folder_hints, type_hints, relation_cues}`로 유지.

---

## Layer 1 — Frontmatter 매칭

### 목적

가장 구조적이고 신뢰도 높은 증거. 엔티티·소스 페이지 식별에 1순위.

### 대상

- `entities/**/*.md` (엔티티 허브 — 엔티티성 질의의 1차 후보)
- `sources/*.md`, `cases/*.md`, `references/*.md`, `notes/*.md`
- 폴더별 `_index.md` (폴더 단위 질의일 때)

### 매칭 규칙

각 파일의 YAML frontmatter를 파싱해 다음을 확인:

1. **`title` 일치/포함** — 추출된 고유명사가 title에 정확 또는 부분 매칭
2. **파일명 일치/포함** — 동일 기준을 파일 basename에도 적용 (title과 다를 수 있음)
3. **`aliases` 포함** — entity 페이지의 `aliases:` 리스트에 키워드 매칭 (한글 표기 등)
4. **`tags` 포함** — 질의 키워드가 tag 리스트에 있는가
5. **`type` 필터** — type 힌트가 있으면 해당 type만 통과
6. **폴더 필터** — 폴더 힌트가 있으면 해당 폴더만 (예: "entities/people" → `entities/people/**/*.md`)

### Confidence 판정

| 조건 | confidence |
|---|---|
| 단일 후보, title/파일명 정확 매칭 또는 aliases 정확 매칭 | high |
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

## Layer 2 — Related (구조화 + 본문 백링크)

### 목적

엔티티·소스 간 관계 기반 증거. "X와 같이 언급된 Y"·"X 관련 페이지" 류 질의의 본진.

볼트 §2 D12: 그래프 데이터는 **구조화 `related:` YAML 필드가 SOT**. 본문 `[[...]]`는 보조.

### 대상

- Layer 1에서 수집된 후보의 in-related (누가 이 페이지를 `related.*` 버킷에 담고 있는가)
- 후보의 자체 `related:` 5개 버킷 (entities/sources/cases/references/notes) out-links
- 본문 `[[...]]` 위키링크 (구조화 데이터 보조)

### 매칭 규칙

1. **역방향 related 추적**: 후보 페이지 경로·파일명이 어느 페이지의 `related.{bucket}` 리스트에 들어있는지 검색 (전 볼트). YAML 파싱 후 5개 버킷 모두 확인.
2. **순방향 related 수집**: 후보 페이지 frontmatter `related:` 5개 버킷의 각 링크 수집.
3. **본문 wiki-link fallback**: 위 두 단계로 부족하면 본문 `[[...]]` 매칭으로 보강 (구조화 데이터 누락 페이지 대응).

### Confidence 판정

| 조건 | confidence |
|---|---|
| 구조화 `related:` 버킷에서 명확히 매칭 | high |
| 본문 `[[...]]` 링크로만 연결 | medium |
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

frontmatter·related 어디에도 안 잡히는 자유 키워드 질의의 마지막 수단.

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
- Layer 2 related 관계 0
- Layer 3 본문 매치 0

→ [references/report-template.md](report-template.md)의 "모름" 템플릿 사용. 질의 재구성 제안 1~3개를 함께 출력 (예: 동의어·부분어·폴더 한정 등).

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
- 구조화 `related:` 필드가 누락된 옛 페이지는 Layer 2가 본문 fallback에만 의존 → confidence 한 단계 낮음. classify로 정비 권장.
