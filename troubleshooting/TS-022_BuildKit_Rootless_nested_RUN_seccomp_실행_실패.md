[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-022 — Rootless BuildKit의 Dockerfile RUN 단계가 seccomp·process sandbox 제약으로 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins Rootless BuildKit Agent, Backend/Frontend Dockerfile Build 경로 |

## 문제 개요

`seokpan-app` PR #48의 Backend/Frontend Dockerfile을 실제 Jenkins Rootless BuildKit Agent에서 검증하던 중, Image Build가 Dockerfile의 첫 `RUN` 단계에서 진행되지 못했다.

문제는 Application Source나 Dockerfile 명령 자체가 아니라 Rootless BuildKit이 Kubernetes Pod 안에서 `RUN`(ExecOp)을 실행할 때 필요한 nested 실행 환경을 현재 Pod 보안 설정이 허용하지 못한 것이었다.

검증 흐름에서 확인한 대표 증상은 `/proc mount ... operation not permitted` 계열 오류였다.

## 원인 분석

초기에는 BuildKit Container에 다음 설정만 추가하면 해결될 것으로 판단했다.

```yaml
securityContext:
  seccompProfile:
    type: Unconfined
```

하지만 리뷰에서 BuildKit의 Rootless Kubernetes 실행 패턴을 다시 대조한 결과 이 조치만으로는 불완전했다.

두 설정은 역할이 다르다.

```text
seccompProfile: Unconfined
→ Rootless BuildKit이 필요한 unshare/mount 계열 syscall 차단 해제

BUILDKITD_FLAGS=--oci-worker-no-process-sandbox
→ 각 RUN(ExecOp)마다 별도 process sandbox를 만들면서 /proc를 다시 구성하는 경로를 사용하지 않음
```

따라서 Kubernetes에서 Rootless BuildKit의 nested `RUN`을 안정적으로 실행하려면 현재 프로젝트 구조에서는 두 조건을 함께 적용해야 했다.

## 조치

`seokpan-gitops/cicd/jenkins-jcasc-configmap.yaml`의 `buildkit-rootless` PodTemplate을 다음처럼 보완했다.

### BuildKit Container에 한정된 seccomp 완화

```yaml
containers:
  - name: buildkit
    securityContext:
      seccompProfile:
        type: Unconfined
```

### Rootless BuildKit process sandbox 비활성화

```text
BUILDKITD_FLAGS=--oci-worker-no-process-sandbox
```

이 설정은 Jenkins Controller나 다른 Container가 아니라 BuildKit Agent Container에만 적용했다.

## 중간 판단과 리뷰 보완

이 사건에서 중요한 점은 `seccompProfile: Unconfined`만 넣은 최초 수정안을 해결 완료로 간주하지 않았다는 것이다.

리뷰에서 다음 두 조건을 Merge 전 Gate로 다시 요구했다.

1. `--oci-worker-no-process-sandbox`까지 포함해 Rootless Kubernetes 실행 패턴과 맞출 것
2. Server-side Dry-run만으로 끝내지 않고 실제 `verify-pr48-dockerfile` Job을 재실행할 것

리뷰 반영 후 최종 JCasC에는 두 설정이 모두 포함됐다.

## 검증

### 실제 Pod 설정 확인

실제 Agent Pod YAML에서 다음이 확인됐다.

```text
securityContext:
  privileged: false
  runAsUser: 1000
  seccompProfile:
    type: Unconfined

BUILDKITD_FLAGS=--oci-worker-no-process-sandbox
```

즉 Cluster 전체나 Jenkins Controller의 보안 설정을 해제한 것이 아니라 Rootless BuildKit Container에 필요한 범위만 적용했다.

### 동일 Job 재실행

기존 장애가 발생했던 `verify-pr48-dockerfile` Job을 다시 실행한 결과, 이전에 막혔던 첫 `RUN` 단계가 정상 진행됐다.

대표적으로 Runtime Image의 사용자 생성 단계가:

```text
RUN groupadd ...
→ DONE
```

으로 통과했다.

이후 동일 Rootless BuildKit 경로에서 Backend/Frontend Build와 Push 검증이 계속 진행돼 기존 nested `RUN` blocker가 해소된 것을 확인했다.

## 보안 Trade-off

`--oci-worker-no-process-sandbox`는 Rootless BuildKit의 process isolation을 일부 완화한다. 프로젝트 검토에서는 이 설정을 BuildKit Agent Container 하나에만 제한하고, 신뢰된 CI Pipeline을 실행하는 현재 1차 프로젝트 범위에서 해당 Trade-off를 수용했다.

따라서 이 사례를 일반 Application Container에 `Unconfined`를 적용해야 한다는 기준으로 확장하지 않는다.

## Before → Change → After

```text
Before
Rootless BuildKit Agent
→ Dockerfile 첫 RUN
→ nested 실행을 위한 /proc mount / process sandbox 경로 차단
→ operation not permitted
→ Image Build 중단

Change
BuildKit Container에 seccompProfile: Unconfined
+ BUILDKITD_FLAGS=--oci-worker-no-process-sandbox
→ 실제 Agent Pod 반영 확인

After
동일 verify-pr48-dockerfile Job 재실행
→ 기존 첫 RUN 단계 통과
→ 후속 Backend/Frontend Build 검증 진행 가능
```

## 관련 사건

같은 Jenkins/BuildKit → Harbor 검증 흐름에서 다음 사건들이 앞서 발생했지만 Root Cause는 각각 다르다.

- [TS-021 — Pod의 Project Endpoint가 CoreDNS에서 SERVFAIL로 실패](TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md)
- [TS-017 — Jenkins Rootless BuildKit Harbor CA Trust 미적용 문제](TS-017_buildkit-harbor-ca-trust.md)
- [TS-020 — BuildKit Harbor Robot Credential 파일 계약 불일치](TS-020_BuildKit_Harbor_Credential_파일_계약_불일치.md)

TS-022는 DNS·TLS Trust·Credential이 아니라 **Rootless BuildKit의 Dockerfile ExecOp 실행 환경** 문제를 다룬다.

## 관련 근거

- Docs Issue #55: https://github.com/seokpan/seokpan-docs/issues/55
- GitOps Issue #33: https://github.com/seokpan/seokpan-gitops/issues/33
- GitOps PR #34: https://github.com/seokpan/seokpan-gitops/pull/34
- 발견 및 후속 Build 검증 App PR #48: https://github.com/seokpan/seokpan-app/pull/48
- 현재 JCasC: https://github.com/seokpan/seokpan-gitops/blob/main/cicd/jenkins-jcasc-configmap.yaml
