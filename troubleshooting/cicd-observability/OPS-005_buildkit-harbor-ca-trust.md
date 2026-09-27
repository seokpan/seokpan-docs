[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-017 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 ~ 2026-09-03 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins Rootless BuildKit Agent Pod의 Harbor TLS 인증 경로, CI/CD Build → Push 단계 |

## 문제 개요

`ci-verify-buildkit-push` 검증 Job에서 Harbor Push를 시도할 때 다음 오류가 반복됐다.

```text
failed to fetch anonymous token: Get "https://harbor.seokpan.soldesk.store/service/token?...":
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

`harbor-ca-cert` ConfigMap에는 Harbor 내부 CA(Certificate Authority, 인증 기관) 인증서가 정상적으로 마운트되어 있었고, `curl --cacert`로 직접 확인하면 TLS 검증도 통과했다. 그러나 `buildctl`을 통한 Push에서만 CA 신뢰 오류가 계속 발생했다.

## 원인 분석

### 1차 가설 — `buildkitd.toml` 적용 문제

JCasC(Jenkins Configuration as Code) PodTemplate의 BuildKit Container에 `--config=/home/user/.config/buildkit/buildkitd.toml`을 지정하면 해결될 것으로 보았다.

이 과정에서 `containerTemplate.args`를 YAML List로 작성해 JCasC `ConfiguratorException: Item isn't a Scalar`가 발생했고 Jenkins Controller가 CrashLoopBackOff에 빠졌다. `args`를 Scalar 문자열로 수정한 뒤 Controller는 정상화됐지만 Harbor TLS 오류는 그대로 재현됐다.

### 2차 가설 — 지속형 buildkitd와 daemonless buildkitd의 충돌

Container 시작 시 계속 실행되는 buildkitd와 Jenkinsfile의 `buildctl-daemonless.sh`가 작업마다 임시 buildkitd를 실행하는 구조가 충돌하는지 확인했다.

`command: "cat"` + `ttyEnabled: true`로 Container를 대기 상태로 두고 지속형 daemon을 제거했지만 동일한 TLS 오류가 재현돼 이 가설도 기각됐다.

### 최종 확인된 원인

`buildctl-daemonless.sh`로 실행된 BuildKit이 Harbor Token을 요청할 때는 `buildkitd.toml`의 Registry CA 설정을 사용하지 않았다.

실패 지점의 Debug Stack에서 `session/auth/authprovider.(*authProvider).FetchToken` 경로를 확인했고, Token 요청을 수행하는 BuildKit Client Process가 Harbor 내부 CA를 신뢰하도록 `SSL_CERT_FILE`을 지정해야 했다.

당시 동작은 다음과 같았다.

```text
Harbor CA ConfigMap 마운트
→ buildkitd.toml에 Registry CA 설정
→ buildkitd가 Registry와 직접 통신할 때는 사용 가능

buildctl의 Harbor Token 요청
→ buildkitd.toml의 CA 설정을 사용하지 않음
→ OS 기본 CA 기준으로 Harbor 내부 CA를 신뢰하지 못함
→ x509 unknown authority
```

## 조치

- JCasC `buildkit` Container 환경변수에 `SSL_CERT_FILE=/etc/buildkit/certs/ca.crt` 추가
- 불필요해진 `BUILDKITD_CONFIG`와 Container `args(--config=...)` 제거
- `command: "cat"` + `ttyEnabled: true`로 Container를 대기 상태로 유지
- 실제 buildkitd는 Jenkinsfile의 `buildctl-daemonless.sh` 호출 시점에만 기동
- Jenkinsfile Build & Push 단계에서도 동일한 `SSL_CERT_FILE`을 사용하도록 통일

## 검증

- Pod 내부 Process와 `/proc/<pid>/cmdline`을 통해 실제 buildkitd 실행 구조 확인
- Foreground Debug 재현에서 `SSL_CERT_FILE` 적용 전 `x509: certificate signed by unknown authority` 확인
- `SSL_CERT_FILE` 적용 후 기존 x509 오류가 사라지고 다음 인증 단계의 `401 Unauthorized`까지 진행되는 것을 확인해 CA 신뢰 문제가 해소됐음을 확인
- 후속 인증 파일 문제까지 별도로 수정한 뒤 실제 `ci-verify-buildkit-push`를 재실행했을 때 TLS 오류 재발 없이 Harbor Push와 Digest 등록까지 진행됨을 확인

## 관련 후속 사건

CA 신뢰 문제를 해결한 뒤 Harbor 인증 단계에서 `401 Unauthorized`가 새로 확인됐다.

해당 문제는 Harbor Robot Account 인증정보가 저장된 Kubernetes Secret의 파일명 `.dockerconfigjson`과 BuildKit이 찾는 `config.json` 파일명이 달랐던 별도 원인이었다. 같은 BuildKit → Harbor 검증 과정에서 연속으로 발견됐지만 수정 위치와 검증 단계가 다르므로 [TS-020 — BuildKit이 Harbor Robot 인증 파일을 찾지 못해 401 Unauthorized 발생](TS-020_BuildKit_Harbor_Credential_파일_계약_불일치.md)에서 별도 사례로 관리한다.

## 관련 독립 CA 사건

이후 Kubernetes에서 Harbor Image를 실제 Pull하는 단계에서는 각 Node의 containerd가 같은 내부 CA를 신뢰하지 못하는 별도 문제가 확인됐다.

BuildKit의 Harbor Token 요청과 Kubernetes Node의 containerd Image Pull은 인증서를 검증하는 주체와 수정 위치가 다르므로 [TS-024 — Kubernetes containerd가 Harbor 내부 CA를 신뢰하지 못해 Image Pull 실패](TS-024_Kubernetes_containerd_Harbor_CA_Trust_미적용.md)에서 독립 사례로 관리한다.

## Before → After

```text
Before
Harbor 내부 CA가 Pod에 마운트되어 있어도
buildctl의 Token 요청이 해당 CA를 사용하지 못함
→ x509 unknown authority

After
SSL_CERT_FILE로 BuildKit Client가 사용할 CA 파일을 지정
→ x509 오류 해소
→ Harbor 인증 단계까지 진행
```

## 관련 근거

- GitOps PR #19 — BuildKit Agent 안정성·재현성 및 Harbor CA 설정 초기 보완: https://github.com/seokpan/seokpan-gitops/pull/19
- GitOps PR #24 — Root 연결 및 최종 `SSL_CERT_FILE` 방식 반영: https://github.com/seokpan/seokpan-gitops/pull/24
- GitOps Issue #21 — CoreDNS 해결 이후 CA/인증 재검증 흐름: https://github.com/seokpan/seokpan-gitops/issues/21
- 후속 인증정보 Issue #126: https://github.com/seokpan/seokpan-infra/issues/126
- 후속 인증정보 PR #127: https://github.com/seokpan/seokpan-infra/pull/127
