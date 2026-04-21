# Retro 4단계 상세 워크플로우

각 단계의 입력·처리·B모드 확인·출력.

---

## Step 1. 모드·제목 판정

### 인자 파싱

| 인자 | 처리 |
|---|---|
| 없음 | 대화형 (아래 참조) |
| 문자열 1개 | vault-level, 제목으로 사용 |
| `--project {name} {title}` | project mode |

### 프로젝트 맥락 자동 감지

현재 세션이 다음 중 하나에 해당하면 project mode 전환을 **제안**:

- 최근 편집·참조한 파일이 `06. studio-sana/{name}/` 하위
- 사용자 대화에 프로젝트명(`rams` 등)이 반복 등장
- 회고 내용이 특정 프로젝트 의사결정·구조·마일스톤과 직결

감지되면:

```
프로젝트 {name} 맥락인 것 같은데, project retro로 작성할까요?
- 예: 06. studio-sana/{name}/retros/에 독립 파일 생성 + log.md pointer
- 아니오: vault-level로 log.md에만 append
```

감지 안 되면 바로 vault-level 진행.

### 제목 정제

사용자 입력 제목을 볼트 스타일로 다듬음:

- 한글 명사구, 간결
- 20자 이내 권장
- 날짜 prefix 제거 (파일명엔 붙지만 `title:` 필드엔 없음)
- 감상적 표현 제거 ("되돌아봄", "정리해봄" → 명사화)
- 고유명사·숫자(차수·버전)는 유지

---

## Step 2. 맥락 수집

### 분기

- **세션 내용 있음**: 이전 대화에서 회고 관련 내용을 이미 말했으면 그걸 시안으로 프리필
- **내용 없음**: 대화형 질문으로 수집

### 대화형 질문 세트

한 번에 다 물어도 되고, 1~2개씩 나눠도 OK (사용자 스타일 맞춰).

1. **맥락** — 어떤 상황·세션·작업에 대한 회고?
2. **잘 됐나** — 기대대로 된 것, 효과적이었던 접근?
3. **부족했나** — 어긋난 것, 왜?
4. **배운 점** — 다음에 적용할 교훈?
5. **후속 조치** — 구체적 to-do?

최소 **맥락 + 1~2개 인사이트**는 있어야 의미 있는 retro. 이하면 사용자에게 더 채울지 물어봄.

### 맥락 프리필 주의

세션에 있는 내용을 시안으로 활용할 때:

- 사용자가 명시적으로 "회고 내용으로 쓸 만한 것"이라고 말한 구간 우선
- 추측으로 살 붙이지 말 것 — 불확실한 부분은 "{보강 필요}"로 placeholder 남김
- 프리필 시안을 보여주며 "이대로 갈지, 뭘 보태거나 뺄지" 물어봄

---

## Step 3. 시안 작성

### vault-level (log.md 항목)

```markdown
## [YYYY-MM-DD] retro | {정제 제목}

- 도메인: meta
- 추가된 페이지: {있으면 링크, 없으면 "없음"}
- 갱신된 페이지: {맥락에서 언급된 페이지 링크들}
- 핵심 인사이트: {맥락+배운점 합성, 2~3문장}
```

### project-level (독립 파일 + log.md pointer)

독립 파일: `06. studio-sana/{project}/retros/{YYYY-MM-DD}-{title}.md`

```markdown
---
title: {제목}
type: retro
domain: project
tags: [회고, {project}]
created: {YYYY-MM-DD}
updated: {YYYY-MM-DD}
---

# {제목}

## 맥락
{회고 대상 상황·세션·작업. 1~2문단}

## 잘 된 것
- {point}
- {point}

## 부족했던 것
- {point}
- {point}

## 배운 점
- {point}

## 후속 조치
- [ ] {action}
- [ ] {action}
```

log.md pointer:

```markdown
## [YYYY-MM-DD] retro | {제목}

- 도메인: project ({project_name})
- 추가된 페이지: [[06. studio-sana/{project}/retros/{date}-{title}]]
- 갱신된 페이지: [[index]]
- 핵심 인사이트: {1문장. 상세는 retro 파일 참조}
```

### B모드 확인

**신규 파일(project)·log 항목 모두 사용자 확인 필수:**

```
## 작성 예정

[모드별 프리뷰]

생성해도 될까요?
```

confirm 후 Step 4.

---

## Step 4. 저장

### vault-level

`log.md`의 앞쪽 `---` 구분선 뒤에 새 retro 이벤트를 `Edit` 도구로 append.

기존 항목 사이 간격 관습 유지 (현재 log.md는 빈 줄 1개 간격).

### project-level

1. `06. studio-sana/{project}/retros/` 폴더 없으면 생성 (`mkdir -p`)
2. `{YYYY-MM-DD}-{title}.md` 파일 `Write`
3. `log.md`에 pointer entry append (vault-level 방식과 동일)
4. `index.md`의 해당 프로젝트 섹션에 retros 링크 추가:
   - 이미 `### retros` 서브섹션 있으면 최신 retro 링크 추가
   - 없으면 프로젝트 섹션에 `### retros` 서브섹션 신규 생성

### 자동 실행

Step 3에서 프리뷰 확인을 받았으므로 Step 4 전체는 확인 없이 자동.

---

## 에러 복구

| 실패 단계 | 상태 | 처리 |
|---|---|---|
| Step 1~2 | 변경 없음 | 에러 출력만 |
| Step 3 | 변경 없음 | 동일 |
| Step 4 (vault) | log.md append 실패 | 원인 보고, 수동 추가 안내 |
| Step 4 (project, 파일은 성공 / log·index 실패) | 파일은 그대로 두기 | 수동 log·index 갱신 안내. 파일 롤백 금지 |

원칙: **부분 성공 상태 보존**. 다음 재실행이 이어서 처리하도록.
