# media 추론 (ingest 자동 판정)

페이지에 기록할 `media` 값을 URL host·파일 확장자·MIME으로 자동 판정. type 제약 없음 — `media`는 모든 type에 선택 적용.

enum (예시): `article | video | image | podcast | paper`. `media: null` 허용(분류 보류). 실제 enum SOT는 볼트 `CLAUDE.md §2`. **스킬은 enum을 하드코딩하지 않고 이 예시를 참조만 한다** (lint/SKILL.md의 동적 읽기 원칙과 동일).

---

## 적용 순서 (먼저 매칭되는 규칙이 이김)

1. **host-first**: URL 입력이면 호스트 규칙을 확장자보다 먼저 적용.
2. **extension/MIME**: 파일 입력이거나 URL인데 host 규칙 미매칭일 때.
3. **fallback**: URL이면 `article`, 파일이면 확장자에 따라 판정 실패 시 `article`.

---

## 규칙표

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

---

## Edge cases

- **YouTube URL에 `.pdf` 포함**: host 규칙(1번)이 확장자보다 먼저 → `video` high.
- **Spotify 도메인인데 music track (episode/show 경로 아님)**: 2번 미매칭 → 7번 fallback(`article`) confidence medium → B모드 확인 발동.
- **medium.com의 이미지 파일 URL**: host 규칙 없음 → 5번 확장자 규칙 → `image` high.
- **arxiv 추상 페이지 (HTML, PDF 아님)**: 3번 host 규칙 매칭 → `paper` high. 본문은 abstract이지만 미디어 유형 축으로는 paper.
- **`.pdf`가 YouTube 링크 안에 쿼리로 섞인 경우**: host 규칙 우선.

---

## B모드 확인 트리거

- confidence **high**: Step 2 확인 블록에 `**media:** {value}` 한 줄만 표시. 사용자 무입력 시 통과.
- confidence **medium/low**: Step 2 확인 블록에 `근거` + `대안` 함께 노출하여 B모드 확정.
- 사용자가 분류 보류 원하면 `media: null`로 기록 (fail/warn 없음).

---

## Step 2 반환값

요약과 함께 media 추론 결과를 반환:

```
{
  summary: "{80~120자 한 줄}",
  points: ["...", "..."],
  media: {media},
  media_confidence: high | medium | low,
  media_basis: "host:youtube.com" | "ext:.pdf" | "fallback:article"
}
```

Step 4 템플릿이 이 media 값을 frontmatter에 그대로 주입.

---

## 볼트 §2에 media enum 미정의 시

- 스킬은 위 규칙으로 media를 그대로 기록한다 (enum 부재여도 동작).
- 배치 리포트 말미에 "볼트 `CLAUDE.md §2`에 media enum 미정의 — SOT 업데이트 권장" 힌트를 한 줄 기록.
