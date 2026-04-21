# Ingest 7단계 상세 워크플로우

CLAUDE.md §5.2와 1:1 대응. 각 단계의 입력·처리·출력·에러 처리.

> **이 문서는 단일 모드 기준.** 배치 모드(소스 2개 이상 또는 `--inbox-all`)의 3라운드 통합 확인 플로우는 [batch.md](batch.md) 참조. 배치 모드도 Step 4~7은 이 문서 규칙대로 소스별 sequential 실행.

---

## Step 1. 소스 읽기

### 분기

| 입력 | 도구 | 비고 |
|---|---|---|
| HTTP(S) URL, 일반 도메인 | `WebFetch` | HTML → text |
| HTTP(S) URL, Chrome 도메인 | `mcp__Claude_in_Chrome__navigate` + `get_page_text` | sources.md 참조 |
| 파일 `.md`·`.txt`·`.html` | `Read` 직접 | |
| 파일 `.pdf` | `Read` + `pages` 인자 | 10페이지 초과 시 분할 |
| 파일 이미지 | `Read` (비전) | 캡션·OCR |
| 기타 확장자 | 지원 안 함 | 변환 요청 |

### 에러

- URL 응답 실패 → 재확인 요청, 재시도 불가 시 중단
- 파일 미존재 → 즉시 에러
- Chrome MCP 미연결 → 대안 안내 (sources.md)

### 출력

원본 텍스트 + 메타(원제, URL/경로, 도메인 호스트, 소스 성격 추정).

---

## Step 2. 요약 + 핵심 논점

### 처리

- 한 줄 요약 1개
- 핵심 논점 3~5개
- 소스 성격 태깅 (기사·인터뷰·영상 스크립트·SNS 포스트·공식 발표·논문·리뷰)
- 이후 단계에서 쓸 힌트 축적 (엔티티 후보·제목 후보 등)

### B모드 확인

사용자에게 제시:

```
## 소스 요약

- 한 줄: {summary}
- 핵심 논점:
  1. {point}
  2. {point}
  ...
- 성격: {기사 | SNS | 영상 | ...}

**이 방향으로 진행해도 될까요? 수정할 관점 있으면 지시해 주세요.**
```

사용자 피드백 반영 후 Step 3.

---

## Step 3. 도메인 분류

### 처리

1. 볼트의 `_enterwiki.md`·`_sanawiki.md`·`_sanarchive.md`를 읽음
2. 각 도메인의 "다루는 범위"·"다루지 않는 것" 섹션 파싱
3. 소스 내용과 키워드·주제·성격 매칭
4. 후보 도메인 랭킹 + confidence (high/medium/low)
5. top1 제시, 걸치는 경우 대안도 함께
6. **medium 추론** — URL host·확장자·MIME으로 `article | video | image | podcast | paper` 중 판정 (classifier.md "medium 추론" 섹션). 도메인 제안과 함께 반환.

상세는 [classifier.md](classifier.md).

### B모드 확인

```
## 도메인 제안

**추천:** {domain} (confidence: {level})
- 근거: {한 줄}

{대안 있으면:}
**대안 후보:**
- {domain}: {근거}

**medium:** {medium} (confidence: {medium-level})
{medium confidence low/medium일 때만 추가:}
- 근거: {host/확장자 근거 — 예: "host:youtube.com", "ext:.pdf"}
- 대안: {alt_medium}

이대로 `{domain}` / `{medium}`으로 진행할까요?
```

> medium confidence가 high면 `**medium:** {value}` 한 줄만 출력(근거·대안 숨김). low/medium이면 근거+대안 함께 노출. 사용자 override 시 그대로 따름.

### 교차 도메인

sana/enter·sana/archive 등 걸치는 소스는 **주 도메인 하나만** 선택. 부 도메인 연결은 Step 6에서 링크로.

---

## Step 4. 원본 처리

### 분기

| 입력 타입 | 처리 |
|---|---|
| URL (일반·Chrome 양쪽) | 이동 없음. Step 5에서 `source:` 필드에 URL 기록 |
| `01. inbox/` 내 파일 | `{domain}/raw/`로 **자동 이동** (확인 없음) |
| 볼트 밖 파일 | **이동 금지**. "볼트로 복사할까요?" 확인 후 복사 (외부 원본 보존) |

### raw/ 경로

| 도메인 | raw 경로 |
|---|---|
| enter | `02. enterwiki/raw/` |
| sana | `03. sanawiki/raw/` |
| archive | `05. sanarchive/raw/briefs/` 또는 `/deliverables/` (파일 성격 따라) |
| entity | raw 없음 — skip |
| project | `06. studio-sana/{project}/raw/` |

raw 하위 폴더가 미존재하면 생성. 파일명 충돌 시 ` (2)` 등 붙여 회피.

---

## Step 5. 소스 페이지 작성

### 경로 결정

```
{domain}/wiki/{category}/{제목}.md
```

- `category`: `_{domain}.md`의 서브구조에서 선택 (예: 음악·패션·케이스)
- `제목`: LLM 정제. 규칙:
  - 원제에서 볼트 스타일로 압축 (한글 명사구)
  - 날짜 prefix 금지 (CLAUDE.md §3.3)
  - Obsidian 파일명 안전 문자만 (`/`·`\` 금지)

### 템플릿

볼트의 `CLAUDE.md §4.2` 템플릿 사용:

```yaml
---
title: {정제된 제목}
type: source
domain: {domain-enum}
medium: {medium-enum}
tags: [{중간입자도 2~4개}]
source: {URL 또는 비움}
updated: {YYYY-MM-DD}
---
```

> `medium`은 Step 3의 medium 추론 결과(`article | video | image | podcast | paper` 중 하나). 상세 규칙은 [classifier.md](classifier.md)의 "medium 추론" 섹션.

섹션:
- `## 한 줄 요약`
- `## 핵심 논점` — Step 2 논점
- `## 인용할 만한 구절` — 원본 발췌
- `## 띠니의 관찰` — 2~4문단. 다른 도메인과의 연결·후속 액션 힌트 포함
- `## 관련` — 엔티티 링크 + 외부 맥락

소스 유형에 따라 추가 섹션 자유 (예: "언급된 아티스트", "주요 인용").

### B모드 확인

**신규 파일 생성이므로 사용자 확인 필수:**

```
## 작성 예정

- 경로: {path}
- 제목: {title}
- 태그: [{tags}]
- 섹션: 한 줄 요약, 핵심 논점({N}), 인용({N}), 띠니의 관찰, 관련

생성해도 될까요?
```

confirm 후 `Write`.

---

## Step 6. 관련 페이지 업데이트

### 엔티티 링크 삽입

1. 소스 내용에서 인명·브랜드·작품 후보 추출
2. 각 후보에 대해 `04. entities/**/{이름}.md` 존재 확인
3. **존재하면** 새 소스 페이지의 `## 관련` 섹션에 `[[경로|이름]]` 추가
4. **존재하지 않으면** 스킵 + 리포트 말미에 엔티티 후보 힌트로 기록 (v1은 생성 안 함)

### 기존 도메인 페이지 업데이트

같은 도메인에 관련 주제 페이지가 있으면 상호 링크 후보 발견. 하지만:

- **v1은 자동 적용 금지.** 본문 재작성은 위험.
- 링크 1줄 추가는 자동 OK, 본문 수정은 "제안"으로만 리포트 기재.
- 사용자가 별도로 요청하면 그때 진행.

---

## Step 7. index.md + log.md 갱신

### index.md

해당 도메인 섹션의 카테고리에 한 줄 추가:

```markdown
- [[{path}|{title}]] — {한 줄 요약 압축}
```

카테고리 미존재 시: 가장 근접한 카테고리에 배치, 리포트에 "새 카테고리 고려" 힌트 기록.

### log.md

append-only, 앞쪽 `---` 뒤에 추가:

```markdown
## [{YYYY-MM-DD}] ingest | {title}

- 도메인: {domain}
- 추가된 페이지: [[{path}]]
- 갱신된 페이지: [[index]] + {기타}
- 핵심 인사이트: {2~3문장}
```

### 자동

두 파일 모두 확인 없이 `Edit`. CLAUDE.md §5.2 규칙.

---

## 완료 출력

```
✓ ingest 완료

- 도메인: {domain}
- 신규 페이지: [[{path}]]
- 갱신된 페이지: [[index]], [[log]]
- 원본 처리: {이동 경로 | URL 참조만}
- 엔티티 후보 (참고): {name1}, {name2} → /promote 검토 권장
- 관련 페이지 제안 (검토): [[page1]], [[page2]]
```

---

## 에러 복구 (파괴적 복구 금지)

| 실패 단계 | 상태 | 처리 |
|---|---|---|
| Step 1~3 | 파일 변경 없음 | 에러 출력만 |
| Step 4 | inbox 파일 이동 실패 | 파일은 inbox에 남음. 원인 보고 |
| Step 5 | raw 이동은 됐을 수 있음 | 되돌리지 말고 에러만 보고. 재실행 시 사용자가 이동된 파일을 직접 지정 |
| Step 6~7 | 소스 페이지는 생성됨 | index/log 수동 갱신 안내 |

원칙: **부분 성공 상태를 그대로 두고, 다음 재호출이 이어서 처리**할 수 있게 한다.
