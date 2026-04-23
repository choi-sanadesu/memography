# Query 탐색 전략 — retrieval.md 매핑 + 자연어 매핑 휴리스틱

볼트 `.claude/retrieval.md`가 SOT. 이 문서는 (a) Phase별 스킬 동작 매핑, (b) 자연어 질의를 retrieval.md Phase 1 질문 유형 표에 매핑하는 휴리스틱을 다룬다.

핵심 원칙:

- **순차 진행·조기 종료** — Phase 0~3을 순서대로 진행하되, 충분한 증거가 나오면 다음 Phase로 넘어가지 않는다.
- **File-back 필수** — 모든 주장은 실제 파일 발췌로 지탱한다.
- **retrieval.md SOT** — 본 문서가 retrieval.md와 충돌하면 retrieval.md를 따른다 (스킬은 동적 로드).

---

## Phase 0 — 세션 시작 매핑

### retrieval.md 정의

> - 사나가 특정 페이지·엔티티·폴더·태그·inbox 지정 시: 직행
> - 그 외: `hot.md` 로드

### 스킬 동작

질의 파싱 단계에서 직접 지정 여부 판단:

| 질의 패턴 | Phase 0 동작 |
|---|---|
| `[[entities/people/뉴진스]] 알려줘` 같은 명시적 위키링크 | 해당 파일 직접 read, hot.md 건너뜀 |
| `entities/people/ 폴더에서` 같은 폴더 명시 | 해당 폴더 진입, Phase 1 폴더 매핑 건너뜀 |
| `#비주얼디렉션 태그 페이지` 같은 태그 명시 | 태그 grep 또는 `meta/tags.md` 참조, Phase 2로 |
| `inbox에서` 같은 명시 | inbox 직접 진입 (Phase 1 기본은 inbox 제외) |
| `Smerz가 뭐야?`처럼 일반 질의 | hot.md 먼저 read |

### hot.md read 후

hot.md의 Pinned·Auto·Manual 섹션에서 질의 키워드 매칭:

- 매칭되면 hot.md 해당 항목의 위키링크에서 출발 → Phase 1 건너뛰고 Phase 2로
- 매칭 안 되면 Phase 1 진입

---

## Phase 1 — 자연어 질의 → 질문 유형 매핑

### retrieval.md 정의

> | 질문 유형 | 진입점 |
> |---|---|
> | hot에 다룬 주제 | hot 링크에서 출발 |
> | 특정 인물·브랜드·작품·개념·미디어 | `entities/{카테고리}/{이름}` |
> | 사나 작업 서사 | `cases/` |
> | 특정 원본(인터뷰·영상·아티클) | `sources/` |
> | 방법론·가이드·인사이트 | `notes/` |
> | 큐레이션 모음 | `references/` |
> | inbox | Phase 0 직접 지정에서만 진입 |
> | raw | 리트리벌 대상 아님 |

### 자연어 매핑 휴리스틱

질의에서 키워드·의도를 뽑아 위 표에 매핑:

| 질의 키워드 / 의도 | 매핑 | 진입점 |
|---|---|---|
| 사람 이름·고유명사 (대문자 토큰·한글 명사구) | 인물 | `entities/people/{name}` |
| "그룹", "아이돌", "팀" + 고유명사 | 인물 (그룹) | `entities/people/{group}` (볼트 §1: 그룹은 people에 포함) |
| 브랜드명, "레이블", "에이전시", "스튜디오" | 브랜드 | `entities/brands/{name}` |
| "앨범", "MV", "전시", "영화", "책" + 작품명 | 작품 | `entities/works/{name}` |
| "디자인", "프레임워크", "이론", "방법론" + 용어 | 개념 | `entities/concepts/{term}` |
| "매거진", "플랫폼", "Spotify", "Instagram" 류 | 미디어 | `entities/media/{name}` |
| "내가 한 작업", "사나의 케이스", "studio-sana" | 사나 작업 | `cases/` |
| 특정 URL·기사 제목·인터뷰·영상 언급 | 소스 | `sources/` |
| "어떻게", "방법", "가이드", "인사이트" | 방법론 | `notes/` |
| "추천", "큐레이션", "참고할 만한" | 레퍼런스 | `references/` |

### 진입 절차

1. 질의에서 추출한 키워드로 위 표 매핑 → 진입점 폴더 결정
2. 해당 폴더에서 파일명·basename으로 직접 매칭 시도:
   - 정확 매칭 → 파일 직접 read (Phase 2 시작)
   - 부분 매칭 또는 0건 → 폴더 `_index.md` 폴백 (Phase 2 폴백 규칙)
3. 매핑이 모호하면 (인물인지 브랜드인지 불명확 등) → 가장 가능성 높은 1개 진입점 우선, 결과 부족 시 다른 진입점 추가 시도

### 매핑 실패 시

자연어 질의가 어느 유형에도 명확히 매핑되지 않으면 → 루트 `index.md` 카탈로그를 entry point로 사용 (Phase 2 폴백). 사나 검증 패턴 정당화.

---

## Phase 2 — 확장

### retrieval.md 정의

> - hop 상한: 2
> - 로드 깊이: L1(제목+frontmatter) → 관련성 있으면 L2(본문) 승격
> - 누적 토큰 ~5k 초과 시 중단
> - 폴백: 폴더 `_index.md` → 루트 `index.md`

### 스킬 동작

#### Hop 추적

Phase 1 진입점에서 출발해 hop 2까지 확장:

- **hop 1**: 진입점 페이지의 `related:` 5개 버킷 (entities/sources/cases/references/notes) + 본문 `[[...]]` 위키링크 수집
- **hop 2**: hop 1 페이지들의 `related:` 5개 버킷 + 본문 위키링크 수집
- **hop 3+**: **금지**. retrieval.md 한도 준수.

각 hop마다 관련성 평가:
- 질의 키워드가 frontmatter `title`·`tags`·`aliases`·`summary`에 있음 → 관련성 high → L2(본문) 승격
- frontmatter에 없지만 hop으로 연결 → L1 유지 (제목+frontmatter만 메모)

#### 로드 깊이

| 깊이 | 로드 내용 | 비용 |
|---|---|---|
| L1 | frontmatter + 첫 H1·H2 (~200토큰) | 낮음 |
| L2 | 본문 전체 (~500~2000토큰) | 중간 |

L1 → L2 승격 트리거:
- frontmatter 키워드 매칭 + hop 1 이내
- 또는 사용자 질의가 본문 인용을 명시적으로 요구 ("어떻게 설명했어?", "원문 그대로")

#### 토큰 한도 (5k)

누적 토큰 카운트 (대략):
- L1 페이지 5~10개
- 또는 L2 페이지 2~3개

5k 초과 직전 중단 → 그 시점까지 수집된 증거로 답변 합성. 답변에 "토큰 한도 도달, 일부 페이지 미탐색" 명시.

#### 폴백

Phase 1 진입점이 매칭 실패하거나 hop 2까지 증거 부족하면:

1. **폴더 `_index.md` 폴백**: Phase 1 진입점 폴더의 `_index.md` 카탈로그 read → 후보 재추출
2. **루트 `index.md` 폴백**: 모든 폴더에서 매칭 실패하면 루트 `index.md` 전수 스캔 (사나가 실제로 사용한 패턴 — 이건 정상 폴백)

폴백도 hop 2 한도 안에서 (hop 1: index.md, hop 2: index.md가 가리키는 후보 페이지).

---

## Phase 3 — 기록

### retrieval.md 정의

> 참조한 페이지들을 `log.md`에 `query` 이벤트로 기록 (touched 집계용).

### 스킬 동작

답변 합성 후, 참조한 모든 페이지를 `log.md`에 한 건의 query 이벤트로 append:

```markdown
## [{YYYY-MM-DD}] query | {질의 한 줄}

- 참조: [[{path1}]], [[{path2}]], [[{path3}]]
- 결과: 확정 | 부분 | 모름
```

### 자동

확인 없이 `Edit`. append-only. **이 기록 누락 시 hot.md Auto 섹션 운영(7일 touched 집계)이 깨짐** — 절대 스킵 금지.

### 예외

- "모름" 결과도 기록 (참조: 없음, 결과: 모름) — 어떤 질의가 응답 못 받았는지 추적 가치
- 사나가 명시적으로 `--no-log` 같은 플래그 요청 시에만 스킵 (v0.1 미지원, 추후 검토)

---

## "모름" 반환 조건

Phase 0~2 절차 종료까지 file-back 가능한 증거가 0이면 "모름":

- Phase 0 hot.md: 매칭 0
- Phase 1 폴더 직행: 매칭 0
- Phase 2 hop 2 + 폴백 (`_index.md`·`index.md`): 매칭 0

→ [references/report-template.md](report-template.md)의 "모름" 템플릿 사용. 질의 재구성 제안 1~3개 + log.md 기록.

---

## 비용·한계 주의

- 볼트 규모가 수백 페이지 이상이라도 **Phase 1 폴더 직행**이 핵심 — 전수 스캔 필요 없음. retrieval.md 설계 의도.
- 토큰 5k 한도는 자연스러운 컷오프 — 의미 있는 답변 합성에 충분, 컨텍스트 폭주 방지.
- 의미 유사도는 처리 안 함 — 동의어·표기 변이는 entity `aliases:` 필드 또는 사나 사전 정규화에 의존.
- 영상 콘텐츠 내용 이해는 out of scope (ingest와 동일 한계).

---

## retrieval.md 부재 시 fallback

대상 볼트에 `.claude/retrieval.md`가 없으면:

1. 사나에게 "retrieval.md 부재 — fallback 모드 진행" 1회 보고
2. fallback 절차:
   - Phase 0 건너뜀 (hot.md 존재만 확인, 있으면 read)
   - Phase 1: 자연어 매핑 휴리스틱은 유지하되 frontmatter 전수 스캔으로 보강
   - Phase 2: hop·token 한도 그대로 적용
   - Phase 3: log.md 기록 그대로
3. 답변 끝에 "fallback 모드 진행" 1줄 명시 + retrieval.md 작성 권장

이 fallback은 v0.1.0 한정 안전망 — 정상 운영에선 retrieval.md 항상 존재 가정.
