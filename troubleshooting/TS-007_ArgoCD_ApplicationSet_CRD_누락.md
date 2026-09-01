[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-007 — Argo CD ApplicationSet CRD 누락으로 Controller가 장시간 CrashLoopBackOff

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 발견·구성 보완, 2026-08-31 회귀검증에서 기존 장애로 재확인 |
| **상태** | **실제 장애 · 해결** |
| **주 담당** | **최유준 — CI/CD 및 Argo CD 설치·초기 구성** |
| **영향 범위** | 정태훈(Kubernetes/GitOps Integration), GitOps를 사용하는 전체 작업 |

## 문제 개요

버전 고정(Version Lock) 이후 Argo CD/CRD 회귀검증을 위해 실제 `argocd` Namespace를 확인하던 중, 과거 설치 상태에서 `argocd-applicationset-controller`가 약 26시간 `CrashLoopBackOff`였던 Evidence가 다시 확인됐다.

직접 확인된 로그:

```text
failed to wait for applicationsets initial sync:
no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

CRD 확인 결과:

```text
applications.argoproj.io
appprojects.argoproj.io

applicationsets.argoproj.io
→ 없음
```

즉 전체 Argo CD가 설치되지 않은 것이 아니라, **ApplicationSet Controller가 필요로 하는 CRD만 누락된 불완전 설치 상태**였다.

## 원인 분석

초기 Argo CD 설치/검증은 개별 구성요소 위주로 진행됐고, 초기 구성 자동화(Bootstrap)이:

```text
Release Manifest에 포함된 필수 CRD 전체 존재
→ Controller 전체 Ready
```

를 하나의 완료조건으로 강제하지 않았다.

따라서:

```text
Server Running
Repo Server Running
Redis Running
Application Controller Running
```

만 보면 Argo CD가 정상처럼 보일 수 있었지만, 실제로는 ApplicationSet 기능이 죽어 있었다.

## 조치

후속 Argo CD Bootstrap에서는 수동으로 CRD 일부를 골라 설치하지 않고 **공식 Release Manifest 전체를 Server-Side Apply**하는 방향으로 보완됐다.

현재 `argocd_bootstrap`에는 다음 검증도 포함된다.

- `applications.argoproj.io`
- `appprojects.argoproj.io`
- `applicationsets.argoproj.io`
- 대형 CRD 여부 확인
- ApplicationSet Controller Deployment `Available`

## 검증 관점

이 사례의 중요한 점은 “26시간 뒤에 고쳤다”가 아니라, 당시 장애가 이후 **버전 고정 회귀검증에서 다시 추적되어 Root Cause와 현재 코드의 예방책까지 연결되었다**는 점이다.

```text
과거 장애 Evidence
→ Version Lock 회귀검증
→ applicationsets CRD 누락 확인
→ 현재 Bootstrap과 비교
→ Release Manifest SSA + CRD/Controller 검증이 재발 방지책임을 확인
```

## 역할 영향

- **최유준:** Argo CD 설치/Bootstrap Owner. Release Manifest와 필수 CRD·Controller Ready를 함께 보장해야 한다.
- **정태훈:** Kubernetes/GitOps Consumer. Root Application 및 하위 Application을 사용하므로 Argo의 부분 장애가 곧 GitOps 통합 장애가 된다.
- **GitOps 전체:** Server UI/API가 뜬다는 사실만으로 ApplicationSet 기능 정상까지 주장할 수 없게 됐다.

## 관련 근거

- Parent Version Lock Issue #38: https://github.com/seokpan/seokpan-infra/issues/38
- Argo 초기 Issue #23: https://github.com/seokpan/seokpan-infra/issues/23
- Version Lock PR #46의 Argo/CRD 회귀검증 코멘트: https://github.com/seokpan/seokpan-infra/pull/46
- 현재 Bootstrap Role: `ansible/roles/argocd_bootstrap/`

## 후속 Bootstrap 검증 Gap

Version Lock 회귀검증에서 실제 ApplicationSet CRD 누락 장애는 재발하지 않았지만, Bootstrap의 Ready 검증 범위 자체는 한 차례 더 보강 여지가 확인됐다.

현재 Bootstrap 검증은 다음 Deployment의 Ready를 대기한다.

- `argocd-server`
- `argocd-repo-server`
- `argocd-redis`
- `argocd-dex-server`
- `argocd-applicationset-controller`
- `argocd-notifications-controller`

반면 `argocd-application-controller`는 StatefulSet이며, 해당 Ready 대기가 동일 검증 목록에는 포함되지 않았다. 실제 Runtime에서는 `argocd-application-controller-0`이 `Running`이고 전체 Argo CD 동작도 PASS였으므로 이것을 새 장애로 분류하지는 않는다.

다만 **"Argo CD Bootstrap 완료"를 강하게 주장하려면 Application Controller StatefulSet도 명시적으로 Ready 상태를 기다리는 편이 더 정확하다.** 이 항목은 후속 Validation hardening 대상이다.
