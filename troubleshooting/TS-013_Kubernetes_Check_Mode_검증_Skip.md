[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-013 — Kubernetes 통합 Playbook의 Ansible Check Mode(실제 변경 없이 예상 결과를 확인하는 모드)에서 검증 명령이 Skip되어 Assert가 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 이유빈(Ansible 통합), Kubernetes 자동화를 실행·검증하는 팀원 전체 |

## 문제 개요

Kubernetes 전체 자동화 재현 검증을 위해 `kubernetes_cluster.yml`이 확장됐다.

구조:

```text
Preflight Validation
→ cp-01 Bootstrap
→ cp-02/03 Join
→ worker-01/02 Join
→ Calico
→ Post Validation
```

실제 Runtime은 이미 정상이고, 최신 `main` 브랜치에서 Syntax Check/Check Mode를 검증했다.

## 증상

Check Mode에서 read-only 검증 `command`까지 skip되면서 이후 `assert`가 참조할 변수가 만들어지지 않았다.

즉:

```text
변경 Task Skip
→ 정상

read-only Validation도 Skip
→ register 변수 없음
→ Assert 실패
```

## 조치

조회/검증 Task에는:

```yaml
check_mode: false
changed_when: false
```

를 적용했다.

그 결과:

```text
변경은 하지 않음
하지만 현재 상태는 실제 조회
```

가 가능해졌다.

## 검증

수정 후:

```text
5 nodes Ready
Calico 5/5 Running
CoreDNS 2/2 Running
PRE_CLUSTER_GATE=PASS
```

및 Check Mode PASS.

## 의미

Check Mode의 목적은 **“아무 Task도 실행하지 않는다”가 아니라 “상태를 바꾸지 않으면서 안전하게 예상 결과를 검증한다”**는 것이다.

이 원칙은 후속 NFS 자동화에서도 동일하게 중요해졌다.

## 관련 근거

- Umbrella Issue #7: https://github.com/seokpan/seokpan-infra/issues/7
- PR #52: https://github.com/seokpan/seokpan-infra/pull/52
- PR #56: https://github.com/seokpan/seokpan-infra/pull/56
