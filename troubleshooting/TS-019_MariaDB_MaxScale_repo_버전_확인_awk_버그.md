[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-019 — MariaDB/MaxScale repo 버전 확인 awk 로직 버그로 인한 Dry-run 신뢰성 저하

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | Ansible 재현성 전반(이유빈), DB Role 전체 |

## 문제 개요

이슈 #63에서 MariaDB/MaxScale repo 버전을 `10.5.29/23.08.13`에서 `11.8.9/24.02.9`로 동기화하며 도입한 "repo가 목표 버전으로 등록되어 있는지 확인"하는 awk 검사 로직 자체에 두 가지 결함이 남아 있었다.

## 원인 분석

### 1. awk range 패턴이 섹션 헤더 한 줄만 추출

```text
잘못된 로직: /^\[mariadb-maxscale\]/,/^\[/  형태의 range 패턴
→ 실제로는 헤더 한 줄만 걸리고 버전 문자열이 있는 baseurl 줄을 검사하지 못함
```

그 결과 이미 목표 버전으로 정확히 등록되어 있어도 매번 "미등록"으로 오판했다. `ansible-playbook`을 실행할 때마다 `mariadb_repo_setup` 스크립트가 불필요하게 재실행되고 `changed`로 보고되어, 재실행 시 아무 상태 변화가 없어야 할 멱등성(Idempotency)이 깨져 있었다.

### 2. 확인 태스크가 Check Mode를 지원하지 않음

확인 태스크가 `ansible.builtin.shell`로 작성돼 있어 `--check` 실행 시 태스크 자체가 skip됐다. 그 결과 dry-run으로는 실제 적용 시 변경이 발생할지 여부를 사전에 전혀 예측할 수 없었다.

```text
--check --diff 실행 → 확인 태스크 skip → "실행해봐야 아는" 상태
```

## 조치

`mariadb`, `maxscale` 두 role의 `setup_repo.yml`에 동일하게 적용했다.

```bash
awk '/^\[mariadb-maxscale\]/{flag=1; next} /^\[/{flag=0} flag' /etc/yum.repos.d/mariadb.repo | grep -q "24.02"
```

- range 패턴을 flag 기반 방식으로 교체해 섹션 전체(baseurl 포함)를 정확히 검사
- 확인 태스크에 `check_mode: false` 추가(실제 상태 변경 태스크인 다운로드/등록 태스크는 기존대로 skip 유지)

## 검증

1. mariadb-01/02, maxscale-01에서 수정된 awk 명령을 수동 실행해 `rc=0` 확인
2. `--check --diff` dry-run 실행 시 확인 태스크가 더 이상 skip되지 않고 실제 실행 결과와 일치하는 예측을 보여줌
3. 이미 목표 버전으로 등록된 서버에서 실제 재실행 시 repo 등록 태스크가 `changed`로 잘못 보고되지 않고 `skipped`로 나옴
4. 의도적으로 버전을 다르게 설정해 drift 상황을 재현 → 정상적으로 재등록(`changed`)됨을 확인

## 운영 기준으로 반영한 내용

이후 Ansible 작업 전반에 다음 원칙을 적용했다.

```text
잘못된 기준: 저장소가 존재하는가?
정확한 기준: 저장소가 존재하고 현재 목표 Version Series를 가리키는가?

조회(확인)만 하는 태스크에는 기본적으로 check_mode: false를 추가해
dry-run 신뢰성을 확보한다.
```

## Before → After

```text
Before
awk range 패턴이 섹션 헤더만 검사
→ 목표 버전이어도 매번 "미등록"으로 오판
→ --check에서는 확인 태스크 자체가 skip
→ dry-run 결과와 실제 실행 결과가 어긋남

After
flag 기반 awk로 섹션 전체 정확히 검사
→ 목표 버전이면 재등록 없이 skipped
→ check_mode: false로 조회 태스크는 --check에서도 항상 실행
→ dry-run이 실제 실행 결과를 정확히 예측
```

## 관련 근거

- Issue #63 (최초 range 방식 도입 배경): https://github.com/seokpan/seokpan-infra/issues/63
- Issue #105: https://github.com/seokpan/seokpan-infra/issues/105
- PR #107: https://github.com/seokpan/seokpan-infra/pull/107
