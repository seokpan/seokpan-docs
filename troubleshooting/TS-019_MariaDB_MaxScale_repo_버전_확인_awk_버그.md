[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-019 — MariaDB/MaxScale Repository 버전 확인 오류로 Dry-run 결과가 실제 실행과 달라짐

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MariaDB/MaxScale Ansible Role의 Repository 버전 확인과 Dry-run 결과 |

## 문제 개요

Issue #63에서 MariaDB/MaxScale Repository 버전을 `10.5.29/23.08.13`에서 `11.8.9/24.02.9`로 맞추면서, 현재 Repository가 목표 버전을 가리키는지 확인하는 `awk` 검사 로직을 추가했다. 그러나 이 검사 로직에 두 가지 결함이 남아 있었다.

## 원인 분석

### 1. `awk` 범위 패턴이 섹션 헤더 한 줄만 검사

```text
잘못된 로직: /^\[mariadb-maxscale\]/,/^\[/  형태의 range 패턴
→ 실제로는 헤더 한 줄만 걸리고 버전 문자열이 있는 baseurl 줄을 검사하지 못함
```

그 결과 이미 목표 버전이 정확히 등록되어 있어도 매번 "미등록"으로 잘못 판단했다. `ansible-playbook`을 실행할 때마다 `mariadb_repo_setup` 스크립트가 불필요하게 다시 실행되고 `changed`로 보고되어, 같은 설정으로 다시 실행하면 변경이 없어야 하는 멱등성(Idempotency)이 깨져 있었다.

### 2. 확인 Task가 Check Mode에서 실행되지 않음

확인 Task가 `ansible.builtin.shell`로 작성되어 있어 `--check` 실행 시 Task 자체가 `skipped` 처리됐다. 그 결과 Dry-run만으로는 실제 실행 시 Repository 변경이 발생할지 미리 판단할 수 없었다.

```text
--check --diff 실행
→ Repository 버전 확인 Task가 skipped
→ 실제 실행 전에는 변경 여부를 알 수 없음
```

## 조치

`mariadb`, `maxscale` 두 Role의 `setup_repo.yml`에 동일하게 적용했다.

```bash
awk '/^\[mariadb-maxscale\]/{flag=1; next} /^\[/{flag=0} flag' /etc/yum.repos.d/mariadb.repo | grep -q "24.02"
```

- 범위 패턴을 flag 기반 방식으로 바꿔 섹션 전체와 `baseurl`을 정확히 검사
- 상태 확인 Task에 `check_mode: false`를 추가해 `--check`에서도 실제 조회가 수행되도록 함
- 실제 상태를 변경하는 다운로드·Repository 등록 Task는 기존대로 Check Mode에서 실행하지 않음

## 검증

1. `mariadb-01/02`, `maxscale-01`에서 수정된 `awk` 명령을 수동 실행해 `rc=0` 확인
2. `--check --diff` 실행 시 Repository 버전 확인 Task가 더 이상 `skipped`되지 않고 실제 실행 결과와 일치하는 변경 예측을 보여주는지 확인
3. 이미 목표 버전으로 등록된 서버에서 실제 재실행 시 Repository 등록 Task가 `changed`로 잘못 보고되지 않고 `skipped`되는지 확인
4. 의도적으로 다른 버전을 설정한 상태를 재현해 정상적으로 재등록(`changed`)되는지 확인

## 운영 기준으로 반영한 내용

이후 Repository 버전 확인은 단순히 파일이나 섹션의 존재 여부가 아니라 **현재 설정이 목표 버전을 실제로 가리키는지**까지 확인한다.

```text
잘못된 기준
Repository 설정이 존재하는가?

현재 기준
Repository 설정이 존재하고 목표 Version Series를 가리키는가?
```

조회만 수행하는 Task는 Check Mode에서도 현재 상태를 확인할 수 있도록 구성해 Dry-run 결과와 실제 실행 결과의 차이를 줄인다.

## Before → After

```text
Before
awk 범위 패턴이 섹션 헤더만 검사
→ 목표 버전이어도 매번 "미등록"으로 잘못 판단
→ --check에서는 확인 Task 자체가 skipped
→ Dry-run 결과와 실제 실행 결과가 어긋남

After
flag 기반 awk로 섹션 전체와 baseurl을 검사
→ 목표 버전이면 재등록 없이 skipped
→ 조회 Task는 --check에서도 현재 상태 확인
→ Dry-run에서 실제 변경 여부를 사전에 판단 가능
```

## 관련 근거

- Issue #63 — 최초 범위 패턴 도입 배경: https://github.com/seokpan/seokpan-infra/issues/63
- Issue #105: https://github.com/seokpan/seokpan-infra/issues/105
- PR #107: https://github.com/seokpan/seokpan-infra/pull/107
