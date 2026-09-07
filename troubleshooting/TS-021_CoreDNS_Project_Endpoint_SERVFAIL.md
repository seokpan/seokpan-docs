[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-021 — Pod의 Project Endpoint가 CoreDNS에서 SERVFAIL로 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Kubernetes Pod의 프로젝트 FQDN 이름해석, Jenkins/BuildKit → Harbor Consumer 경로 |

## 문제 개요

Jenkins Rootless BuildKit Agent Pod에서 Harbor Push를 검증하던 중 다음 오류로 Pipeline이 중단됐다.

```text
dial tcp: lookup harbor.seokpan.soldesk.store on 10.96.0.10:53: server misbehaving
```

Node와 VM에서는 `harbor.seokpan.soldesk.store`가 정상적으로 `192.168.53.61`로 해석됐지만, Pod 내부에서는 같은 이름이 `SERVFAIL`을 반환했다.

Jenkins/BuildKit의 Plugin Version, PriorityClass, BuildKit Image와 설정을 다시 확인한 뒤에도 동일한 `SERVFAIL`이 재현되어 Application/BuildKit 설정이 아니라 Cluster DNS 경로의 문제로 범위를 좁혔다.

## 원인 분석

실제 이름해석 경로가 서로 달랐다.

```text
Node / VM
→ /etc/hosts
→ harbor.seokpan.soldesk.store = 192.168.53.61
→ 정상

Kubernetes Pod
→ Node의 /etc/hosts를 상속하지 않음
→ CoreDNS(10.96.0.10)
→ upstream DNS로 forward
→ upstream DNS에는 프로젝트 FQDN Record 없음
→ SERVFAIL
```

`harbor.seokpan.soldesk.store`는 당시 공용 DNS Server의 실제 Record가 아니라 Ansible `common_hosts`가 Node `/etc/hosts`에만 배포하는 Project Endpoint였다.

기존 05·06 설계에서는 이 차이를 이미 고려해 Host `/etc/hosts`와 Pod/CoreDNS Endpoint 제공을 분리하도록 정의했지만, 실제 구현에는 Host/Node 측 `common_hosts`만 반영되고 `coredns_records` 자동화가 누락돼 있었다.

따라서 이 문제는 새로운 DNS 구조가 필요한 장애라기보다 **기존 설계에 있던 Pod용 Project Endpoint publication 자동화가 구현에서 빠진 Gap**이었다.

## 해결안 비교

현재 1차 프로젝트 범위에서 실제 적용 가능한 방식을 비교했다.

| 후보 | 장점 | 단점 | 판정 |
|---|---|---|---|
| **CoreDNS `hosts` + Ansible `coredns_records`** | 기존 05·06 설계와 일치, 추가 서버 없음, Pod 전체 공통 적용, Ansible 재현·멱등성 검증 가능 | CoreDNS Cluster-wide 변경이므로 회귀검증 필요 | **채택** |
| 기존 내부 DNS에 Record 등록 + CoreDNS Forward | 일반적인 DNS 운영 구조, Node/Pod 공통 사용 가능 | 현재 관리 가능한 내부 DNS가 확인되지 않았고 외부 의존성 추가 | 장기 후보 |
| 별도 내부 DNS Server 구축 | 중앙관리와 확장성 우수 | VM·HA·Monitoring·Backup 등 범위 증가, MVP 대비 과도한 복잡도 | 제외 |
| CoreDNS `file` Plugin Zone 관리 | 여러 Record 관리에 유리 | 현재 소수 Endpoint에는 Zone/SOA 관리가 과함 | 제외 |

Pod별 `hostAliases`나 BuildKit에만 별도 hosts 성격의 설정을 넣는 방식은 특정 Consumer만 우회하고 같은 문제가 다른 Pod에서 반복될 수 있어 공식 해결안에서 제외했다.

## 조치

### 1. Project Endpoint Single Source of Truth 도입

공용 `project_endpoints` 정의를 만들고 Host와 Pod publication이 같은 값을 소비하도록 정리했다.

현재 CoreDNS에 publish하는 Endpoint:

```text
harbor.seokpan.soldesk.store  → 192.168.53.61
db.seokpan.soldesk.store      → 10.1.93.90
game.seokpan.soldesk.store    → 10.1.93.90
grafana.seokpan.soldesk.store → 10.1.93.90
```

`k8s-api`, Jenkins, Argo CD 외부 Endpoint처럼 당시 아직 SAN·Route·MVP 필요성이 확정되지 않은 항목은 이름이 존재한다는 이유만으로 미리 publish하지 않았다.

### 2. Host와 Pod publication 분리

```text
project_endpoints
├─ common_hosts
│  └─ Host /etc/hosts
└─ coredns_records
   └─ Kubernetes Pod DNS
```

이 구조로 동일 FQDN/IP를 `/etc/hosts`와 CoreDNS에 따로 하드코딩해서 Drift가 생기는 것을 피했다.

### 3. CoreDNS 관리 범위 제한

CoreDNS 전체 Corefile을 새 템플릿으로 덮어쓰지 않고, 프로젝트가 관리하는 `hosts` block만 제한적으로 관리하도록 구현했다.

안전장치로 다음을 포함했다.

- 현재 Corefile/marker 구조 확인
- 변경 전 Backup
- Render 결과 Preflight 검증
- 관리 block만 Patch
- 실제 변경 시에만 CoreDNS Rollout
- Kubernetes Service DNS 회귀검증
- Project Endpoint DNS 검증
- External DNS 회귀검증
- 실패 시 진단 Evidence 수집 및 이전 Corefile 복구

## 구현 과정에서 발생한 별도 장애

최초 적용 과정에서 CoreDNS 관리 block의 newline escaping 결함으로 Render된 Corefile 구조가 깨져 신규 CoreDNS Pod가 `CrashLoopBackOff`가 되는 별도 문제가 발생했다.

변경 전 자동 Backup으로 기존 Corefile을 복구한 뒤 line 구조 기반 Preflight와 rollback/recovery를 보완해 재적용했다.

이 문제는 **원래 BuildKit `SERVFAIL`의 Root Cause와는 다른 구현 결함**이므로 본 보고서의 해결 원인으로 합치지 않는다. 향후 신규 Troubleshooting 감사에서 독립 사례 가치와 근거를 별도로 판단한다.

## 검증

### CoreDNS / DNS 회귀

수정 후 확인 결과:

- Kubernetes Service DNS: PASS
- Redis Service DNS: PASS
- Harbor Project Endpoint DNS: PASS
- DB Project Endpoint DNS: PASS
- Game Project Endpoint DNS: PASS
- Grafana Project Endpoint DNS: PASS
- External DNS(`github.com`): PASS
- 신규 CoreDNS Pod 2개: Running / restart 0

### Ansible 멱등성

동일 Playbook 2회차 실행:

```text
CoreDNS change required = False
changed=0
failed=0
unreachable=0
```

Host `/etc/hosts` Project Endpoint 배포도 관리 대상 Host에서 재실행 `changed=0 / failed=0 / unreachable=0`을 확인했다.

### Pod → Harbor 경로

CoreDNS 수정 후 Pod에서:

```text
harbor.seokpan.soldesk.store
→ 192.168.53.61
```

이름해석이 정상화됐다.

이후 확인한 경로:

- Pod → Harbor DNS Resolve: PASS
- TCP 443: PASS
- TLS 1.3 Handshake: PASS
- Harbor Certificate Hostname 확인
- Harbor nginx HTTP 200: PASS

`seokpan-gitops#21`에서 기존 Jenkins Job을 다시 실행했을 때 **기존 DNS `SERVFAIL`은 더 이상 재현되지 않았고**, 다음 단계에서 `x509: certificate signed by unknown authority`가 새로 나타났다. 따라서 #99의 DNS 선행조건이 실제 BuildKit Consumer에서도 해소된 것을 확인했다.

CA Trust와 이후 Credential 인증 문제는 DNS와 다른 Root Cause이므로 별도 사례로 관리한다.

## Before → Change → After

```text
Before
Host/Node는 /etc/hosts로 Project Endpoint 해석 가능
→ Pod는 CoreDNS 사용
→ upstream DNS에 프로젝트 Record 없음
→ BuildKit Pod에서 Harbor FQDN SERVFAIL

Change
Project Endpoint Registry를 공용 기준으로 정의
→ common_hosts와 coredns_records가 동일 Endpoint를 소비
→ CoreDNS의 관리 hosts block만 Ansible로 제한 적용
→ Backup / Preflight / Regression / Rollback / Idempotency Gate 추가

After
Pod에서 Project Endpoint DNS 정상
→ Kubernetes Service DNS와 External DNS 회귀 없음
→ Ansible 2회차 changed=0
→ BuildKit 재실행에서 기존 SERVFAIL 미재발
→ 다음 독립 계층(CA Trust) 검증으로 진행 가능
```

## 관련 후속 사건

이 사건 해결 후 BuildKit → Harbor Consumer 검증은 다음 독립 문제로 이어졌다.

- [TS-017 — Jenkins Rootless BuildKit Harbor CA Trust 미적용 문제](TS-017_buildkit-harbor-ca-trust.md)
- [TS-020 — BuildKit Harbor Robot Credential 파일 계약 불일치](TS-020_BuildKit_Harbor_Credential_파일_계약_불일치.md)

세 사건은 같은 Pipeline 검증 흐름에서 순차적으로 드러났지만 Root Cause와 수정 지점이 달라 별도 Troubleshooting으로 관리한다.

## 관련 근거

- Docs Issue #33: https://github.com/seokpan/seokpan-docs/issues/33
- Infra Issue #99: https://github.com/seokpan/seokpan-infra/issues/99
- Infra PR #108: https://github.com/seokpan/seokpan-infra/pull/108
- GitOps Issue #21: https://github.com/seokpan/seokpan-gitops/issues/21
- GitOps PR #19: https://github.com/seokpan/seokpan-gitops/pull/19
- GitOps PR #24: https://github.com/seokpan/seokpan-gitops/pull/24
- Project Endpoint 현행 변경 이력: [PROJECT_CHANGES.md](../PROJECT_CHANGES.md)
