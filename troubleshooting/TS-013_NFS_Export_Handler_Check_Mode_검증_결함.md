[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-013 — NFS Export 자동화에서 Handler 지연과 Ansible Check Mode Skip 때문에 검증이 실제 반영 상태를 보장하지 못할 수 있었음

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **Pull Request 검토 예방 · 해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 이유빈(Ansible 통합), 정태훈(Storage 사용/Redis) |

## 문제 개요

NFS Server의 `/etc/exports`를 Template로 관리하고 변경 시 Handler에서 `exportfs -ra`를 실행하도록 자동화했다. 구조 자체는 일반적이지만 검토 과정에서 검증 순서 문제가 발견됐다.

Ansible Handler는 기본적으로 Play 후반에 실행될 수 있으므로 다음 순서가 가능했다.

```text
/etc/exports 변경
→ 바로 exportfs -v 검증
→ 아직 Handler 미실행
```

이 경우 파일에는 새 값이 있지만 실제 Export Table에는 아직 반영되지 않은 상태를 검증할 수 있었다.

또한 `exportfs -v` 같은 read-only 조회 Task가 Check Mode에서 Skip되면 `--check`가 성공하더라도 실제 Export 상태를 확인하지 않은 것이 된다.

## 조치

1. `configure_exports` 이후 `meta: flush_handlers` 실행
   - `exportfs -ra`가 검증보다 먼저 수행되도록 순서 보장
2. `exportfs -v` 조회 Task에 `check_mode: false`
   - Check Mode에서도 read-only 상태 검증 수행
3. 두 Export가 실제 출력에 존재하는지 `assert` 추가
   - `/srv/nfs/k8s`
   - `/srv/nfs/db-backup`

## 검증

수정 후 dry-run 결과:

```text
ok=8
changed=0
failed=0
```

두 Export Assert 모두 통과했다.

## 핵심 원칙

Kubernetes 자동화에서 확인한 것과 같은 원칙이 NFS 자동화에도 적용됐다.

> **Check Mode에서 변경을 막는 것과 상태 확인 명령까지 생략하는 것은 다른 문제다.**

이 사례에서는 Handler 실행 시점과 Check Mode의 read-only 검증 조건을 맞춰 실제 반영 상태를 기준으로 검증하도록 수정했다.

## 관련 근거

- NFS Role PR #78: https://github.com/seokpan/seokpan-infra/pull/78
- 검토 반영 Comment: https://github.com/seokpan/seokpan-infra/pull/78#issuecomment-5479385513
- Merge Commit: https://github.com/seokpan/seokpan-infra/commit/1643b2583895a528813d5273532a64fa655fa483
