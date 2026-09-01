[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-005 — Ansible 공용 실행환경이 버전 고정 없이 시스템 기본 환경에 의존

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 시작, 2026-08-31 `main` 브랜치 반영 |
| **상태** | **핵심 해결 · 빈 환경 재현 검증은 일부 후속 과제 존재** |
| **주 담당** | **이유빈 — 네트워크 및 공통 인프라·Ansible 통합** |
| **영향 범위** | 정태훈, 김상희, 최유준 전원 |

## 문제 개요

Controller에는 이미 다음 환경이 있었다.

- System Python `3.9.25`
- System ansible-core `2.14.18`

문제는 이 값들이 프로젝트에서 선택·검증한 버전 구성표(Version Matrix)가 아니라 **그냥 이미 설치되어 있던 환경**이었다는 점이다.

또한 담당자별 `kubernetes.core`/Python Kubernetes Client 버전이 달라질 수 있어 같은 Playbook이 사용자마다 다른 결과를 낼 위험이 있었다.

## 문제 정의

목표는 “최신 버전으로 올리기”가 아니라:

> **GitHub 저장소(Repository) 정의만으로 동일한 프로젝트 실행환경을 다시 만들 수 있게 하는 것**

이었다.

## 최종 버전 구성표(Version Matrix)

| Component | Project 기준 |
|---|---:|
| Project Python | `3.12.13` |
| ansible-core | `2.20.8` |
| Python kubernetes client | `36.0.3` |
| kubernetes.core | `6.5.0` |
| System Python | `3.9.25` 유지 |
| System ansible-core | `2.14.18` 유지 |

시스템 기본 환경은 Rollback용으로 남기고, Project `.venv`와 Collection 경로를 분리했다.

## 교차 회귀검증 중 발견된 별도 문제

이 작업은 단순 버전 고정(Version Lock)에 그치지 않고 팀 영역별 회귀검증을 수행했다. 그 과정에서 Argo CD `ApplicationSet` Controller의 기존 `CrashLoopBackOff`가 발견됐지만, 이를 새 버전 구성표(Version Matrix)의 회귀 문제로 오판하지 않고 기존 Argo CD 초기 구성 자동화(Bootstrap) 문제로 분리했다. 해당 실제 장애는 **TS-007**에 별도 기록한다.

## Deprecated Ansible 수집 정보(Fact)

`ansible-core 2.20.8`에서 `INJECT_FACTS_AS_VARS` 관련 Warning이 확인됐다.

현재 기능 장애는 아니었지만 향후 top-level fact 자동 주입 제거에 대비해, 먼저 확인된 `kubeadm_control_plane` Template의:

```text
ansible_default_ipv4.address
```

참조를:

```text
ansible_facts['default_ipv4']['address']
```

형태로 변경하는 Issue #66 / PR #77을 반영했다.

다만 그 이후 빈 Project 환경에서 수행한 Issue #84 Kubernetes 최소 동작 검증(Smoke Test)에서도 `INJECT_FACTS_AS_VARS default to True is deprecated` 경고가 `kubernetes_validate` 계열 검증에서 다시 확인됐다. Smoke 자체는 `failed=0`, `unreachable=0`으로 통과했으므로 기능 장애는 아니지만, **PR #77이 GitHub 저장소 전체의 같은 유형 참조를 모두 제거했다는 의미는 아니다.** 따라서 deprecated fact 정리는 추가 점검이 남은 호환성 보완 항목으로 유지한다.

## Python Patch Version 검증 결함

초기 Bootstrap은 `Python 3.12.x`만 확인해 `3.12.12`, `3.12.14`도 통과할 수 있었다.

공식 Matrix가 `3.12.13` Exact Lock이므로 후속 PR에서 patch-level까지 검사하도록 수정했다.

검증:

- `3.12.13` → PASS
- 테스트 Wrapper로 `3.12.12` 조건 → FAIL, Exit 1

## 제한사항

빈 Project 환경에서 Bootstrap으로 버전 구성표(Version Matrix)를 재생성한 뒤 Smoke가 실제로 진행됐다. 최신 Issue #84 근거 기준:

- Kubernetes 주요 Playbook / Cluster Ansible Check Mode(실제 변경 없이 예상 결과를 확인하는 모드) Smoke: **PASS**
- MariaDB / MaxScale / NFS Data·Storage Smoke: **PASS**
- Common `common_hosts`: PASS
- Network/LB: **TS-019의 NIC Fact / Check Mode 검증 결함으로 일부 FAIL**
- Argo CD Bootstrap: **TS-018의 kubeconfig 경로·권한 불일치로 BLOCKED**
- System Ansible Rollback 최종 검증: 아직 완료 근거 부족
- Kubernetes Smoke에서 `INJECT_FACTS_AS_VARS` 경고 재확인: 기능 PASS와 별개로 남은 deprecated fact 참조 추가 점검 필요

따라서 버전 구성표(Version Matrix) 재생성 자체는 확인됐지만:

> **Version Lock 구현 완료 = 전체 인프라 빈 환경 재현 검증 100% 완료**

라고 표현하면 안 된다. 현재 남은 재현성 Blocker는 버전 고정 자체보다 Network/LB 자동화와 공용 kubeconfig 실행 경계 쪽으로 좁혀졌다.

## 담당 역할 및 영향

- **이유빈:** 공용 실행환경 제공 담당
- **정태훈:** Kubernetes/Calico/kubernetes.core/Argo CD 회귀검증
- **최유준:** CI/CD·모니터링·관측·Argo CD/CRD 검증
- **김상희:** MariaDB/MaxScale/NFS/Redis 관련 검증

## 관련 근거

- Parent Issue #38: https://github.com/seokpan/seokpan-infra/issues/38
- Main 담당자 Issue #41: https://github.com/seokpan/seokpan-infra/issues/41
- Version Lock PR #46: https://github.com/seokpan/seokpan-infra/pull/46
- Deprecated Fact Issue #66: https://github.com/seokpan/seokpan-infra/issues/66
- Deprecated Fact PR #77: https://github.com/seokpan/seokpan-infra/pull/77
- Exact Python Issue #73: https://github.com/seokpan/seokpan-infra/issues/73
- Exact Python PR #75: https://github.com/seokpan/seokpan-infra/pull/75
- 빈 환경 재현 검증 Issue #84: https://github.com/seokpan/seokpan-infra/issues/84
- Issue #84 Smoke 근거: https://github.com/seokpan/seokpan-infra/issues/84#issuecomment-5490758252

## 회귀검증에서 추가로 드러난 기존 환경 Gap

Version Lock 자체와 별개로, 동일 회귀검증 과정에서 DB 관리용 `db_admin` 계정에 `BINLOG MONITOR` / `SLAVE MONITOR` 권한이 빠져 있던 사실도 확인됐다. 필요한 GRANT를 실제 Master에서 추가한 뒤 Replica로 정상 복제되는 것까지 재확인했다.

이 문제는 Project Ansible 버전 구성표(Version Matrix)가 원인이 아니라 **새 실행환경 검증이 기존 수동 DB 권한 구성의 누락을 드러낸 부가 발견**이다. 별도의 장애 연쇄나 후속 자동화 PR이 확인되지 않아 독립 TS로 분리하지 않고 이 사례의 회귀검증 근거로만 보존한다.
