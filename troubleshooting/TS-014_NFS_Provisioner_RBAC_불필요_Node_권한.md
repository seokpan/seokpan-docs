[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-014 — NFS Provisioner RBAC 검토에서 초기 판단을 재검증해 불필요한 Node 조회 권한을 제거

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **예방형 · 해결** |
| **주 담당** | **김상희 — 스토리지 / 정태훈 — Kubernetes 연동 및 Pull Request 검증** |
| **영향 범위** | Kubernetes Cluster 권한 경계 전체 |

## 최초 검토에서의 판단

Storage Manifest 5개를 처음 대조할 때 `rbac.yaml`에 `nodes get/list/watch`가 빠져 있다는 점을 보고, **Upstream 공식 RBAC에 포함돼 있다고 판단하여 추가 확인/보완 대상으로 제시했다.** 이 판단은 `seokpan-infra#44`의 초기 코멘트에 그대로 남아 있다.

즉 처음부터 “과권한을 발견했다”가 아니다. 오히려 초기에는 **권한을 추가해야 할 가능성**을 제기했다.

## Pull Request 검토(Review)에서 판단이 뒤집힌 이유

실제 GitOps PR #13을 검토하면서 현재 사용하는 NFS Subdir External Provisioner v4.0.2의 동작과 필요한 권한을 다시 대조했다. 그 결과 `nodes get/list/watch`를 프로젝트의 상시 ClusterRole에 넣어야 한다는 근거가 충분하지 않다고 판단했다.

Cluster-scoped 권한은 Argo CD의 목표 상태(Desired State)에 들어가면 지속적으로 유지되므로, “Upstream에 있었던 것으로 보인다”는 이유만으로 권한을 추가하는 것은 최소권한 원칙에 맞지 않았다.

## 최종 조치

- `nodes get/list/watch` 권한 제거
- Provisioner가 실제 요구하는 RBAC만 유지
- StorageClass `nfs-k8s`, Provisioner `v4.0.2`, `storage-infra` 경계 재검토
- 검증용(Validation) PVC/Pod는 상시 목표 상태(Desired State)에서 분리
- `archiveOnDelete`는 실제 실행 환경(Runtime) 데이터 보존 관점으로 별도 확정

## 후속 검증

검토 수정 후 Storage 실제 실행 환경 단계에서 다음 경로를 확인했다.

```text
Argo CD Storage Desired State
→ NFS Provisioner
→ StorageClass nfs-k8s
→ PVC
→ Dynamic PV Bound
→ NFS 하위 디렉터리
→ Pod Mount / Write / Read
```

즉 불필요한 Node 조회 권한을 제거한 상태에서도 Storage 기능 수행에 필요한 경로가 유지됐다.

## 이 사례의 핵심

이 사례는 단순 RBAC 수정이 아니라 **초기 기술 판단도 근거가 부족하면 검토에서 뒤집어야 한다는 사례**다.

```text
초기 판단
"Upstream 기준으로 nodes 권한 추가 필요 가능"
        ↓
실제 Version/동작/권한 재검토
        ↓
필수 근거 부족
        ↓
권한 추가 대신 제거
        ↓
실제 실행 환경 기능 검증
```

## 관련 근거

- 초기 검토 기록: https://github.com/seokpan/seokpan-infra/issues/44#issuecomment-5450379547
- GitOps Issue #10: https://github.com/seokpan/seokpan-gitops/issues/10
- GitOps PR #13: https://github.com/seokpan/seokpan-gitops/pull/13
- PR #13 검토: https://github.com/seokpan/seokpan-gitops/pull/13#pullrequestreview-5067388224
- 후속 Storage 근거: https://github.com/seokpan/seokpan-infra/issues/44