# Retro 템플릿 + 예시

두 모드의 구체적 형식과 실제 사례.

---

## vault-level 템플릿

`log.md`에 append되는 retro 이벤트. 별도 파일 생성 없음.

```markdown
## [YYYY-MM-DD] retro | {제목}

- 도메인: meta
- 추가된 페이지: {있으면 링크, 없으면 "없음"}
- 갱신된 페이지: {회고 맥락에서 수정된 페이지 링크들}
- 핵심 인사이트: {2~3문장. 맥락+배운점+후속조치 압축}
```

### 실제 예시

```markdown
## [2026-04-17] retro | 첫 ingest 테스트 + 구조 갱신

- 도메인: meta
- 추가된 페이지: 없음
- 갱신된 페이지: [[CLAUDE]], [[index]]
- 핵심 인사이트: 첫 ingest(코펜하겐 씬) B모드로 정상 작동. 회고에서 템플릿 갱신 4건, 워크플로우 분기 2건, 환경별 역할 분담 1건 도출. 코드 환경에서 일괄 반영.
```

---

## project-level 템플릿

독립 파일 + `log.md` pointer 2종이 한 쌍.

### 독립 파일

경로: `06. studio-sana/{project}/retros/{YYYY-MM-DD}-{title}.md`

```markdown
---
title: {제목}
type: retro
domain: project
tags: [회고, {project_name}]
created: {YYYY-MM-DD}
updated: {YYYY-MM-DD}
---

# {제목}

## 맥락

{어떤 상황·세션·작업에 대한 회고인지 1~2문단. 시점·관련 문서·참여 도구 명시}

## 잘 된 것

- {point}
- {point}
- {point}

## 부족했던 것

- {point}
- {point}

## 배운 점

- {point}
- {point}

## 후속 조치

- [ ] {구체적 to-do}
- [ ] {to-do}
```

### 섹션 가이드

| 섹션 | 쓰는 법 |
|---|---|
| 맥락 | 언제·어디서·무엇을. 다른 문서가 있으면 링크로 연결 |
| 잘 된 것 / 부족했던 것 | 구체적 사건·결정 위주. 추상적 원칙 금지 |
| 배운 점 | "다음엔 X를 먼저 Y하자" 형태의 실행 가능한 교훈 |
| 후속 조치 | 체크박스 to-do. 완료되면 결과 페이지 링크 추가 |

### log.md pointer

```markdown
## [YYYY-MM-DD] retro | {제목}

- 도메인: project ({project_name})
- 추가된 페이지: [[06. studio-sana/{project}/retros/{YYYY-MM-DD}-{title}]]
- 갱신된 페이지: [[index]]
- 핵심 인사이트: {1문장. 상세는 retro 파일 참조}
```

### 실제 예시 (가상: rams 프로젝트)

독립 파일 `06. studio-sana/rams/retros/2026-04-17-5차-드라이런-회고.md`:

```markdown
---
title: 5차 드라이런 회고
type: retro
domain: project
tags: [회고, rams]
created: 2026-04-17
updated: 2026-04-17
---

# 5차 드라이런 회고

## 맥락

RAMS Phase 0 최종 드라이런. 4에이전트·3게이트 구조로 end-to-end 실행 시도.
결과: Phase 0에서 훅·MCP·settings 3건 FAIL로 중단.

## 잘 된 것

- 에이전트 간 handoff 타입드 JSON 구조 정상 작동
- ADR 24개 누적, 의사결정 맥락 보존
- 드라이런 7회 완주로 패턴 식별 가능

## 부족했던 것

- PreToolUse 훅 플러그인 미지원 (플랫폼 제약)
- 설계 완성도 대비 플랫폼 선검증 부족
- 드라이런과 실전 환경의 갭

## 배운 점

- 플랫폼 제약은 설계 진입 **전** 선검증 필수
- "설계 완성도"와 "실행 가능성"은 다른 축
- 서브에이전트 병렬화와 훅 의존성은 충돌 가능

## 후속 조치

- [ ] 다음 시리즈에서 Phase 0 선행 검증 의무화
- [ ] 플랫폼 제약 체크리스트 볼트에 등록
```

`log.md` pointer:

```markdown
## [2026-04-17] retro | 5차 드라이런 회고

- 도메인: project (rams)
- 추가된 페이지: [[06. studio-sana/rams/retros/2026-04-17-5차-드라이런-회고]]
- 갱신된 페이지: [[index]]
- 핵심 인사이트: 4에이전트 구조 정상 작동했으나 PreToolUse 훅 미지원으로 중단. 플랫폼 선검증의 중요성 재확인.
```

---

## 제목 정제 가이드

| 입력 (날것) | 정제 |
|---|---|
| "오늘 한 작업 회고" | "오늘 작업 회고" (구체화 필요하면 맥락 기반으로 대체) |
| "v5 RAMS 프로젝트 드라이런 5회차 회고하기" | "5차 드라이런 회고" |
| "첫 ingest 테스트했는데 어땠는지 정리" | "첫 ingest 테스트 회고" |
| "4월 17일 구조 갱신 작업 돌아봄" | "구조 갱신 회고" |

원칙:
- 한글 명사구
- 날짜는 파일명·frontmatter `created`에만, `title`에는 없음
- 감상적 동사(돌아봄·살펴봄) 제거, 명사화
- 고유명사·차수·버전 유지

---

## frontmatter 필드

### 독립 파일 (project retro)

| 필드 | 값 | 비고 |
|---|---|---|
| `title` | 정제 제목 | 날짜 없음 |
| `type` | `retro` | CLAUDE.md §3.1 유효값 |
| `domain` | `project` | CLAUDE.md §3.1 유효값 |
| `tags` | `[회고, {project_name}]` | project_name은 폴더명 그대로 (예: `rams`) |
| `created` | 회고 작성일 | YYYY-MM-DD |
| `updated` | 작성일 | 이후 수정되면 갱신 |

### 파일명 규칙

`{YYYY-MM-DD}-{title}.md`

- **예외적으로 날짜 prefix 허용** (CLAUDE.md §3.3의 일반 원칙에 대한 공식 예외)
- 이유: `retros/` 폴더는 시간순 로그 성격
- `{title}`은 파일명 안전 문자로 (스페이스 허용, 콜론·슬래시 금지)
