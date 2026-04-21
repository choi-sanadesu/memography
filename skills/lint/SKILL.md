---
name: lint
description: Alexandria(또는 memorial-vault) Obsidian 볼트의 헬스체크를 수행한다. frontmatter 스키마 위반, 깨진 위키링크, 고립 페이지, inbox 잔여, stale `updated:` 필드, 엔티티 승격 후보 6개 항목을 점검하고 구조화된 리포트를 출력한다. 파일을 자동 수정하지 않는다(v1 기준). "볼트 린트", "린트", "볼트 점검", "헬스체크", "vault lint", "/lint" 같은 표현이 나올 때 트리거.
---

# alexandria-keeper: lint

볼트 헬스체크. `CLAUDE.md §5.4`에 정의된 lint 항목 + 실제 운영 중 재발하는 버그 패턴을 6개 체크로 통합.

## 언제 쓰는가

- 사용자가 "린트", "볼트 점검", "헬스체크" 등을 요청할 때
- ingest 세션 직후, 규칙 위반이 섞였을 가능성이 있을 때
- 주기적 점검 (주간·월간)
- C모드 전환 전 볼트 위생 확인

## 동작 원칙

- **리포트만 출력한다.** v1에서는 어떤 파일도 자동으로 수정하지 않는다.
- **제안은 사용자 확인 후 별도 조치로 진행한다.** 스킬 자체는 읽기 전용으로 동작한다.
- **볼트 규칙은 대상 볼트의 `CLAUDE.md`에서 읽는다.** `domain` enum, `type` enum, 제외 경로 등은 하드코딩하지 않는다. 규칙이 바뀌어도 스킬 수정 없이 동작해야 한다.

## Phase 1: 탐색

1. 대상 볼트 루트 확인 (현재 작업 디렉토리 또는 사용자 지정).
2. 볼트의 `CLAUDE.md`를 읽어 스키마 규칙 파악:
   - `domain` 유효값 (§3.1)
   - `type` 유효값 (§3.1)
   - 필수 frontmatter 필드 (§3.1)
   - 제외 경로 (raw/ 불변, inbox 완충지대 등)
3. 대상 파일 수집:
   - **포함**: 루트 `CLAUDE.md`·`index.md`·`log.md`, 도메인 정의 `_*.md`, `*/wiki/**/*.md`, `04. entities/**/*.md`, `06. studio-sana/**/*.md`
   - **제외**: `.trash/`, `.obsidian/`, `*/raw/**`, `01. inbox/**`

## Phase 2: 6개 체크 실행

각 체크의 상세 규칙·판정 기준·리포트 포맷은 [references/checks.md](references/checks.md) 참조.

| # | 체크 | 방식 |
|---|---|---|
| 1 | frontmatter 스키마 | 결정적 (YAML 파싱 + enum 검사) |
| 2 | 깨진 위키링크 | 결정적 (파일 존재 검사) |
| 3 | 고립 페이지 (백링크 0) | 결정적 (역참조 그래프) |
| 4 | inbox 잔여 | 결정적 (파일 수) |
| 5 | stale `updated:` | 결정적 (90일 기준, 설정 가능) |
| 6 | 엔티티 승격 후보 | LLM 휴리스틱 (2+ 도메인 언급) |

## Phase 3: 리포트 출력

[references/report-template.md](references/report-template.md) 포맷 사용.

요약 테이블 + 체크별 상세 섹션 + 제안 조치 목록을 순서대로 출력.

## Phase 4: 제안 (선택)

사용자가 수정 진행을 요청하면 **별도 Edit/Write 호출로** 건별 확인하며 처리한다. 스킬 내부에서 자동 실행하지 않는다.

다음 조건에서는 자동 수정을 제안하지 않는다:
- 체크 5·6 (stale, 엔티티 후보) — 판정이 주관적
- 체크 3 (고립 페이지) — 의도적 독립 페이지일 수 있음

자동 수정이 안전한 항목:
- 체크 1의 enum 위반 (대체값이 명확할 때만)
- 체크 2의 깨진 링크 중 정확히 한 후보로 대체 가능한 경우

v1에서는 자동 수정 전부 보류. 제안 텍스트만 출력하고 사용자가 판단.

## 출력 예시

실제 리포트 형태는 [references/report-template.md](references/report-template.md)에 있는 템플릿을 따른다. 이번 세션에서 수동으로 찾은 문제 유형을 한 리포트로 묶으면 대략:

- frontmatter: `_sanarchive.md` `domain: sanarchive` → `archive` 제안
- 깨진 링크: `index.md`의 `[[06. studio-sana/rams/MEMORY]]` (대상 미존재)
- 고립 페이지: 없음
- inbox: 0 잔여
- stale: 없음 (전부 2026-04-17 생성)
- 엔티티 후보: Smerz, Erika de Casier (코펜하겐 씬 페이지에서 반복 언급, 엔티티 페이지 없음)

## 한계 (v1)

- **페이지 간 모순 탐지** (§5.4의 1·2·7번 항목)은 별도 `review` 스킬로 분리 예정. 이 스킬은 규칙 기반 검사에 집중.
- **엔티티 후보 감지**는 heuristic이라 false positive가 생긴다. 제안으로만 취급하고 사용자 판단으로 필터.
- 볼트 규모가 커지면 (>500 페이지) 검사 속도 저하 가능. 그 시점에 서브에이전트 병렬화 검토.
