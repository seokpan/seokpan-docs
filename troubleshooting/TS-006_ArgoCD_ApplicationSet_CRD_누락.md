[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-006 — Argo CD ApplicationSet CRD 누락으로 Controller가 장시간 CrashLoopBackOff

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 발견·구성 보완, 2026-08-31 회귀검증에서 재확인 |
| **상태** | **실제 장애 · 해결** |
| **주 담당** | **최유준 — CI/CD 및 Argo CD 설치·초기 구성** |
| **영향 범위** | 정태훈(Kubernetes 플랫폼·GitOps 연동), GitOps를 사용하는 전체 작업 |

## 문제 개요

Kubernetes/Calico/API 자체는 정상인 상태에서 `argocd-applicationset-controller`가 약 **26시간 CrashLoopBackOff** 상태였던 것이 확인됐다.

Controller Log의 핵심은 다음과 같았다.

```text
no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

Cluster에는 다음 상태가 확인됐다.

- `applications.argoproj.io` 존재
- `appprojects.argoproj.io` 존재
- `applicationsets.argoproj.io` 부재

## 원인 분석

Argo CD 일부 구성요소는 설치되어 있었지만 ApplicationSet Controller가 요구하는 CRD가 초기 구성 자동화(Bootstrap)에 완전하게 포함되지 않았다. Controller는 필요한 API Kind를 찾지 못해 Cache Sync를 완료하지 못하고 반복 종료했다.

이 문제는 프로젝트 Python·ansible-core·kubernetes.core 버전 고정 작업 때문에 새로 발생한 회귀 문제가 아니라, 기존 Argo CD 초기 구성의 누락이 회귀검증 과정에서 드러난 것이었다.

## 조치

Argo CD Bootstrap Role을 다음과 같이 보완했다.

- Argo CD Release Manifest를 Server-Side Apply 방식으로 적용
- 대형 CRD까지 설치 범위에 포함
- `applicationsets.argoproj.io` 등 필수 CRD 존재 여부 확인
- `argocd-applicationset-controller`를 포함한 핵심 Controller의 `Available=True` 대기

## 검증

```text
Release Manifest 적용
→ 필수 CRD 존재 확인
→ ApplicationSet Controller Available
→ 최상위 Application(Root Application) Synced / Healthy
```

장애 원인이었던 ApplicationSet CRD 누락이 제거되고 Controller가 정상 기동하는 것을 확인했다.

## 담당 역할 및 영향

- **최유준:** Argo CD 설치 및 초기 구성 자동화 제공 영역 수정
- **정태훈:** Kubernetes 회귀검증에서 장애를 확인하고 GitOps 연동 상태 재검증
- **GitOps 사용 영역:** 하위 Application을 사용하기 전에 Argo CD 제어 구성요소가 정상인지 확인

## 관련 근거

- Argo CD Bootstrap Issue #23: https://github.com/seokpan/seokpan-infra/issues/23
- 장애 근거 Comment: https://github.com/seokpan/seokpan-infra/issues/23#issuecomment-5450542886
- Bootstrap 보완 PR #47: https://github.com/seokpan/seokpan-infra/pull/47
- Bootstrap 구현: https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/argocd_bootstrap/tasks/main.yml
