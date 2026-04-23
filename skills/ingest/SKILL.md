---
name: ingest
description: Memography 볼트에 새 소스를 inbox 단계까지 편입하는 표준 워크플로우. URL(HTTP/HTTPS) 또는 파일 경로를 입력받아 `inbox/{YYYY-MM-DD}-{slug}.md`에 페이지를 작성하고 log.md에 ingest 이벤트를 기록한다. 분류는 별도 /classify 스킬이 담당. 단일 또는 여러 소스 배치 처리 지원. google.com·youtube.com·instagram.com·twitter.com/x.com 도메인은 Claude in Chrome MCP 경유. media 필드(`article | video | image | podcast | paper`)는 URL host·확장자·MIME으로 자동 추론. type은 항상 `inbox` 마커 (최종 type은 classify Step 3에서 확정). B모드(신규 페이지 생성은 확인, log.md 기록은 자동) 기준. 배치 모드에서는 Step 1~2를 라운드 통합 확인으로 압축. "ingest", "인제스트", "볼트에 추가", "소스 추가", "/ingest" 같은 표현이 나올 때 트리거.
---

# memography: ingest

볼트에 새 소스를 inbox까지 편입. 도메인 분류·최종 폴더 결정은 하지 않는다 — `/classify`의 책임.

## 언제 쓰는가

- URL(기사·SNS 포스트·영상 링크 등)이나 파일(`.md`·`.pdf`·`.txt`·이미지 등)을 볼트에 편입할 때
- 외부 자료를 빠르게 inbox에 모아두고 분류는 나중에 일괄 처리하고 싶을 때

## 입력

### 단일 모드

| 타입 | 예시 |
|---|---|
| URL | `https://www.instagram.com/reel/...` |
| 파일 경로 (텍스트·PDF) | `~/Downloads/article.pdf`, 절대경로 |
| 이미지 파일 | `~/Downloads/screenshot.png`, `.jpg`·`.jpeg`·`.webp`·`.gif` |

### 배치 모드

| 호출 | 동작 |
|---|---|
| `/ingest {url1} {url2} {file1}` | 리스트 지정 배치 |

자유 텍스트 붙여넣기는 v0.1에서 미지원 (출처 모호성 회피).

배치 모드의 라운드 통합 확인 플로우는 [references/batch.md](references/batch.md) 참조.

## 동작 원칙

- 대상 볼트의 `CLAUDE.md §1`(폴더 구조)·`§2`(스키마)·`§5.1`(ingest 단계)을 **읽어서** 실행 기준으로 삼는다. 하드코딩 금지.
- **type은 항상 `inbox`**: 수집 시점엔 목적지(`source`/`note`/...) 결정 안 함. classify Step 3에서 최종 type 확정.
- **media 자동 추론**: URL host·확장자·MIME으로 판정 (상세: [references/media.md](references/media.md)). `media: null` 허용(분류 보류 시).
- **B모드 원칙 준수**: 신규 페이지 생성은 사용자 확인, log.md 기록은 자동.
- **파괴적 복구 금지**: 중간 단계 실패 시 부분 성공 상태를 보존.
- **엔티티 안 건드림**: ingest는 inbox/까지만. entities/ 갱신·신규 엔티티 페이지 생성은 classify의 책임.

## 5단계 워크플로우

[references/workflow.md](references/workflow.md)에 상세 분기·에러 처리 기술. 개요:

| # | 단계 | 처리 | B모드 확인 |
|---|---|---|---|
| 1 | 소스 읽기 | URL→fetch/Chrome, 파일→Read, 이미지→Read(비전) + 메타 추출(`sips`/`stat`) | — |
| 2 | 요약 + media 추론 | 80~120자 summary + 핵심 논점 + media 자동 판정 ([media.md](references/media.md)) | media confidence low/medium → ✋ |
| 3 | 원본 처리 | URL→이동 없음 / 외부 파일→`raw/images/`(재사용) 또는 inbox 옆 / inbox 내 파일→그대로 | 외부 파일만 ✋ |
| 4 | inbox 페이지 작성 | `inbox/{YYYY-MM-DD}-{slug}.md` (플랫). frontmatter: `title? / type: inbox / tags / updated / summary` (+ `source:` if URL, + `media:` if 추론). 목적지 결정 안 함 | ✋ **신규 페이지 생성 확인** |
| 5 | log.md 기록 | `## [date] ingest \| {제목}` append | 자동 |

## Chrome MCP 분기

URL 호스트가 다음 패턴 중 하나와 매칭되면 `mcp__Claude_in_Chrome__*` 경유:

- `*.google.com`
- `*.youtube.com`
- `*.instagram.com`
- `*.twitter.com` · `*.x.com`

Chrome MCP 미연결 환경이면 에러 출력 + 대안 안내 (sources.md 참조).

[references/sources.md](references/sources.md)에 URL/파일 로딩 상세.

## 이미지 분기

입력 파일 확장자가 이미지(`.png`·`.jpg`·`.jpeg`·`.webp`·`.gif`)면 Step 1에서 메타 추출(`sips`·`stat`) → `Read`로 비전 캡션 → 텍스트 인지 시 OCR 자동 전사. Step 4는 `type: inbox` + `media: image` + 본문 첫 줄에 `![[raw/images/...]]` embed. 재사용 가치 있으면 `raw/images/`에 저장, 단발이면 inbox 옆. 세부 절차·금지 사항은 [references/images.md](references/images.md).

## 주의

- **엔티티 페이지 생성은 이 스킬이 하지 않는다.** classify Step 3 B모드에서 사나 승인 후 생성.
- **index.md 갱신도 이 스킬이 하지 않는다.** classify Step 5의 책임.
- **파일명 규칙**: `{YYYY-MM-DD}-{slug}.md`. 한글 허용, 특수문자 회피.
- **type은 항상 `inbox`**: 수집 시점에 source/note 판단 금지. classify가 최종 type 결정.

## 완료 출력

```
✓ ingest 완료

- 신규 페이지: [[inbox/{YYYY-MM-DD}-{slug}]]
- type: inbox (classify에서 확정 예정)
- media: {value} ({confidence})
- 원본 처리: {raw/images/로 이동 | inbox 옆 | URL 참조만}
- 갱신된 페이지: [[log]]
- 다음 단계: `/classify`로 inbox 일괄 정리 가능
```

## 한계 (v0.1)

- **Sequential 실행** — 진짜 subagent 병렬은 추후. Chrome MCP 싱글 세션 + log.md 공유 쓰기 충돌 방지.
- **`--inbox-all` 미지원** — inbox는 ingest의 목적지(쓰는 곳), 분류는 `/classify`. 옛 alexandria-keeper의 "inbox 일괄 ingest" 의미는 사라짐.
- 영상 콘텐츠(YouTube·릴스)의 **영상 내용 자체 이해**는 out of scope. 설명·자막·댓글까지만.
- 이미지는 **GIF 1프레임만** 분석. HEIC/HEIF는 `sips`로 JPG 변환 후 처리. EXIF GPS·카메라 기종 같은 확장 메타는 추후 검토.
- `--auto` 플래그(C모드 전환용)는 추후 도입. v0.1은 B모드 고정.
- **media 필드 누락 시 `/lint` warning**. `media: null`은 유효 (fail/warn 없음).
