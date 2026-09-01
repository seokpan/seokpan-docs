[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-015 — NFS Export 자동화에서 Ansible 후속 실행 작업(Handler) 지연과 Ansible Check Mode(실제 변경 없이 예상 결과를 확인하는 모드) Skip 때문에 검증이 실제 반영 상태를 보장하지 못할 수 있었음

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **Review 예방 · 해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 이유빈(Ansible 통합), 정태훈(Storage 사용/Redis) |

## 문제 개요

NFS Role은:

```text
/etc/exports 수정
→ Ansible 후속 실행 작업(Handler): exportfs -ra
```

구조였다.

하지만 Ansible Handler는 기본적으로 Play 끝에서 실행된다.

그 전에:

```bash
exportfs -v
```

로 검증하면 **새 `/etc/exports` 내용이 Runtime Export Table에 아직 반영되지 않은 상태**를 읽을 수 있다.

또 Check Mode에서는 read-only 검증 명령이 skip될 수 있었다.

## 조치

PR Review에서 다음을 요청했다.

1. 검증 전에:

```yaml
meta: flush_handlers
```

2. read-only 검증에:

```yaml
check_mode: false
changed_when: false
```

3. 실제 Export 존재 Assert 추가

## 검증

수정 후:

```text
nfs : NFS Export 반영
```

Task가 Handler 즉시 실행을 보장하고,

```text
NFS export가 활성화되어 있는지 확인
```

Task가 Check Mode에서도 실행된다.

후속 Assert로:

```text
/srv/nfs/k8s
```

Export 존재를 확인한다.

Dry-run:

```text
failed=0
Export assert PASS
```

### TS-013와의 공통점

두 사례는 구성요소는 다르지만 같은 자동화 품질 문제를 보여준다.

```text
Change를 하지 않는 모드
≠
Validation도 하지 않는 모드
```

즉 **read-only Validation은 Check Mode에서도 실제로 실행되어야 한다.**

## 관련 근거

- Issue #67: https://github.com/seokpan/seokpan-infra/issues/67
- PR #78: https://github.com/seokpan/seokpan-infra/pull/78
