# Ingest 5단계 상세 워크플로우

볼트 `CLAUDE.md §5.1`과 1:1 대응. 각 단계의 입력·처리·출력·에러 처리.

> **이 문서는 단일 모드 기준.** 배치 모드(소스 2개 이상)의 라운드 통합 확인 플로우는 [batch.md](batch.md) 참조. 배치 모드도 Step 3~5는 이 문서 규칙대로 소스별 sequential 실행.

---

## Step 1. 소스 읽기

### 분기

| 입력 | 도구 | 비고 |
|---|---|---|
| HTTP(S) URL, 일반 도메인 | `WebFetch` | HTML → text |
| HTTP(S) URL, Chrome 도메인 | `mcp__Claude_in_Chrome__navigate` + `get_page_text` | sources.md 참조 |
| 파일 `.md`·`.txt`·`.html` | `Read` 직접 | |
| 파일 `.pdf` | `Read` + `pages` 인자 | 10페이지 초과 시 분할 |
| 파일 이미지 | `Read` (비전) + `sips`/`stat` | 메타·캡션·OCR — 상세 [images.md](images.md) |
| 기타 확장자 | 지원 안 함 | 변환 요청 |

### 에러

- URL 응답 실패 → 재확인 요청, 재시도 불가 시 중단
- 파일 미존재 → 즉시 에러
- Chrome MCP 미연결 → 대안 안내 (sources.md)

### 이미지 특이사항

- 메타 추출(`sips` 실패·없는 환경)은 폴백하고 실패해도 Step 2~5로 진행. 빈 값은 frontmatter에서 비워 둔다.
- **볼트 밖 이미지는 복사 전 메타 추출 금지.** Step 3에서 복사에 동의한 뒤 볼트 내 최종 경로에서 추출.
- HEIC/HEIF는 `sips -s format jpeg --out {tmp.jpg}`로 선변환 후 처리. 변환 실패 시 중단.
- GIF는 첫 프레임만 분석 — 여러 프레임이 의미 있으면 본문에 별도 명시.

### 출력

원본 텍스트(또는 이미지 캡션·OCR) + 메타(원제, URL/경로, 도메인 호스트, 소스 성격 추정, 이미지일 때는 `format`·`resolution`·`captured`).

---

## Step 2. 요약 + media 추론

### 처리

- 한 줄 summary 1개 (80~120자, 볼트 §2 규칙)
- 핵심 논점 3~5개
- 소스 성격 태깅 (기사·인터뷰·영상 스크립트·SNS 포스트·공식 발표·논문·리뷰·이미지)
- **media 자동 판정** — URL host·확장자·MIME으로 `article | video | image | podcast | paper` 중 판정 ([media.md](media.md))
- 이후 단계에서 쓸 힌트 축적 (제목 후보 등)

### B모드 확인

사용자에게 제시:

```
## 소스 요약

- summary: {80~120자}
- 핵심 논점:
  1. {point}
  2. {point}
  ...
- 성격: {기사 | SNS | 영상 | ...}

**media:** {media} (confidence: {level})
{media confidence low/medium일 때만 추가:}
- 근거: {host/확장자 — 예: "host:youtube.com", "ext:.pdf"}
- 대안: {alt_media}

이 방향으로 진행해도 될까요? 수정할 부분 있으면 지시해 주세요. (분류 보류 원하면 "media: null"로 남길 수 있습니다)
```

> media confidence가 high면 `**media:** {value}` 한 줄만 출력(근거·대안 숨김). low/medium이면 근거+대안 함께 노출. 사용자 override 시 그대로 따름. 사용자가 null 요청 시 `media: null`로 기록.

사용자 피드백 반영 후 Step 3.

---

## Step 3. 원본 처리

### 분기

| 입력 타입 | 처리 |
|---|---|
| URL (일반·Chrome 양쪽) | 이동 없음. Step 4에서 `source:` 필드에 URL 기록 |
| `inbox/` 내 파일 | 그대로 둠 (이미 inbox에 있음) |
| 볼트 밖 파일, 재사용 가치 있음 (이미지·반복 참조) | "`raw/images/`로 복사할까요?" ✋ 확인 후 복사 |
| 볼트 밖 파일, 단발 | "inbox 옆에 복사할까요?" ✋ 확인 후 복사 |

### 원칙

- **원본 보존**: 볼트 밖 원본은 이동 금지. 복사만.
- **`raw/images/`**: 볼트 §1·§7 — 재사용 이미지 영구 보관. 두 문서 이상에서 쓸 가능성 있으면 여기.
- 파일명 충돌 시 ` (2)` 등 붙여 회피.

---

## Step 4. inbox 페이지 작성

### 경로

```
inbox/{YYYY-MM-DD}-{slug}.md
```

- `{YYYY-MM-DD}`: 오늘 날짜 (수집 시점 기준)
- `{slug}`: LLM 정제. 규칙:
  - 원제에서 볼트 스타일로 압축 (한글 명사구 또는 영문 공식 표기)
  - Obsidian 파일명 안전 문자만 (`/`·`\` 금지)
  - 30자 이내 권장

### 템플릿 (기본)

```yaml
---
title: {정제된 제목}
type: inbox
tags: [{중간입자도 3~5개, 볼트 meta/tags.md 카탈로그 기준}]
updated: {YYYY-MM-DD}
summary: {Step 2의 80~120자}
source: {URL 또는 비움 — URL 입력일 때만}
media: {Step 2의 media 추론값 또는 null}
---
```

> `type: inbox` 고정. 최종 type(`source` | `note` | `entity` | `case` | `reference`)은 classify Step 3에서 확정.
> `title`은 사람 가독성 제목 — 볼트 §2 선택 필드. lint는 검증 안 하지만 ingest는 작성 (볼트 관행).

섹션:
- `## 한 줄 요약`
- `## 핵심 논점` — Step 2 논점
- `## 인용할 만한 구절` — 원본 발췌
- `## 띠니의 관찰` — 2~4문단. 다른 페이지와의 연결·후속 액션 힌트 포함

소스 유형에 따라 추가 섹션 자유 (예: "언급된 아티스트", "주요 인용").

### 이미지 입력 분기

입력이 이미지면 `media: image` 자동 + 본문 첫 줄에 embed:

```yaml
---
title: {정제된 제목}
type: inbox
tags: [{3~5개}]
updated: {YYYY-MM-DD}
summary: {Step 2 캡션 기반 80~120자}
source: {원 URL 또는 비움}
media: image
format: {png | jpg | webp | gif | ...}
resolution: {W}x{H}
captured: {YYYY-MM-DD}
---

![[raw/images/{file}]]   # 또는 inbox 옆 경로

## 캡션
{비전 기반 1~2문장}

## 추출 텍스트 (OCR)
{텍스트 인지 시에만. 원문 그대로}

## 띠니의 관찰
{일반 템플릿과 동일}
```

세부 가이드(캡션 톤, OCR 판단 기준, PII 마스킹)는 [images.md](images.md).

### B모드 확인

**신규 파일 생성이므로 사용자 확인 필수:**

```
## 작성 예정

- 경로: inbox/{YYYY-MM-DD}-{slug}.md
- 제목: {title}
- type: inbox (classify에서 확정)
- 태그: [{tags}]
- summary: {80~120자}
- 섹션: 한 줄 요약, 핵심 논점({N}), 인용({N}), 띠니의 관찰

생성해도 될까요?
```

confirm 후 `Write`.

---

## Step 5. log.md 기록

append-only, 파일 끝에 추가:

```markdown
## [{YYYY-MM-DD}] ingest | {title}

- 추가된 페이지: [[inbox/{YYYY-MM-DD}-{slug}]]
- type: inbox (classify 대기)
- media: {media}
- 원본 처리: {raw/images/로 복사 | inbox 옆 복사 | URL 참조만}
- 핵심 인사이트: {1~2문장}
```

### 자동

확인 없이 `Edit`. 볼트 §5.1·§6 규칙.

> **index.md는 갱신하지 않는다.** index.md 갱신은 classify Step 5의 책임.

---

## 완료 출력

```
✓ ingest 완료

- 신규 페이지: [[inbox/{YYYY-MM-DD}-{slug}]]
- type: inbox (classify에서 확정 예정)
- media: {value} ({confidence})
- 원본 처리: {경로}
- 갱신된 페이지: [[log]]
- 다음 단계: `/classify`로 inbox 일괄 정리 가능
```

---

## 에러 복구 (파괴적 복구 금지)

| 실패 단계 | 상태 | 처리 |
|---|---|---|
| Step 1~2 | 파일 변경 없음 | 에러 출력만 |
| Step 3 | 외부 파일 복사 실패 | 원본은 그대로. 원인 보고 |
| Step 4 | inbox 페이지 생성 실패 | Step 3 복사본 정리 안내 |
| Step 5 | inbox 페이지 생성됨, log 갱신 실패 | log.md 수동 갱신 안내 |

원칙: **부분 성공 상태를 그대로 두고, 다음 재호출이 이어서 처리**할 수 있게 한다.
