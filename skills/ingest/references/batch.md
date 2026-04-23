# 배치 Ingest 플로우

여러 소스를 한 번에 처리하는 모드. 단일 ingest의 5단계 중 **Step 1~2를 통합 확인**으로 압축하고, Step 3~5는 소스별 sequential 자동 실행.

---

## 활성 조건

`/ingest` 인자에 소스 2개 이상 (URL·파일 경로 혼합 가능)일 때 자동으로 배치 모드 진입.

> v0.1에서는 `--inbox-all`을 지원하지 않는다. inbox 자체가 ingest의 목적지이므로, 이미 inbox에 있는 파일을 다시 ingest할 이유가 없음 (분류는 `/classify`).

---

## 준비: 소스 수집

- 인자 순서 그대로 사용
- URL·파일 혼합 가능

수집 결과를 `{sources}` 리스트로 보유.

---

## R1. 전체 요약 + media 추론 (Step 1~2 통합)

### 처리

각 소스에 대해 순차 실행 (Chrome MCP는 싱글 세션이라 sequential 안전):

1. Step 1 (읽기) — URL/파일/Chrome 분기는 단일 모드 워크플로우 준수
2. Step 2 (요약 + media 자동 추론) — [media.md](media.md) 규칙
3. 결과를 구조화 보관

### 확인 프리젠테이션

```
## R1 소스 요약 ({N}개)

| # | 출처 | 성격 | 한 줄 요약 | media (conf) |
|---|---|---|---|---|
| 1 | instagram.com/reel/... | SNS 릴스 | {summary1} | video (high) |
| 2 | inbox/report.pdf | PDF 리포트 | {summary2} | paper (medium) |
| 3 | example.com/article | 기사 | {summary3} | article (high) |
| 4 | ... | ... | ... | ... |

각 소스 핵심 논점:

#1 {title1}:
- {point}
- {point}

#2 {title2}:
- {point}
- {point}
...

⚠️ media confidence low/medium 항목:
- #2: paper (medium) — 근거: "ext:.pdf". 대안: article. 어느 쪽?

---

**전체 방향 OK?** 특정 항목만 조정하려면 "#3 시선 다른 쪽으로" 또는 "#2 media는 article로" 같이 지시.
```

### 피드백 처리

| 사용자 응답 | 처리 |
|---|---|
| 전체 승인 | R2로 진입 |
| 특정 항목 재작업 | 해당 항목만 Step 2 재실행 후 R1 재제시 |
| 특정 항목 제외 | `{sources}`에서 제거 후 R2로 |
| media override | 해당 항목 media 변경 후 R2로 |

### 에러

- 특정 소스 Step 1 실패 → 해당 항목 제외, 다른 소스 계속. R1 결과에 실패 항목 명시.

---

## R2. 생성 예정 (Step 3~4 프리뷰)

### 처리

내부적으로 Step 3·4를 **시뮬레이션**만 (실제 파일 생성·이동 없음):

- 슬러그 LLM 정제
- 경로 결정 (`inbox/{YYYY-MM-DD}-{slug}.md`)
- 태그 추출
- 원본 처리 방식 결정 (이동 없음 / `raw/images/` 복사 / inbox 옆 복사)

### 확인 프리젠테이션

```
## R2 생성 예정 ({N}개 신규 페이지)

| # | 경로 | 제목 | type | media | 태그 | 원본 처리 |
|---|---|---|---|---|---|---|
| 1 | inbox/2026-04-23-newjeans-comeback.md | NewJeans 컴백 인터뷰 | inbox | article | [...] | URL (이동 없음) |
| 2 | inbox/2026-04-23-pdf-report.md | 2026 K-POP IR 리포트 | inbox | paper | [...] | inbox 옆 복사 |
| 3 | inbox/2026-04-23-moodboard.md | 스프링 무드보드 | inbox | image | [...] | raw/images/로 복사 |
| 4 | ... | ... | ... | ... | ... | ... |

---

**{N}개 전부 생성?** 제외: "#3 제외". 슬러그 수정: "#1 슬러그 'XX'로".
```

### 피드백 처리

| 사용자 응답 | 처리 |
|---|---|
| 전체 승인 | 실행 Phase 진입 |
| 개별 제외 | 해당 소스 `{sources}`에서 제거, R2 재제시 |
| 슬러그 수정 | 해당 항목만 슬러그 갱신, R2 재제시 |
| 전체 중단 | 아무 파일도 건드리지 않고 종료 |

---

## 실행 Phase (Step 3~5 sequential 자동)

각 소스에 대해 순서대로:

1. **Step 3**: 원본 처리 (외부 파일 복사 또는 URL은 그대로)
2. **Step 4**: inbox 페이지 작성 (R2 확정 경로·슬러그 사용, type=inbox 고정)
3. **Step 5**: `log.md` 갱신 (개별 ingest 이벤트)

`log.md`는 **소스마다 개별 ingest 이벤트로 기록** (배치 단위 요약 엔트리는 따로 만들지 않음 — 개별 추적성 유지).

> **index.md는 갱신하지 않는다** — classify의 책임. 배치 ingest 후 `/classify` 실행 권장.

### 진행률 업데이트

```
✓ [1/5] {title1} — inbox/2026-04-23-newjeans-comeback.md
✓ [2/5] {title2} — inbox/2026-04-23-pdf-report.md
⚙ [3/5] 처리 중...
```

### 사용자 확인 없음

R1~R2에서 이미 전체 프리뷰 확인을 받았으므로, 실행 Phase는 전부 자동. 단일 모드와 달리 **소스별 확인 반복 없음**.

---

## 에러 처리 (부분 실패 허용)

**원칙: 하나 실패가 전체를 막지 않는다. 부분 성공 상태 보존.**

| 상황 | 처리 |
|---|---|
| R1에서 소스 읽기 실패 | 해당 소스 제외, 다른 소스 계속 |
| R1에서 media confidence low | 사용자 판단 대기, 나머지 진행 안 함 |
| R2에서 경로 충돌 | 해당 소스만 manual mode 안내, 나머지 계속 |
| 실행 Phase의 Step 4 실패 | 그 소스만 에러 로그, 다음 소스 계속 |
| `log.md` 갱신 실패 | 페이지는 생성됨. 수동 갱신 안내 |

---

## 최종 리포트

```
✓ 배치 ingest 완료 (성공 {M} / 전체 {N})

### 성공 (모두 inbox/ 도착)
- [[inbox/2026-04-23-newjeans-comeback]] (article)
- [[inbox/2026-04-23-pdf-report]] (paper)
- [[inbox/2026-04-23-moodboard]] (image)
...

### 실패 ({N - M}개)
- {source3}: Step 4 경로 충돌 — 수동 확인 필요
- {source5}: Step 1 fetch 실패 — URL 재확인 필요

### 다음 단계
`/classify`로 inbox 일괄 분류 권장.
```

---

## 단일 모드와의 관계

- **단일 소스**: 단일 모드 워크플로우 그대로. 배치 로직 비활성.
- **배치에서 "#3만 단일 모드로" 요청**: 그 소스 배치에서 빼고 단일 `/ingest {source}`로 재호출 안내.

---

## 제약 (v0.1)

- **Sequential 실행**. 진짜 subagent 병렬은 추후. 이유: Chrome MCP 싱글 세션, `log.md` 공유 파일 쓰기 충돌 위험.
- **컨텍스트 누적 주의**. 10개 이상 소스는 LLM 컨텍스트 부담 — 대규모 배치는 5~8개 단위로 나눠 실행 권장.
- **media confidence low 자동 해결 없음**. R1에서 사용자 판단 필수.
- **`--inbox-all` 미지원**. inbox는 ingest의 목적지라 의미 없음. 분류는 `/classify`.
