---
name: ingest
description: Alexandria(또는 memorial-vault) Obsidian 볼트에 새 소스를 추가하는 표준 워크플로우. URL(HTTP/HTTPS) 또는 파일 경로를 입력받아 소스 요약 페이지를 작성하고 index.md·log.md를 동시 갱신한다. 단일 소스 또는 여러 소스 배치 처리 지원 (`--inbox-all`로 인박스 전체 처리도 가능). google.com·youtube.com·instagram.com·twitter.com/x.com 도메인은 Claude in Chrome MCP 경유. B모드(신규 페이지 생성은 확인, 기존 페이지 수정은 자동) 기준. 배치 모드에서는 Step 1~3을 3라운드 통합 확인으로 압축. "ingest", "인제스트", "볼트에 추가", "소스 추가", "/ingest" 같은 표현이 나올 때 트리거.
---

# alexandria-keeper: ingest

볼트에 새 소스를 표준 워크플로우로 편입. `CLAUDE.md §5.2`의 7단계를 순서대로 실행하며 B모드 확인 포인트를 지킨다.

## 언제 쓰는가

- URL(기사·SNS 포스트·영상 링크 등)이나 파일(`.md`·`.pdf`·`.txt`·이미지 등)을 볼트에 편입할 때
- `01. inbox/`에 쌓인 파일을 개별 처리할 때 (배치는 v2)

## 입력

### 단일 모드

| 타입 | 예시 |
|---|---|
| URL | `https://www.instagram.com/reel/...` |
| 파일 경로 (텍스트·PDF) | `01. inbox/article.pdf`, 절대경로 |
| 이미지 파일 | `01. inbox/screenshot.png`, `.jpg`·`.jpeg`·`.webp`·`.gif` |

### 배치 모드 (v1.1)

| 호출 | 동작 |
|---|---|
| `/ingest {url1} {url2} {file1}` | 리스트 지정 배치 |
| `/ingest --inbox-all` | `01. inbox/` 전체 처리 |

자유 텍스트 붙여넣기는 v1에서 미지원 (출처 모호성 회피).

배치 모드의 3라운드 통합 확인 플로우는 [references/batch.md](references/batch.md) 참조.

## 동작 원칙

- 대상 볼트의 `CLAUDE.md §3.1`(스키마)·`§4.2`(템플릿)·`§5.2`(단계)를 **읽어서** 실행 기준으로 삼는다. 하드코딩 금지.
- **B모드 원칙 준수**: 신규 페이지 생성은 사용자 확인, 기존 페이지 수정은 바로 진행 + 요약 보고.
- **파괴적 복구 금지**: 중간 단계 실패 시 부분 성공 상태를 보존.

## 7단계 워크플로우

[references/workflow.md](references/workflow.md)에 상세 분기·에러 처리 기술. 개요:

| # | 단계 | 처리 | B모드 확인 |
|---|---|---|---|
| 1 | 소스 읽기 | URL→fetch/Chrome, 파일→Read, 이미지→Read(비전) + 메타 추출(`sips`/`stat`) | — |
| 2 | 요약·핵심 논점 | LLM 추출 (1줄 요약 + 3~5 논점), 이미지는 캡션·OCR로 대체 | ✋ 방향 확인 |
| 3 | 도메인 분류 | `_{domain}.md` 읽어 매칭 | ✋ 도메인 확정 |
| 4 | 원본 처리 | 파일→`{domain}/raw/` 자동 이동, URL→`source:`만 | 자동 |
| 5 | 소스 페이지 작성 | §4.2 템플릿, 제목 LLM 정제. 이미지는 `type: image` 분기 | ✋ **신규 생성 확인** |
| 6 | 관련 페이지 업데이트 | 기존 엔티티에 링크, 엔티티 신규 생성은 금지 | 링크=자동, 본문수정=제안만 |
| 7 | `index.md` + `log.md` 갱신 | 자동 추가 | 자동 |

## Chrome MCP 분기

URL 호스트가 다음 패턴 중 하나와 매칭되면 `mcp__Claude_in_Chrome__*` 경유:

- `*.google.com`
- `*.youtube.com`
- `*.instagram.com`
- `*.twitter.com` · `*.x.com`

Chrome MCP 미연결 환경이면 에러 출력 + 대안 안내 (sources.md 참조).

[references/sources.md](references/sources.md)에 URL/파일 로딩 상세.

## 이미지 분기

입력 파일 확장자가 이미지(`.png`·`.jpg`·`.jpeg`·`.webp`·`.gif`)면 Step 1에서 메타 추출(`sips`·`stat`) → `Read`로 비전 캡션 → 텍스트 인지 시 OCR 자동 전사. Step 5는 `type: image` 템플릿으로 분기해 frontmatter에 `format`·`resolution`·`captured`를 채우고, 본문 첫 줄에 `![[...]]` embed를 둔다. 세부 절차·금지 사항은 [references/images.md](references/images.md).

## 분류기

대상 볼트의 `_{domain}.md` 파일들을 읽어 "다루는 범위"·"다루지 않는 것" 기준으로 매칭.

[references/classifier.md](references/classifier.md)에 분류 규칙·판단 힌트.

## 주의

- **엔티티 페이지 생성은 이 스킬이 하지 않는다.** 엔티티 승격은 `/promote` 스킬의 책임(v2). 이 스킬은 "이미 있는 엔티티"에만 링크하고, 후보 이름은 리포트 말미에 힌트로 남긴다.
- **파일명은 LLM 정제 제목 사용.** `CLAUDE.md §3.3` 규칙(한글·날짜 prefix 없음·Unicode 허용)을 따른다.
- **관련 페이지 본문 재작성 금지.** 링크 1줄 추가만 자동. 본문 수정은 "제안"으로만 리포트에 표기.
- **index.md 카테고리 미존재** 시 새 카테고리 섹션을 맘대로 만들지 말고, 가장 근접한 카테고리에 배치 후 사용자 이의 제기 기회 제공.

## 완료 출력

```
✓ ingest 완료

- 도메인: {domain}
- 신규 페이지: [[{path}]]
- 갱신된 페이지: [[index]], [[log]]
- 원본 처리: {raw/로 이동 | URL 참조만}
- 엔티티 후보 (참고): {name1}, {name2} → /promote 검토 권장
- 관련 페이지 제안 (검토): [[page1]], [[page2]]
```

## 한계 (v1.2)

- **Sequential 실행** — 진짜 subagent 병렬은 미구현. Chrome MCP 싱글 세션 + `index.md`·`log.md` 공유 쓰기 충돌 방지.
- **배치 확장자 필터 없음** — `--inbox-all`은 전체 파일 대상. 패턴·확장자 필터는 후속.
- 교차 도메인 소스는 1개 도메인만 선택. 부 도메인은 Step 6 링크로 연결.
- 영상 콘텐츠(YouTube·릴스)의 **영상 내용 자체 이해**는 out of scope. 설명·자막·댓글까지만.
- 이미지는 **GIF 1프레임만** 분석. HEIC/HEIF는 `sips`로 JPG 변환 후 처리. EXIF GPS·카메라 기종 같은 확장 메타는 v1.3+ 검토.
- `--auto` 플래그(C모드 전환용)는 v2에서 도입 예정. v1.2는 B모드 고정.
