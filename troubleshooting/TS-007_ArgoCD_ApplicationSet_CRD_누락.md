[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-007 — Argo CD ApplicationSet CRD 누락으로 Controller가 장시간 CrashLoopBackOff

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-28 발견·구성 보완, 2026-08-31 회귀검증에서 기존 장애로 재확인 |
| **상태** | **실제 장애 · 해결** |
| **주 담당** | **최유준 — CI/CD 및 Argo CD 설치·초기 구성** |
| **영향 역할** | 정태훈(Kubernetes/GitOps Integration), GitOps를 사용하는 전체 작업 |
| **핵심 범주** | Argo CD / CRD / Bootstrap / Runtime Dependency |
| **Evidence** | Runtime CrashLoopBackOff + Controller Log + CRD 존재 여부 + 현재 Bootstrap 검증 코드 |

## 증상

Version Lock 회귀검증 과정에서 Kubernetes/Calico/API 자체는 정상이었지만 `argocd-applicationset-controller`가 약 **26시간 CrashLoopBackOff** 상태였던 것이 확인됐다.

Controller Log의 핵심은 다음과 같았다.

```text
no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

그리고 Cluster에는:

- `applications.argoproj.io` 존재
- `appprojects.argoproj.io` 존재
- **`applicationsets.argoproj.io` 부재**

상태였다.

## 원인

Argo CD Deployment 일부는 존재했지만 ApplicationSet Controller가 요구하는 CRD가 완전하게 Bootstrap되지 않았다. Controller는 필요한 API Kind를 발견하지 못해 Cache Sync를 완료하지 못하고 반복 종료했다.

중요하게도 이 문제는 Project Python/ansible-core/kubernetes.core Version Lock을 도입해서 새로 발생한 Regression이 아니었다. **기존 Runtime에 이미 존재하던 Argo CD Bootstrap 불완전 상태**가 회귀검증 과정에서 드러난 것이다.

## 조치

Argo CD Bootstrap Role을 대형 CRD까지 포함해 Release Manifest를 Server-Side Apply하는 구조로 정리했다. 또한 대형 CRD가 실제 생성됐는지 별도 검증하도록 했다.

현재 main의 Bootstrap은:

- Argo CD Release Manifest Server-Side Apply
- `argocd_large_crds` 존재 확인
- `applicationsets.argoproj.io` 같은 대형 CRD 미존재 시 실패
- `argocd-applicationset-controller` 포함 핵심 Deployment `Available=True` 대기

를 수행한다.

## 검증 관점

```text
잘못된 완료 판단
Argo CD 일부 Pod 존재
→ 설치 완료라고 판단

보강된 완료 판단
Release Manifest Apply
→ 필수 CRD 존재 확인
→ ApplicationSet Controller Available
→ Root Application Synced/Healthy
```

## 역할 영향

- **최유준:** Argo CD 설치/Bootstrap 제공 영역 수정
- **정태훈:** Version Lock/Kubernetes 회귀검증 중 장애 분리 및 GitOps 통합 소비
- **전체 GitOps 사용 영역:** Root/Child Application 운영 전 Argo CD 제어면 완전성 확보

## 근거

- Argo CD Bootstrap Issue #23: https://github.com/seokpan/seokpan-infra/issues/23
- 장애 Evidence Comment: https://github.com/seokpan/seokpan-infra/issues/23#issuecomment-5450542886
- Bootstrap 보완 PR #47: https://github.com/seokpan/seokpan-infra/pull/47
- 현재 Bootstrap 구현: https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/argocd_bootstrap/tasks/main.yml

---


## 후속 Bootstrap 검증 Gap

Version Lock 환경에서 Argo CD/CRD를 다시 검증한 뒤, `argocd_bootstrap`의 핵심 Ready 대기 목록에 `argocd-application-controller` StatefulSet 확인이 빠져 있다는 지적도 남았다.

이 항목은 당시 ApplicationSet CrashLoopBackOff의 직접 원인은 아니고, 실제 별도 장애가 재현된 것도 아니다. 다만 Argo CD 설치 성공을 판단할 때 일부 Controller만 Ready를 확인하면 전체 제어면 정상성을 과대평가할 수 있으므로 **Bootstrap Validation hardening 항목**으로 이 사례 안에 보존한다. 독립 TS로는 분리하지 않는다.
