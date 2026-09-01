[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-011 — Kubernetes 통합 Playbook의 Ansible Check Mode에서 검증 명령이 Skip되어 Assert가 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 이유빈(Ansible 통합), Kubernetes 자동화를 실행·검증하는 팀원 전체 |

## 문제 개요

개별 Kubernetes 자동화를 `kubernetes_cluster.yml` 하나의 흐름으로 통합했다.

```text
cp-01 초기 구성 자동화(Bootstrap)
→ cp-02/cp-03 Join
→ Worker Join
→ Calico
→ Gateway Add-on
```

## 증상

Ansible Check Mode에서 실제 변경 Task를 실행하지 않는 것은 정상 동작이지만, 기존 코드에서는 상태를 확인하기 위한 `command` 계열 검증까지 함께 Skip됐다. 그 결과 후속 `assert`가 참조할 변수 자체가 생성되지 않아 Check Mode가 실패했다.

즉 Kubernetes 실행 상태의 문제가 아니라 **변경 작업과 read-only 검증을 Check Mode에서 같은 방식으로 취급한 자동화 설계 문제**였다.

## 조치

- kubeadm config 검증, API `/readyz`, Node 등록 확인을 Check Mode에서도 실행
- Worker Node 등록 확인도 read-only 검증으로 분리
- Gateway `kubectl diff`, `wait`, `rollout`, `get`은 Check Mode에서도 실행
- 실제 `kubectl apply`, Join Token 생성, Join 실행처럼 상태를 바꾸는 작업만 Check Mode에서 Skip

## 검증

- Syntax Check PASS
- 전체 Check Mode PASS
- 실제 실행 PASS
- CP3 + Worker2 `changed=0`, `failed=0`, `unreachable=0`
- Calico/Gateway 상태 정상

## 핵심 원칙

```text
Check Mode = 모든 명령을 실행하지 않는 모드  X
Check Mode = 상태 변경은 막되, 변경 판단에 필요한 read-only 검증은 수행  O
```

이 수정으로 멱등성을 검증하기 위한 Check Mode가 검증 로직 자체 때문에 실패하는 문제를 제거했다.

## 관련 근거

- Parent Issue #7: https://github.com/seokpan/seokpan-infra/issues/7
- PR #67: https://github.com/seokpan/seokpan-infra/pull/67
- Merge Commit `619708717c7c...`
