# 소스 입력 처리

URL·파일 입력의 탐지·해석·로딩 세부 사항.

---

## URL 입력

### 도메인 파싱

URL에서 호스트 추출. 서브도메인 포함 (`www.`, `m.`, `reel.` 등 전부 매칭 대상).

일반 도메인 vs Chrome 도메인 분기.

### Chrome 경유 도메인

아래 패턴에 호스트가 매칭되면 `mcp__Claude_in_Chrome__*` 도구 경유:

| 도메인 패턴 | 이유 |
|---|---|
| `*.google.com` | 검색 결과·Docs 등 JS 렌더링 |
| `*.youtube.com` | 영상 페이지 JS 렌더링 |
| `*.instagram.com` | 로그인 세션 필요 |
| `*.twitter.com` · `*.x.com` | 로그인 세션 필요 |

매칭 예시:
- `www.instagram.com/reel/...` ✓
- `twitter.com/user/status/...` ✓
- `x.com/user/status/...` ✓
- `m.youtube.com/watch?v=...` ✓

### Chrome 도구 사용 순서

1. `mcp__Claude_in_Chrome__navigate` — URL로 이동
2. `mcp__Claude_in_Chrome__read_page` 또는 `get_page_text` — 텍스트 추출
3. 필요시 `mcp__Claude_in_Chrome__read_console_messages` — 동적 로딩 실패 진단

소스별 추출 포인트:
- **Instagram 릴스/포스트**: 캡션, 댓글 상위 몇 개
- **Twitter/X**: 트윗 본문, 쓰레드면 연속 트윗
- **YouTube**: 제목, 설명란, 자막(있으면)
- **Google 검색**: 상위 결과 제목·스니펫

### ⚠️ 보안 제약 (절대 준수)

대상 볼트의 `CLAUDE.md §10`(보안·준수 원칙)을 **반드시 준수**한다. 단 한 번의 경계 침범도 허용되지 않는다.

**허용 범위:**
- 렌더링된 공개 콘텐츠 텍스트 추출 (`get_page_text`·`read_page`)
- 사용자 명시 지시로 이루어지는 UI 상호작용 (클릭·입력)
- 스크린샷·스냅샷

**금지 행위 (예외 없음):**
- 내부 API·관리자 엔드포인트 탐색·호출·탈취 시도
- 인증·권한 우회 (토큰 탈취, 세션 하이재킹 등)
- 보호된 데이터 추출 (네트워크 payload 가로채기, DOM private 필드 등)
- 제3자 사적 정보 접근
- `javascript_tool`·`preview_eval`·`eval` 계열로 데이터 긁기 (디버깅·사용자 명시 지시 외)
- 보안 미보장 사이트 접근 (HTTPS 미지원·브라우저 경고 발생)

**불확실 시:** 수행 금지, 사용자 보고. "될까?" 호기심으로 경계 시험 금지. 위반 가능성 감지 시 즉시 중단.

**원칙: 스킬은 사용자 세션의 연장이다. 사용자가 수동으로 하지 않을 행위는 스킬도 하지 않는다.**

### Chrome MCP 미연결 환경

사용자의 현재 환경에 Chrome MCP가 없으면 (터미널 Claude Code 단독 등):

```
⚠️ Chrome MCP 연결 없음

이 URL({host})은 브라우저 세션이 필요합니다.

대안:
1. 코워크·Chrome 연결된 환경에서 재실행
2. 브라우저에서 직접 열고 본문 복사
   → 로컬 파일로 저장 (예: 01. inbox/{이름}.md)
   → `/ingest {파일경로}`로 재호출
```

`WebFetch`로 강제 시도 금지 — 로그인 벽·JS 렌더링 때문에 빈 응답이거나 오도될 위험.

### 일반 URL

Chrome 도메인이 아니면 `WebFetch`. 기본 HTTP GET + HTML→text.

응답이 비거나 "JavaScript를 켜주세요" 등 안내 페이지면 **Chrome 경로 폴백** 제안:

```
⚠️ 이 페이지는 JS 렌더링이 필요해 보입니다.
Chrome MCP 경로로 재시도할까요?
```

---

## 파일 입력

### 경로 해석

| 입력 | 처리 |
|---|---|
| 절대 경로 | 그대로 |
| `01. inbox/*` | 볼트 루트 기준 |
| 상대 경로 | 현재 작업 디렉토리 기준 |

### 확장자별 로딩

| 확장자 | 도구 | 비고 |
|---|---|---|
| `.md` · `.txt` · `.html` | `Read` | 직접 |
| `.pdf` | `Read` + `pages` 인자 | 10페이지 초과 시 `pages: "1-10"` 우선 |
| `.png` · `.jpg` · `.jpeg` · `.webp` | `Read` | 비전 모델로 내용 파악 |
| `.docx` · `.xlsx` · `.pptx` | anthropic-skills의 해당 스킬 경유 | 별도 스킬 있으면 활용 |
| `.json` · `.csv` · `.xml` | `Read` | 구조화 데이터, 그대로 |
| 기타 | 미지원 | 사용자에게 텍스트 변환 요청 |

### 원본 이동 (Step 4)

- `01. inbox/` 안의 파일 → **자동 이동** (확인 없음, 복구 가능하므로)
- 이동 대상: `{domain}/raw/`
  - sanarchive 특수: briefs/ vs deliverables/ 중 파일 성격으로 선택 (외부 의뢰 자료는 briefs, 산출물은 deliverables)
- 충돌 시: 파일명에 ` (2)`·` (3)` 등 suffix로 회피
- 볼트 밖 파일: **이동 금지**. "볼트로 복사할까요?" 확인 후 복사만

이동 도구는 `Bash mv` 또는 동등 수단.

---

## 원제 추출 (Step 5 제목 정제 힌트)

| 소스 | 원제 추출 위치 |
|---|---|
| URL (일반 fetch) | `<title>` 태그 |
| URL (Chrome) | `get_page_text` 시작부 또는 페이지 `<title>` |
| `.md` 파일 | frontmatter `title:` → 없으면 첫 H1 → 없으면 파일명 |
| `.pdf` | PDF 메타데이터 → 없으면 첫 페이지 큰 글자 → 없으면 파일명 |
| 이미지 파일 | 파일명 (EXIF 캡션 있으면 보조) |
| 기타 | 확장자 제거 basename |

**원제는 힌트에 불과.** 최종 제목은 LLM이 볼트 스타일로 정제:

- 과한 수식어 제거 (예: "꼭 알아야 할 5가지" → 실체 키워드만)
- 한글 명사구화
- 20자 이내 권장, 초과 시 핵심만
- 고유명사는 보존

---

## 예시

### Case A — 인스타 릴스

```
입력: https://www.instagram.com/reel/DTPIPGtk-u6/

1. 호스트 = www.instagram.com → Chrome 경로
2. navigate → 페이지 이동
3. get_page_text → 릴스 캡션 + 상위 댓글 추출
4. 영상 자체 내용은 사용자 브리프에 의존 (v1 out of scope)
5. 원제: 없음 → 사용자 브리프 또는 캡션 첫 줄이 힌트
```

### Case B — 일반 기사

```
입력: https://www.example.com/news/article-id

1. 호스트 = www.example.com → Chrome 도메인 아님
2. WebFetch → HTML→text
3. <title> 추출 → 원제 힌트
4. 응답 OK → Step 2로
```

### Case C — inbox의 PDF

```
입력: 01. inbox/report.pdf

1. Read + pages: "1-10" (우선 앞 10페이지)
2. PDF 메타의 Title 필드 확인 → 원제 힌트
3. 분류 → `02. enterwiki/raw/report.pdf`로 자동 이동
4. 이동 후 Step 5에서 페이지 작성
```

### Case D — 볼트 밖 이미지

```
입력: /Users/사용자/Downloads/ref.jpg

1. 볼트 밖 → 이동 대신 복사 확인
2. 사용자 confirm → `03. sanawiki/raw/ref.jpg`로 복사 (원본 보존)
3. Read (비전) → 이미지 내용 파악
4. 도메인 분류 → sana/비주얼
```
