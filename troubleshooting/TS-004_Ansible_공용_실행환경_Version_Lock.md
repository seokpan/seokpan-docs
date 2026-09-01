[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-004 — Ansible 공용 실행환경이 버전 고정 없이 시스템 기본 환경에 의존

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

또한 `ansible-core 2.20.8`에서 `INJECT_FACTS_AS_VARS` 관련 경고가 확인되어, 우선 확인된 `kubeadm_control_plane` Template의 top-level fact 참조를 `ansible_facts[...]` 방식으로 변경했다. 이 항목은 Version Lock 자체의 실패가 아니라 새 실행환경에서 드러난 호환성 경고였으므로 별도의 해결 사건으로 확대하지 않았다.

## 검증

Version Lock PR과 Exact Python 검증 보완이 `main`에 반영된 뒤 다음을 확인했다.

- 프로젝트 `.venv`에서 Python `3.12.13`
- ansible-core `2.20.8`
- Python kubernetes client `36.0.3`
- kubernetes.core `6.5.0`
- 기존 System Python/Ansible은 변경하지 않음
- 잘못된 Python patch version을 실제 실패 조건으로 차단

이 보고서는 **버전 고정과 프로젝트 실행환경 분리까지의 해결 내용만 기록**한다. 이후 별도로 수행된 전체 인프라 빈 환경 Smoke/Rollback 검증은 완료 전 작업이므로 이 보고서의 해결 범위에 포함하지 않는다.

## 담당 역할 및 영향

- **이유빈:** 공용 실행환경 제공 및 Ansible 통합
- **정태훈:** Kubernetes/Calico/kubernetes.core 회귀검증
- **최유준:** CI/CD·관측·Argo CD 관련 검증
- **김상희:** MariaDB/MaxScale/NFS 관련 검증

## 관련 근거

- Parent Issue #38: https://github.com/seokpan/seokpan-infra/issues/38
- Main 담당자 Issue #41: https://github.com/seokpan/seokpan-infra/issues/41
- Version Lock PR #46: https://github.com/seokpan/seokpan-infra/pull/46
- Exact Python Issue #73: https://github.com/seokpan/seokpan-infra/issues/73
- Exact Python PR #75: https://github.com/seokpan/seokpan-infra/pull/75
- Deprecated Fact Issue #66: https://github.com/seokpan/seokpan-infra/issues/66
- Deprecated Fact PR #77: https://github.com/seokpan/seokpan-infra/pull/77
