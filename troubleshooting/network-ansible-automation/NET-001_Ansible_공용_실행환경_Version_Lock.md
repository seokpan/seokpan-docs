[← 전체 트러블슈팅](../README.md) · [네트워크·Ansible 자동화 목차](README.md)

# NET-001 — Ansible 공용 실행환경이 버전 고정 없이 시스템 기본 환경에 의존

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 ~ 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **이유빈 — 네트워크 및 공통 인프라·Ansible 통합** |
| **영향 범위** | 정태훈, 김상희, 최유준 전원 |

## 문제 개요

Ansible Controller에는 이미 다음 실행환경이 설치되어 있었다.

- System Python `3.9.25`
- System ansible-core `2.14.18`

하지만 이 값들은 프로젝트에서 선택·검증한 버전 조합이 아니라 기존 시스템 환경에 설치되어 있던 버전이었다. 담당자별 Python, `kubernetes.core`, Kubernetes Python Client 버전도 달라질 수 있어 같은 Playbook이 실행자에 따라 다른 결과를 낼 위험이 있었다.

## 목표

목표는 단순히 최신 버전으로 올리는 것이 아니라 **GitHub 저장소의 정의만으로 동일한 프로젝트용 Ansible 실행환경을 다시 만들 수 있게 하는 것**이었다.

## 최종 버전 구성표(Version Matrix)

| Component | 프로젝트 기준 |
|---|---:|
| Python | `3.12.13` |
| ansible-core | `2.20.8` |
| Python kubernetes client | `36.0.3` |
| kubernetes.core | `6.5.0` |
| System Python | `3.9.25` 유지 |
| System ansible-core | `2.14.18` 유지 |

시스템 기본 환경은 Rollback 용도로 보존하고, 프로젝트 `.venv`와 Collection 경로를 분리했다.

## 추가로 발견한 검증 결함

초기 Bootstrap은 Python `3.12.x`까지만 확인해 `3.12.12`, `3.12.14`도 통과할 수 있었다. 공식 기준이 `3.12.13` Exact Lock이므로 patch-level까지 검사하도록 수정했다.

검증 결과:

- `3.12.13` → PASS
- 테스트 Wrapper로 `3.12.12` 조건 → FAIL, Exit 1

또한 `ansible-core 2.20.8`에서 `INJECT_FACTS_AS_VARS` 관련 경고가 확인되어, 우선 확인된 `kubeadm_control_plane` Template의 top-level fact 참조를 `ansible_facts[...]` 방식으로 변경했다.

## 검증

Version Lock PR과 Exact Python 검증 보완이 `main`에 반영된 뒤 다음을 확인했다.

- 프로젝트 `.venv`에서 Python `3.12.13`
- ansible-core `2.20.8`
- Python kubernetes client `36.0.3`
- kubernetes.core `6.5.0`
- 기존 System Python/Ansible은 변경하지 않음
- 잘못된 Python patch version을 실제 실패 조건으로 차단

따라서 프로젝트 공용 Ansible 실행환경은 시스템 기본 설치 상태에 의존하지 않고, 프로젝트가 정한 버전 조합을 별도 환경에서 재현하도록 정리됐다.

## 담당 역할 및 영향

- **이유빈:** 공용 실행환경 제공 및 Ansible 통합
- **정태훈:** Kubernetes/Calico/kubernetes.core 회귀검증
- **최유준:** CI/CD·관측·Argo CD 관련 검증
- **김상희:** MariaDB/MaxScale/NFS 관련 검증

## 후속 운영 기준

위 Version Matrix와 발생 당시 원인·조치·검증 결과는 그대로 보존한다. 이후 공용 실행환경에 추가되거나 정합화된 내용은 원 사건을 현재 상태로 덮어쓰지 않고 다음 후속 운영 기준으로 연결한다.

### 1. `ansible.mariadb 6.0.2` Collection 추가

MariaDB 자동화에서 계정 관련 모듈 사용이 확정되면서 Project Collection에 `ansible.mariadb 6.0.2`를 추가했다.

현재 Project Collection 기준은 다음과 같다.

```text
kubernetes.core 6.5.0
ansible.mariadb 6.0.2
```

`requirements.yml`을 기준으로 빈 Project Collection 환경에서도 동일 버전을 다시 설치할 수 있는지 확인했고, Bootstrap은 필요한 Collection의 누락과 버전 불일치를 확인하도록 유지한다.

이 후속 결정으로 최초 Version Matrix의 Python·ansible-core 기준을 변경하지 않는다.

### 2. deprecated Fact 참조 후속 정합화

NET-001의 최초 기록에서는 `INJECT_FACTS_AS_VARS` 경고가 확인된 위치를 우선 수정한 당시 범위를 보존한다.

이후 Kubernetes 담당 범위에서 같은 계열의 top-level Fact 직접 참조를 다시 점검했고, PR #93에서 남아 있던 직접 의존 항목을 추가로 정합화했다.

최종 검증은 Project Python `3.12.13`, ansible-core `2.20.8`, Python kubernetes client `36.0.3`, `kubernetes.core 6.5.0` 환경에서 Kubernetes Actual Run과 Node/Control Plane/Network 회귀 상태를 확인한 범위다.

기존 NET-001의 당시 기록은 보존하고, 이후 확인된 동일 계열 보완 사항은 후속 운영 기준으로 연결한다.

### 3. Controller-local Python도 Project Runtime 사용

Project `.venv`에서 `ansible-playbook`을 실행하더라도 `ansible_controller`의 local connection에서 별도 Interpreter 기준이 없으면 사용자 System Python이 선택될 수 있는 문제가 후속으로 확인됐다.

Gateway TLS Check Mode에서는 Controller-local 모듈이 사용자 Python `3.13`을 선택하면서 Project `.venv`에 설치된 `kubernetes` Library를 찾지 못하는 실패가 발생했다.

이를 다음 기준으로 정합화했다.

```yaml
ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

운영 의미는 다음과 같다.

```text
Project .venv에서 ansible-playbook 실행
→ ansible_playbook_python = Project Python
→ Controller-local Module도 같은 Python 사용
→ Project Dependency / Version Lock 유지
```

이 후속 변경은 최초 NET-001 사건의 당시 결과를 바꾸지 않고, 이후 실행환경 정합화 근거로 연결한다.

### 4. 빈 Project 재현 검증의 범위

빈 Project 실행환경에서는 Project `.venv`와 Collection 경로를 기준으로 공용 환경을 다시 구성하고, Version Lock 확인과 최소 Smoke 및 System 환경 복귀 범위를 검증했다.

이 Evidence는 다음을 의미하지 않는다.

```text
신규 OS에서 Python 자체를 처음부터 설치 완료
신규 Kubernetes Cluster 전체 Clean Build 완료
모든 Playbook의 실제 Apply 완료
```

따라서 실행환경 재현 검증과 신규 인프라 전체 구축 검증을 구분한다.

### 5. 현재 공용 실행 경로

이후 공용 안전 실행기 도입으로 현재 운영 경로에서는 `ars-setup`과 `ars`가 Project Python·`.venv`·고정 Collection 상태를 실행 전에 확인한다.

```text
최초 설정
./tools/ars-setup

이후 실행
ars <playbook.yml>
```

Version Lock과 실행환경 검증은 이 문서의 후속 범위로 연결하지만, Vault/Become 자동 판정, Linux Kernel Keyring Credential 공급, `check_mode:false` Evidence, 위험 Playbook Gate와 Apply 승인 기준은 별도의 운영 결정으로 관리한다.

세부 안전 실행·Credential 운영 기준은 [PROJECT_CHANGES — Ansible 공용 안전 실행기 및 Credential 공급 기준 확정](../../PROJECT_CHANGES.md#ansible-공용-안전-실행기-및-credential-공급-기준-확정)에서 확인한다.

공용 실행환경 이후의 공통 VM 시간 동기화 운영 기준은 [PROJECT_CHANGES — Chrony 공통 NTP 및 상태 기반 검증 기준 확정](../../PROJECT_CHANGES.md#chrony-공통-ntp-및-상태-기반-검증-기준-확정)에서 별도로 관리한다.

## 관련 근거

- Parent Issue #38: https://github.com/seokpan/seokpan-infra/issues/38
- Main 담당자 Issue #41: https://github.com/seokpan/seokpan-infra/issues/41
- Version Lock PR #46: https://github.com/seokpan/seokpan-infra/pull/46
- Exact Python Issue #73: https://github.com/seokpan/seokpan-infra/issues/73
- Exact Python PR #75: https://github.com/seokpan/seokpan-infra/pull/75
- Deprecated Fact Issue #66: https://github.com/seokpan/seokpan-infra/issues/66
- Deprecated Fact PR #77: https://github.com/seokpan/seokpan-infra/pull/77
- Deprecated Fact 후속 PR #93: https://github.com/seokpan/seokpan-infra/pull/93
- MariaDB Collection Issue #94: https://github.com/seokpan/seokpan-infra/issues/94
- MariaDB Collection PR #96: https://github.com/seokpan/seokpan-infra/pull/96
- 빈 Project Bootstrap·Smoke Issue #84: https://github.com/seokpan/seokpan-infra/issues/84
- Controller-local Python Issue #122: https://github.com/seokpan/seokpan-infra/issues/122
- Controller-local Python PR #123: https://github.com/seokpan/seokpan-infra/pull/123
- 공용 안전 실행기 Issue #160: https://github.com/seokpan/seokpan-infra/issues/160
- 공용 안전 실행기 PR #184: https://github.com/seokpan/seokpan-infra/pull/184
- Docs Issue #50: https://github.com/seokpan/seokpan-docs/issues/50
- Docs Issue #145: https://github.com/seokpan/seokpan-docs/issues/145
