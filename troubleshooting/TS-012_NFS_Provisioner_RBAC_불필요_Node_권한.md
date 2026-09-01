[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-012 — NFS Provisioner RBAC 검토에서 초기 판단을 재검증해 불필요한 Node 조회 권한을 제거

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **예방형 · 해결** |
| **주 담당** | **김상희 — 스토리지 / 정태훈 — Kubernetes 연동 및 Pull Request 검증** |
| **영향 범위** | Kubernetes Cluster 권한 경계 전체 |

## 최초 검토에서의 판단

Storage Manifest 5개를 대조할 때 `rbac.yaml`에 `nodes get/list/watch`가 빠져 있다는 점을 보고, Upstream 공식 RBAC에 포함된 권한일 가능성을 근거로 추가 확인 대상으로 제시했다.

즉 처음부터 과권한을 발견한 것이 아니라, 최초에는 **Node 조회 권한을 추가해야 할 가능성**을 검토했다.

## Pull Request 검토에서 판단이 바뀐 이유

GitOps PR #13을 검토하면서 프로젝트가 사용하는 NFS Subdir External Provisioner v4.0.2의 동작과 실제 필요한 권한을 다시 대조했다. 그 결과 `nodes get/list/watch`를 프로젝트 상시 ClusterRole에 넣어야 한다는 근거가 충분하지 않다고 판단했다.

Cluster-scoped 권한은 Argo CD의 목표 상태(Desired State)에 포함되면 계속 유지된다. 따라서 단순히 Upstream 예시에 포함됐다고 추정되는 이유만으로 권한을 추가하는 것은 최소권한 원칙에 맞지 않았다.

## 최종 조치

- `nodes get/list/watch` 권한 제거
- Provisioner가 실제 요구하는 RBAC만 유지
- StorageClass `nfs-k8s`, Provisioner `v4.0.2`, `storage-infra` 경계 재검토
- 검증용 PVC/Pod는 상시 목표 상태에서 분리
- `archiveOnDelete`는 실제 데이터 보존 관점으로 별도 확정

## 후속 검증

권한을 제거한 상태에서 다음 Storage 경로를 실제로 확인했다.

```text
Argo CD Storage Desired State
→ NFS Provisioner
→ StorageClass nfs-k8s
→ PVC
→ Dynamic PV Bound
→ NFS 하위 디렉터리
→ Pod Mount / Write / Read
```

불필요한 Node 조회 권한을 제거한 뒤에도 동적 Provisioning과 Pod 읽기·쓰기가 정상 동작했다.

## 이 사례의 핵심

```text
초기 판단
Node 조회 권한 추가 필요 가능
→ 실제 Version/동작/권한 재검토
→ 필수 근거 부족 확인
→ 권한 추가 대신 제거
→ 실제 Storage 기능 정상 검증
```

초기 기술 판단도 실제 근거가 부족하면 검토 과정에서 수정해야 한다는 사례다.

## 관련 근거

- 초기 검토 기록: https://github.com/seokpan/seokpan-infra/issues/44#issuecomment-5450379547
- GitOps Issue #10: https://github.com/seokpan/seokpan-gitops/issues/10
- GitOps PR #13: https://github.com/seokpan/seokpan-gitops/pull/13
- PR #13 Review: https://github.com/seokpan/seokpan-gitops/pull/13#pullrequestreview-5067388224
- 후속 Storage 검증: https://github.com/seokpan/seokpan-infra/issues/44
