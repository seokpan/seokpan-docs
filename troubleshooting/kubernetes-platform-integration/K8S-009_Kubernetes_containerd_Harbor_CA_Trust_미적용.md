[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-024 — Kubernetes containerd가 Harbor 내부 CA를 신뢰하지 못해 Image Pull 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Kubernetes Control Plane 3대·Worker 2대의 containerd Harbor Image Pull 경로 |

## 문제 개요

`seokpan-app` PR #48에서 Backend/Frontend Container Image를 실제 Kubernetes에서 검증하던 중 Harbor에 Push된 이미지를 Pull하지 못했다.

```text
Harbor Image Pull
→ x509: certificate signed by unknown authority
→ ErrImagePull / ImagePullBackOff
```

Jenkins Rootless BuildKit을 통한 Build/Push는 이미 가능했지만 Kubernetes Node의 containerd는 BuildKit과 별도로 Harbor 인증서를 검증한다. 따라서 BuildKit이 Harbor 내부 CA(Certificate Authority, 인증 기관)를 신뢰하도록 설정해도 Kubernetes Node의 Image Pull에는 자동으로 적용되지 않았다.

## 사건 경계

이 사건은 TS-017과 같은 Harbor 내부 CA를 다루지만 인증서를 검증하는 주체와 수정 지점이 다르다.

```text
TS-017
Jenkins Rootless BuildKit의 Harbor Token 요청
→ SSL_CERT_FILE로 Harbor 내부 CA를 사용하도록 설정

TS-024
Kubernetes Node의 containerd
→ Node의 시스템 신뢰 저장소에 Harbor 내부 CA가 없음
→ Harbor Image Pull 실패
```

따라서 동일한 `x509: certificate signed by unknown authority` 증상이더라도 별도 사건으로 관리한다.

## 원인 분석

전체 Kubernetes Node를 확인했을 때 Harbor 내부 CA를 Node의 시스템 신뢰 저장소에 반영하는 자동화가 실제 환경에 적용되지 않은 상태였다.

당시 Infra Issue에서는 `/etc/containerd/certs.d/...` 방식도 검토했지만, 프로젝트의 실제 자동화는 공용 `ca_trust` Role을 사용해 Root CA 공개 인증서를 OS 신뢰 저장소에 배포하고 `update-ca-trust extract`로 갱신하는 구조다.

현재 `seokpan-infra/main`의 `ca_trust` Role은 다음 흐름을 사용한다.

```text
Controller의 internal CA 공개 인증서 확인
→ 대상 Node의 System Trust Anchor에 배포
→ update-ca-trust extract
→ openssl verify로 최종 신뢰 여부 확인
```

문제의 핵심은 Harbor 서버 인증서나 BuildKit 인증정보가 아니라 **Kubernetes Node의 containerd가 Harbor 내부 CA를 신뢰할 수 있도록 Node 측 설정이 적용되지 않은 것**이었다.

## 조치

Kubernetes Control Plane과 Worker Node에 공용 `ca_trust` 자동화를 적용해 Harbor 내부 CA를 시스템 신뢰 저장소에 반영했다.

이후 containerd가 갱신된 신뢰 저장소를 사용하도록 한 뒤 실제 Harbor Image Pull을 다시 검증했다.

이 작업은 Harbor CA Private Key나 Robot Account 인증정보를 Node에 배포하는 작업이 아니다. Node에는 공개 Root CA 인증서만 전달한다.

## 검증

### 1. 전체 5개 Kubernetes Node에서 실제 Harbor Pull

검증 대상:

- `cp-01`
- `cp-02`
- `cp-03`
- `worker-01`
- `worker-02`

검증 이미지는 다음 Harbor Image를 사용했다.

```text
harbor.seokpan.soldesk.store/seokpan/backend:verify-6
```

각 Node에 임시 Pod를 `nodeName`으로 고정하고 `imagePullPolicy: Always`로 실제 Registry Pull을 수행했다.

결과:

- 5개 Pod 모두 `Running / Ready=true`
- 5개 Node 모두 동일 Image Digest 확인
- 각 Pod Event에서 `Successfully pulled image` 확인
- `x509`, `ErrImagePull`, `ImagePullBackOff`, `Failed to pull image` 없음

따라서 Control Plane 3대와 Worker 2대 모두에서 Harbor Image Pull이 정상임을 확인했다.

### 2. 기존 Workload 영향 확인

Pull 검증 직후 다음 핵심 Workload가 계속 정상 동작하는지 확인했다.

- Argo CD 주요 Component
- Jenkins Controller
- Prometheus
- Alertmanager
- Grafana
- Loki
- Redis
- Node Exporter

검증 시점에서 위 Workload는 계속 `Running` 상태였고 이번 CA 설정 변경으로 새로 발생한 장애는 확인되지 않았다.

### 3. Ansible 재실행 확인

프로젝트 고정 실행환경에서 다음 자동화를 같은 조건으로 연속 두 번 실행했다.

```text
ansible-playbook playbooks/ca_trust.yml --limit 'control_planes:workers'
```

두 실행 모두 5개 Node에서:

```text
changed=0
failed=0
unreachable=0
Verify CA is now trusted → PASS
```

를 확인했다.

즉 현재 Kubernetes Node에는 필요한 CA 설정이 이미 적용되어 있으며, 같은 자동화를 다시 실행해도 불필요한 변경이 발생하지 않는다.

### 4. 임시 Resource 정리

검증용 임시 Pod 5개는 테스트 후 모두 삭제하고 잔존 Resource가 없음을 확인했다.

## Before → Change → After

```text
Before
Harbor Image는 Registry에 정상 Push됨
→ Kubernetes Node/containerd가 Harbor 내부 CA를 신뢰하지 못함
→ x509 unknown authority
→ Pod Image Pull 실패 / ImagePullBackOff

Change
Kubernetes 5개 Node에 ca_trust 자동화 적용
→ 내부 Root CA를 시스템 신뢰 저장소에 반영
→ containerd가 갱신된 CA 신뢰 설정을 사용

After
CP 3 + Worker 2 전체에서 실제 Harbor Pull PASS
→ 동일 Image Digest 확인
→ x509 / ErrImagePull / ImagePullBackOff 없음
→ 기존 핵심 Workload에 새 장애 없음
→ ca_trust 재실행 changed=0
```

## 관련 사건

- [TS-017 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패](TS-017_buildkit-harbor-ca-trust.md): 같은 Harbor 내부 CA를 다루지만 BuildKit에서 발생한 문제
- [TS-022 — Rootless BuildKit이 Dockerfile 첫 RUN에서 /proc 권한 문제로 실패](TS-022_BuildKit_Rootless_nested_RUN_seccomp_실행_실패.md): Image Build 실행환경 문제

TS-024는 **Kubernetes Node의 containerd가 Harbor Image를 Pull할 때 내부 CA를 신뢰하지 못한 문제**만 다룬다.

## 관련 근거

- Docs Issue #56: https://github.com/seokpan/seokpan-docs/issues/56
- Infra Issue #139: https://github.com/seokpan/seokpan-infra/issues/139
- 발견·후속 검증 App PR #48: https://github.com/seokpan/seokpan-app/pull/48
- 현재 `ca_trust` Role: https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/ca_trust/tasks/main.yml
