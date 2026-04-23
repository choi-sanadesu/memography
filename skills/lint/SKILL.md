---
name: lint
description: Memography Obsidian 볼트의 헬스체크를 수행한다. frontmatter 스키마 위반, 깨진 위키링크, 고립 페이지, inbox 잔여, stale updated 필드, 태그 카탈로그 위반 6개 항목을 점검하고 구조화된 리포트를 출력한다. 파일을 자동 수정하지 않는다(v0.1 기준). type enum은 inbox, entity, source, case, reference, note, meta 기준 (inbox 마커는 inbox/ 폴더 위치에서만 유효). "볼트 린트", "린트", "볼트 점검", "헬스체크", "vault lint", "/lint" 같은 표현이 나올 때 트리거.
---

# memography: lint

볼트 헬스체크. 볼트 `CLAUDE.md §5.4`에 정의된 lint 항목 + 운영 중 재발하는 패턴을 6개 체크로 통합.

## 언제 쓰는가

- 사용자가 "린트", "볼트 점검", "헬스체크" 등을 요청할 때
- ingest·classify 세션 직후, 규칙 위반이 섞였을 가능성이 있을 때
- 주기적 점검 (주간·월간)
- C모드 전환 전 볼트 위생 확인

## 동작 원칙

- **리포트만 출력한다.** v0.1에서는 어떤 파일도 자동으로 수정하지 않는다.
- **제안은 사용자 확인 후 별도 조치로 진행한다.** 스킬 자체는 읽기 전용으로 동작한다.
- **볼트 규칙은 대상 볼트의 `CLAUDE.md`에서 읽는다.** `type` enum, `media` enum(있으면), 필수 필드 목록, 제외 경로 등은 하드코딩하지 않는다. 규칙이 바뀌어도 스킬 수정 없이 동작해야 한다. 볼트 §2에 특정 enum이 없으면 해당 enum **값 검증은 skip**(warn도 fail도 아님), 다른 검증(필드 존재성·형식)은 계속 수행.

## Phase 1: 탐색

1. 대상 볼트 루트 확인 (현재 작업 디렉토리 또는 사용자 지정).
2. 볼트의 `CLAUDE.md`를 읽어 스키마 규칙 파악:
   - `type` 유효값 (§2)
   - `media` 유효값 (§2, 정의돼 있으면. 모든 type에 선택 적용)
   - 필수 frontmatter 필드 (§2)
   - 태그 카탈로그 (`meta/tags.md` 경로)
   - 제외 경로 (raw/ 불변, inbox 완충지대 등)
3. 대상 파일 수집:
   - **포함**: 루트 `CLAUDE.md`·`index.md`·`log.md`·`hot.md`, `entities/**/*.md`, `sources/*.md`, `cases/*.md`, `references/*.md`, `notes/*.md`, `meta/tags.md`
   - **제외**: `inbox/**`, `raw/**`, `.obsidian/`, `.git/`, `.trash/`

## Phase 2: 6개 체크 실행

각 체크의 상세 규칙·판정 기준·리포트 포맷은 [references/checks.md](references/checks.md) 참조.

| # | 체크 | 방식 |
|---|---|---|
| 1 | frontmatter 스키마 | 결정적 (YAML 파싱 + enum + summary 길이 + tags 개수) |
| 2 | 깨진 위키링크 | 결정적 (본문 `[[...]]` + `related:` 구조화 YAML 양쪽) |
| 3 | 고립 페이지 (백링크 0) | 결정적 (역참조 그래프, related 우선) |
| 4 | inbox 잔여 | 결정적 (파일 수 + 3일 stale) |
| 5 | stale `updated:` | 결정적 (90일 기준, 설정 가능) |
| 6 | 태그 카탈로그 위반 | 결정적 (`meta/tags.md` 외 태그 사용 감지) |

## Phase 3: 리포트 출력

[references/report-template.md](references/report-template.md) 포맷 사용.

요약 테이블 + 체크별 상세 섹션 + 제안 조치 목록을 순서대로 출력.

## Phase 4: 제안 (선택)

사용자가 수정 진행을 요청하면 **별도 Edit/Write 호출로** 건별 확인하며 처리한다. 스킬 내부에서 자동 실행하지 않는다.

다음 조건에서는 자동 수정을 제안하지 않는다:
- 체크 5 (stale) — 판정이 주관적
- 체크 3 (고립 페이지) — 의도적 독립 페이지일 수 있음

자동 수정이 안전한 항목:
- 체크 1의 enum 위반 (대체값이 명확할 때만)
- 체크 2의 깨진 링크 중 정확히 한 후보로 대체 가능한 경우

v0.1에서는 자동 수정 전부 보류. 제안 텍스트만 출력하고 사용자가 판단.

## 출력 예시

실제 리포트 형태는 [references/report-template.md](references/report-template.md)에 있는 템플릿을 따른다. 가상 시나리오:

- frontmatter: `entities/people/뉴진스.md` `summary` 누락 (필수 필드)
- 깨진 링크: `cases/2026-spring-campaign.md`의 `related.entities: ["[[entities/people/erika-de-casier|에리카]]"]` (대상 미존재)
- 고립 페이지: 1건 (`references/seoul-shopping.md` — 어디서도 링크되지 않음)
- inbox: 4건 (그중 1건 5일 경과)
- stale: 없음
- 태그 카탈로그 위반: `notes/methodology.md`에 미등록 태그 `메서드론` 사용

## 한계 (v0.1)

- **페이지 간 모순 탐지** (예: 동일 인물에 대한 상충 진술)는 별도 review 스킬로 분리 예정. 이 스킬은 규칙 기반 검사에 집중.
- 볼트 규모가 커지면 (>500 페이지) 검사 속도 저하 가능. 그 시점에 서브에이전트 병렬화 검토.
- 신규 엔티티 후보 감지(옛 alexandria-keeper Check 6)는 classify 스킬 Step 2~3 B모드에 흡수. lint는 더 이상 다루지 않음.
