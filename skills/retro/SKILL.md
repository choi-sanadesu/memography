---
name: retro
description: Alexandria(또는 memorial-vault) 볼트에 회고(retrospective) 항목을 기록한다. 볼트 전체 회고는 `log.md`에 retro 이벤트로 append, 프로젝트 단위 회고는 `06. studio-sana/{project}/retros/{YYYY-MM-DD}-{title}.md`로 독립 파일 생성 + `log.md`에 pointer entry 추가. B모드(신규 파일 생성은 확인, log.md append는 자동) 기준. "회고", "retro", "/retro", "회고 남겨", "retro 추가", "회고 기록" 같은 표현이 나올 때 트리거.
---

# alexandria-keeper: retro

회고 기록 표준화. 볼트 전체 회고와 프로젝트별 회고를 일관된 포맷으로 남기고, `log.md`에 시간순 인덱스를 유지.

## 언제 쓰는가

- 세션 종료 시점의 회고 (무엇이 됐고 뭐가 부족했나)
- 큰 구조 변경 후 결과 돌아보기
- 프로젝트 마일스톤 회고
- `/lint` 실행 후 발견한 패턴에 대한 단상

## 두 가지 모드

### vault-level (meta 도메인)

- 볼트 전체 운영·구조·워크플로우 회고
- 산출물: `log.md`에 retro 이벤트 append만
- 용도: "첫 ingest 테스트 회고", "구조 갱신 회고" 같은 범용

### project-level (project 도메인)

- 특정 프로젝트(`06. studio-sana/{name}/`) 회고
- 산출물:
  1. `06. studio-sana/{name}/retros/{YYYY-MM-DD}-{title}.md` 독립 파일
  2. `log.md`에 짧은 pointer entry (볼트 전역 시간축 유지)
- 용도: 프로젝트 내부 의사결정·마일스톤 리뷰

## 입력

| 호출 | 동작 |
|---|---|
| `/retro` | 모드·제목 대화형으로 묻는다 (기본 vault-level 가정) |
| `/retro {title}` | vault-level, 제목 지정 |
| `/retro --project {name} {title}` | project mode |

## 동작 원칙

- 대상 볼트의 `CLAUDE.md §6.2`(log.md 포맷)·`§7`(프로젝트 메모리) 규칙을 **읽어서** 기준으로 삼는다.
- `retros/` 파일명은 `CLAUDE.md §3.3`의 "날짜 prefix 금지" 규칙의 **예외**. 시간순 로그 성격이라 `{YYYY-MM-DD}-{title}.md` 허용.
- 세션 대화에 이미 회고 내용이 있으면 시안을 프리필하고 사용자 보강 요청. 없으면 대화형 질문.
- Mode 기본값은 **vault-level**. 현재 작업 맥락이 특정 프로젝트라면 project mode 전환 제안.

## 4단계 워크플로우

상세는 [references/workflow.md](references/workflow.md).

| # | 단계 | B모드 확인 |
|---|---|---|
| 1 | 모드·제목 판정 | 무인자면 ✋ |
| 2 | 맥락 수집 | 세션 내용 있으면 시안 프리필, 없으면 대화형 |
| 3 | 시안 작성 | ✋ 프리뷰 후 확인 |
| 4 | 저장 | 자동 (log.md / retros/ 둘 다) |

## 템플릿

두 모드의 구체적 형식·실제 예시는 [references/templates.md](references/templates.md).

## 완료 출력

### vault-level

```
✓ retro 기록 (vault-level)
- log 추가: ## [{date}] retro | {title}
- 갱신된 페이지: [[log]]
```

### project-level

```
✓ retro 기록 (project: {name})
- 신규 페이지: [[06. studio-sana/{name}/retros/{date}-{title}]]
- 갱신된 페이지: [[log]], [[index]]
```

## 한계 (v1)

- **자동 맥락 추출 없음**. 세션 대화를 맥락으로 쓰되, 요약은 LLM 현장 판단.
- **결합 회고 없음**. 여러 세션을 하나의 retro로 묶기는 v2.
- **템플릿 커스터마이징 없음**. v1은 고정 구조(맥락·잘된점·부족한점·배운점·후속조치).
