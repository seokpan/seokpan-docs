# TS-018: Harbor Tag Immutability에서 Candidate Tag(scan-*) 예외 정책이 두 차례 무력화된 문제

## 개요

- **분류**: 예방형 문제 (사전 검토·구현 과정에서 발견, 실제 운영 장애는 아님)
- **영향 시스템**: Harbor Registry (`harbor.seokpan.soldesk.store`), `seokpan` Project
- **관련 자동화**: `seokpan-infra` Ansible `harbor` Role (`roles/harbor/tasks/immutability.yml`)
- **관련 작업**: seokpan-app Issue #58 (main Image Pipeline) 선행 조건 구현
- **관련 Issue/PR**: seokpan-infra Harbor Tag Immutability `scan-*` 예외 정책 PR, 리뷰 코멘트 Issue #155
- **작성일**: 2026-09-08
- **작성자**: 최유준

## 배경

seokpan-app Issue #58(main Image Pipeline)은 Candidate Tag(`scan-<main-sha-12>-<BUILD_NUMBER>`)를 Harbor에 Push한 뒤 원격 Vulnerability Scan을 수행하고, 통과 시 Final Tag(`git-<main-sha-12>`)를 같은 Artifact에 추가하는 "Candidate Push → Scan → Promote" 방식을 채택했다.

이 방식이 성립하려면 Harbor의 Tag Immutability 정책이 `git-*`(Final Tag)는 계속 보호하되 `scan-*`(Candidate Tag)는 자유롭게 재생성·삭제할 수 있어야 한다. 그러나 seokpan `harbor` Ansible Role은 기존에 `Repository ** / Tag ** immutable` 단일 규칙으로 모든 Tag를 보호하고 있었으므로(#110에서 구현), `scan-*`에 대한 예외를 추가하는 작업이 필요했다.

## 영향

- 예외 정책이 정확히 적용되지 않으면 Candidate Tag를 재사용하거나 Scan 실패 후 정리(Cleanup)하는 것이 불가능해져, Issue #58의 main Image Pipeline 자체를 구현할 수 없는 상태였다.
- 실제 서비스 장애는 발생하지 않았다. 모든 문제는 구현·리뷰 단계에서 발견되어 운영 반영 전에 수정되었다.

## 원인

이 작업은 세 단계에 걸쳐 서로 다른 원인의 결함이 발견되고 수정되었다. 판단이 검증을 거치며 바뀐 과정을 그대로 남긴다.

### 1차 시도 — `matches` + `excludes`를 하나의 Rule에 함께 사용 (실패)

최초 설계는 기존 Rule의 `tag_selectors`에 `matches: "**"`와 `excludes: "scan-**"`를 함께 넣어 "scan-* 제외 전체 immutable"을 표현하려 했다.

- PUT 요청에 `id` 필드를 누락하여 `400 BAD_REQUEST: "the immutable_rule_id doesn't match the id in the payload body of ImmutableRule"` 발생 → **원인 1**: Harbor `PUT /immutabletagrules/{id}` API는 URL의 rule id와 body 안의 `id` 필드가 일치해야 한다. body에 `id`가 없으면 불일치로 간주되어 거부된다.
- `id` 필드를 추가해 PUT 자체는 `200`으로 성공했으나, 실제로 `scan-*` Tag를 삭제 시도하면 여전히 `412 PRECONDITION: "configured as immutable, cannot be deleted"`로 거부됨을 확인 → **원인 2**: Harbor Immutability Rule의 `tag_selectors`는 selector를 정확히 1개만 허용한다(Harbor UI도 matching/excluding 중 하나만 선택 가능한 구조). `matches`와 `excludes`를 함께 넣으면 저장은 되지만(`200`) 평가 시 AND로 조합되지 않아 예외가 무력화된다.

### 2차 시도 — `excludes` 단일 selector로 전환 (부분 성공, 새로운 결함 발견)

`tag_selectors`를 `excludes: "scan-**"` 단일 항목으로 교체하는 방식으로 수정했다. 이 형태 하나만으로 "scan-** 패턴을 제외한 모든 Tag"를 의미하며, 실제로 `scan-*` 생성(`201`)·삭제(`200`) 및 `git-*` 삭제 거부(`412`)까지 모두 정상 동작함을 확인했다(1차 검증 완료).

### 3차 — 레거시 Rule과 목표 Rule 동시 존재 시 수렴 실패 (PR 리뷰에서 발견, 실제 운영 반영 전 수정)

PR 리뷰(Issue #155)에서 다음 결함이 지적되었다: Harbor Immutability Rule은 같은 scope에 여러 개 존재하면 OR로 평가되므로, 레거시 `matches "**"` Rule이 하나라도 남아 있으면 목표 `excludes "scan-**"` Rule을 별도로 갖고 있어도 `scan-*`는 계속 immutable로 묶인다. 그런데 기존 Ansible 로직은:

- 레거시 Rule과 목표 Rule을 각각 단건으로 탐색
- 레거시 Rule이 있고 목표 Rule이 **없을 때만** 레거시 Rule을 목표 상태로 전환
- 목표 Rule이 이미 존재하면 레거시 Rule은 그대로 방치
- 최종 검증도 목표 Rule의 존재 여부만 확인

즉 "레거시 + 목표가 동시에 존재하는 상태"에서는 아무 조치도 하지 않고, 검증도 이 상태를 통과시켜 버리는 구조였다. 수동으로 레거시 Rule을 재현하여 확인한 결과, 이 상태에서 `scan-*` 삭제를 시도하면 실제로 `412`로 재차 거부됨을 확인했다(문제 재현).

## 조치

### `roles/harbor/defaults/main.yml`
`harbor_immutability_tag_exclude_pattern: "scan-**"` 변수 추가.

### `roles/harbor/tasks/immutability.yml` 전면 개편
1. 목표 상태를 `matches`+`excludes` 조합이 아닌 **`excludes` 단일 selector**로 확정.
2. 단건(레거시 1개/목표 1개) 탐색 방식을 **동일 scope(repository pattern 일치, `action=immutable`, `template=immutable_template`) 내 Rule 전체 수집** 방식으로 변경. 수집된 Rule을 "정확히 목표 상태(excludes 단일 selector)"와 "그 외 전부(충돌 Rule)"로 분리.
3. 목표 Rule이 없고 충돌 Rule이 있으면, 그중 하나를 신규 생성이 아니라 **PUT으로 목표 상태로 전환**(id 재사용).
4. **Cleanup 단계 신설**: 목표 Rule 존재 여부와 무관하게, 목표 상태로 전환되지 않고 남은 충돌 Rule은 전부 DELETE.
5. **최종 검증 강화**: 목표 Rule의 존재·활성 여부뿐 아니라 **scope 내 immutable Rule 총 개수가 정확히 1개**인지까지 assert하여, 충돌 Rule 잔존 상태를 더 이상 통과시키지 않도록 함.
6. 모든 PUT 요청 body에 `id` 필드를 명시적으로 포함(1차 시도에서 발견한 결함 재발 방지).

## 검증

| 항목 | 결과 |
|---|---|
| 최초 실행(레거시 → 목표 전환) | `changed=1` |
| 재실행(수렴 확인) | `changed=0` |
| `git-*` Tag 삭제 시도 | `412` (보호 유지, 회귀 없음) |
| `scan-*` Tag 생성/삭제 | 생성 `201`, 삭제 `200` (예외 정상 동작) |
| **레거시+목표 동시 존재 상태 인위 재현** | `matches "**"` Rule을 API로 직접 추가하여 목표 Rule과 공존시킴 |
| 재현된 문제 확인 | 이 상태에서 `scan-*` 삭제 시도 → `412` (결함 재현) |
| 개편된 Role 실행 | `changed=1`, 로그상 레거시 Rule(id) 삭제(DELETE) 확인 |
| 정리 후 재조회 | scope 내 Rule 정확히 1개(목표 상태)만 남음을 확인 |
| 정리 후 `scan-*` 삭제 재시도 | `200` (성공, 문제 해소) |
| 정리 후 `git-*` 삭제 재시도 | `412` (여전히 보호됨, 회귀 없음) |
| 개편된 Role 재실행 | `changed=0, failed=0` (완전 수렴) |

모든 상태 전이(최초 생성 → 재실행 수렴 → 레거시/목표 동시 존재 → 정리 → 재수렴)가 실제 Harbor API 호출로 검증되었다.

## 재발 방지 / 확인된 API 제약 (팀 지식으로 기록)

1. Harbor `PUT /immutabletagrules/{id}` 호출 시 body에 `id` 필드를 반드시 포함해야 한다. 누락 시 `400 BAD_REQUEST`.
2. Harbor Immutability Rule의 `tag_selectors`는 selector를 정확히 1개만 허용한다. `matches`와 `excludes`를 함께 넣으면 저장은 되지만 평가 시 무시되어 예외가 동작하지 않는다.
3. Harbor Immutability Rule은 같은 scope에 여러 개 존재하면 OR로 평가된다. 이 프로젝트의 Ansible 자동화는 이제 "목표 상태가 아닌 Rule 전체 정리"까지 수행하도록 개편되어, 향후 유사한 중간 상태(레거시/목표 동시 존재)가 재발해도 재실행만으로 자동 정리된다.
4. Harbor Robot Account 조회 API(`GET /api/v2.0/robots`)는 필터 없이 호출하면 project-level 계정이 결과에 잡히지 않는다(`X-Total-Count: 0`). `q=Level=project,ProjectID=<id>` 필터가 반드시 필요하다. (본 사건과는 별도 작업에서 확인된 동일 API의 또 다른 함정이나, 같은 Harbor Immutability/Robot 관련 자동화를 다루는 팀원에게 유용하여 함께 기록한다.)

## 관련 자료

- seokpan-app Issue #58 (main Image Pipeline)
- seokpan-infra Harbor Tag Immutability `scan-*` 예외 정책 PR
- PR 리뷰 코멘트 (Issue #155, 레거시/목표 동시 존재 수렴성 문제 지적)
- seokpan-infra `roles/harbor/tasks/immutability.yml`, `roles/harbor/defaults/main.yml`
- 기존 Immutability 구현 PR #110 (`Repository ** / Tag ** immutable` 최초 도입)

## 후속 운영 기준

- `scan-**` 패턴은 Immutability 예외 대상이므로, 이 패턴의 Tag는 GitOps 등 어떤 배포 경로에서도 참조하지 않는다. 배포 대상은 `git-*` Tag만 사용한다(Issue #58 계약).
- 향후 동일 scope에 다른 목적의 immutable Rule을 추가할 필요가 생기면, 현재 Cleanup 로직이 "목표 상태가 아닌 Rule은 전부 삭제"하도록 되어 있어 충돌할 수 있다. 이 경우 scope 분리 또는 판정 로직 재검토가 선행되어야 한다.
