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

초기 Storage GitOps Manifest를 확인했을 때, NFS Subdir External Provisioner의 ClusterRole에 `nodes` 권한이 없는 것이 눈에 띄었다.

과거 Helm/Upstream 예시에서는 다음 권한이 포함되는 경우가 있어:

```yaml
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

최초 검토에서는 **누락 가능성**으로 보고 추가 여부를 확인하도록 요청했다.

즉 최초 판단은:

```text
없어서 문제일 수도 있다
```

였다.

## PR Pull Request 검토(Review)에서 판단이 뒤집힌 이유

PR #13에서 실제로 해당 권한이 추가된 상태를 다시 검토했다.

그러나 그 시점에는 질문이 달라졌다.

> "현재 프로젝트의 Provisioner가 실제로 Node 조회 권한을 필요로 한다는 근거가 있는가?"

확인 결과:

- 현재 최소 스토리지 배포 목적에는 Node 조회가 필요하다는 근거가 부족
- 불필요한 Cluster 권한은 최소권한 원칙에 반함
- 필요성이 증명되지 않은 권한을 "Upstream 예시에 있다"는 이유만으로 넣을 필요가 없음

따라서 최종적으로는 **초기 판단을 수정하고 `nodes` 권한을 제거**하도록 검토에서 요청했다.

## 최종 조치

`nodes get/list/watch` 제거.

이 사례는 "처음부터 정답을 알고 있었다"고 쓰지 않는다.

```text
1차 검토
→ Upstream 기준을 보고 누락 가능성 제기

PR 재검토
→ 실제 필요성 질문으로 전환

최종 판단
→ 근거 부족
→ 최소권한 원칙 적용
→ 제거
```

## 후속 검증

권한 제거 후에도 Storage Git에 선언한 목표 상태(Desired State)와 실제 실행 환경(Runtime)은 정상 동작했다.

확인된 Evidence:

- Argo CD Storage Application `Synced / Healthy`
- NFS Provisioner `1/1 Running`
- StorageClass `nfs-k8s` 존재
- Redis PVC `Bound`
- 검증용 PVC `Bound`
- Pod에서 실제 NFS Write 성공

따라서 "Node 권한을 제거했더니 Provisioner가 동작하지 않는다"는 문제는 발생하지 않았다.

## 이 사례의 핵심

이 사례의 가치는 권한 3줄 삭제 자체가 아니다.

**초기 검토 판단도 후속 Evidence에 따라 수정할 수 있어야 한다**는 점이다.

그리고 최종 권한은:

```text
필요하다고 증명된 권한
```

만 남겨야 한다.

## 관련 근거

- Storage Issue #11: https://github.com/seokpan/seokpan-gitops/issues/11
- Storage PR #13: https://github.com/seokpan/seokpan-gitops/pull/13
- NFS Provisioner 자동화 Issue #79: https://github.com/seokpan/seokpan-infra/issues/79
- NFS Provisioner PR #80: https://github.com/seokpan/seokpan-infra/pull/80
