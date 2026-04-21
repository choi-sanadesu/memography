# 도메인 분류기

소스 내용을 읽고 어느 도메인에 속할지 판단. 볼트의 `_{domain}.md`를 기준 문서로 삼는다.

---

## 입력

- 소스 텍스트 (Step 1 결과)
- 메타: URL 호스트, 원제, 소스 성격 (기사·SNS·영상 등)
- Step 2의 요약·핵심 논점

## 출력

- 추천 도메인 1개 + confidence (high/medium/low)
- 대안 도메인 0~2개 + 간단 근거

---

## 분류 절차

### 1. 볼트의 도메인 정의 파일 읽기

- `02. enterwiki/_enterwiki.md`
- `03. sanawiki/_sanawiki.md`
- `05. sanarchive/_sanarchive.md`

제외:
- `entity` — 분류 대상 아님 (다른 도메인 소스에서 파생되는 허브)
- `project` — 프로젝트 세션 컨텍스트에서만 ingest (일반 분류 대상 아님)
- `meta` — 운영 문서라 소스 ingest 대상 아님

### 2. 각 도메인 파일에서 기준 추출

- "다루는 범위" / "주 대상" / "확장 대상"
- "다루지 않는 것" (명시적 제외)
- "관점" (도메인 고유 시선)
- 서브구조 (카테고리 후보)

### 3. 소스 매칭

| 매치 유형 | confidence 기여 |
|---|---|
| "주 대상"과 일치 | high |
| "확장 대상"과 일치 | medium |
| "다루지 않는 것"에 해당 | 즉시 제외 |
| 관점 부합 (디자인·업계·개인 취향) | medium 가중 |

### 4. 랭킹 + 대안

- top1 confidence high → 단독 제시
- top1 medium, top2 medium → 두 후보 제시 + 사용자 선택
- top1 low → 세 후보까지 + 명시적 "불확실" 라벨

---

## 교차 도메인 처리

여러 도메인이 해당하는 소스는 현실적으로 흔함. 처리:

- **주 도메인 하나만** 선택해 페이지 생성
- 부 도메인은 Step 6에서 링크로 연결
- 일반적 우선순위 (tiebreaker):

| 우선 | 시나리오 |
|---|---|
| archive | 소스가 사용자의 디자인 작업·포트폴리오 맥락과 직접 연결 |
| enter | 엔터 업계 기획·비즈니스·구조 분석 관점 |
| sana | 개인 취향·감상·왜 좋아하는지 |

사용자가 override하면 그대로 따름.

---

## 판단 힌트 (빠른 참조)

| 소스 성격 | 전형적 도메인 | 메모 |
|---|---|---|
| 아이돌 앨범·컴백·기획 뉴스 | enter | |
| 엔터사 IR·업계 리포트 | enter | |
| 아이돌 그룹 비주얼 디렉션 분석 | enter (비주얼레퍼런스) | 디자이너 관점 포함 |
| 좋아하는 곡·아티스트 소개 | sana (음악) | |
| 패션·공간·음식 추천 | sana | |
| 디자이너 인터뷰 (업계 맥락) | enter | |
| 디자이너 인터뷰 (개인 취향·영감) | sana (사람) | |
| 디자인 케이스 스터디 (사나 본인 프로젝트) | archive (케이스) | |
| 디자인 툴·기법·레퍼런스 | archive (레퍼런스) | |
| 디자인 업계 전반 단상 | archive (인사이트) | |
| 드라마·영화 가십 | 제외 (enter "다루지 않는 것") | |

---

## B모드 확인 포맷

```
## 도메인 제안

**추천:** {domain} (confidence: {level})
- 근거: {한 줄}

{대안이 있으면:}
**대안 후보:**
- {domain2}: {근거}
- {domain3}: {근거}

이대로 `{domain}`으로 진행할까요? 다른 도메인을 원하면 말해주세요.
```

---

## 주의

- 분류 확신이 없을 때 top1만 밀어붙이지 말고 대안 제시.
- `_{domain}.md`의 "다루지 않는 것"을 먼저 확인해 분명히 틀린 후보를 먼저 제거.
- 사용자가 override한 선택은 현재 세션 내에서만 유효. 패턴 학습은 별도 메모리 책임.

---

## media 추론

페이지에 기록할 `media` 값을 URL host·파일 확장자·MIME으로 자동 판정. type 제약 없음.

enum (예시): `article | video | image | podcast | paper`. `media: null` 허용(분류 보류). 실제 enum SOT는 볼트 `CLAUDE.md §3.1`. **스킬은 enum을 하드코딩하지 않고 이 예시를 참조만 한다** (lint/SKILL.md의 동적 읽기 원칙과 동일).

### 적용 순서 (먼저 매칭되는 규칙이 이김)

1. **host-first**: URL 입력이면 호스트 규칙을 확장자보다 먼저 적용.
2. **extension/MIME**: 파일 입력이거나 URL인데 host 규칙 미매칭일 때.
3. **fallback**: URL이면 `article`, 파일이면 확장자에 따라 판정 실패 시 `article`.

### 규칙표

| 순위 | 조건                                                                          | media     | confidence |
| ---- | ----------------------------------------------------------------------------- | --------- | ---------- |
| 1    | host ∈ {`*.youtube.com`, `youtu.be`, `*.vimeo.com`}                           | `video`   | high       |
| 2    | host ∈ {`open.spotify.com/episode·show`, `podcasts.apple.com`, `apple.co/podcast*`} | `podcast` | high       |
| 3    | host ∈ {`arxiv.org`, `*.arxiv.org`} 또는 host에 `doi.org` 포함                 | `paper`   | high       |
| 4    | 확장자 `.pdf` (academic host 힌트 없음)                                       | `paper`   | medium     |
| 5    | 확장자 ∈ {`.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`, `.heic`, `.avif`}          | `image`   | high       |
| 6    | 확장자 ∈ {`.mp3`, `.m4a`} (podcast host 아닌 경우)                            | `podcast` | medium     |
| 7    | 그 외 URL (HTTP/HTTPS 응답 text/html)                                         | `article` | high       |
| 8    | 그 외 파일 (`.md`·`.txt`·`.html`)                                             | `article` | medium     |

### Edge cases

- **youtube URL에 `.pdf` 포함**: host 규칙(1번)이 확장자보다 먼저 → `video` high.
- **spotify 도메인인데 music track (episode/show 경로 아님)**: 2번 미매칭 → 7번 fallback(`article`) confidence medium → B모드 확인 발동.
- **medium.com의 이미지 파일 URL**: host 규칙 없음 → 5번 확장자 규칙 → `image` high.
- **arxiv 추상 페이지 (HTML, PDF 아님)**: 3번 host 규칙 매칭 → `paper` high. 본문은 abstract이지만 미디어 유형 축으로는 paper.
- **`.pdf`가 youtube 링크 안에 쿼리로 섞인 경우**: host 규칙 우선.

### B모드 확인 트리거

- confidence **high**: Step 3 확인 블록에 `**media:** {value}` 한 줄만 표시. 사용자 무입력 시 통과.
- confidence **medium/low**: Step 3 확인 블록에 `근거` + `대안` 함께 노출하여 B모드 확정.
- 사용자가 분류 보류 원하면 `media: null`로 기록 (fail/warn 없음).

### Step 3 반환값

도메인 제안과 함께 media 추론 결과를 반환:

```
{
  domain: {domain},
  domain_confidence: high | medium | low,
  media: {media},
  media_confidence: high | medium | low,
  media_basis: "host:youtube.com" | "ext:.pdf" | "fallback:article"
}
```

Step 5 템플릿이 이 media 값을 frontmatter에 그대로 주입.

### 볼트 §3.1에 media enum 미정의 시

- 스킬은 위 규칙으로 media를 그대로 기록한다 (enum 부재여도 동작).
- 배치 리포트 말미에 "볼트 `CLAUDE.md §3.1`에 media enum 미정의 — SOT 업데이트 권장" 힌트를 한 줄 기록.
