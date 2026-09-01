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

팀원별 `kubernetes.core` 버전 차이도 확인됐다.

## 문제 정의

단순 Upgrade가 목적이 아니었다.

```text
누가 실행해도 같은 환경
Git 정의만으로 다시 생성 가능
기존 System 환경은 Rollback용 보존
```

이 목적이었다.

## 최종 버전 구성표(Version Matrix)

| Component | Version |
|---|---:|
| Project Python | 3.12.13 |
| ansible-core | 2.20.8 |
| Python kubernetes client | 36.0.3 |
| kubernetes.core | 6.5.0 |

시스템 Python 3.9/System Ansible은 Rollback 용도로 유지했다.

## 교차 회귀검증 중 발견된 별도 문제

버전 고정(Version Lock) 후 각 담당자가 자기 영역을 실제로 검증하면서 **기존에 숨어 있던 문제도 발견**됐다.

### Deprecated Ansible 수집 정보(Fact)

Kubernetes Role에서:

```text
ansible_default_ipv4.address
```

사용이 향후 ansible-core에서 제거될 예정이라는 Warning이 발견됐다.

후속 PR에서:

```text
ansible_facts['default_ipv4']['address']
```

방식으로 변경했다.

이후 빈 Project 환경의 최소 동작 검증(Smoke Test)에서 `kubernetes_validate` 계열에 같은 `INJECT_FACTS_AS_VARS` 경고가 다시 확인됐다. 즉 PR #77은 확인된 참조를 제거했지만, 이 계열의 deprecated top-level fact 의존성이 저장소 전체에서 완전히 사라졌다고 단정할 수는 없다. 현재 실행 결과 자체는 PASS였으므로 **기능 장애가 아니라 후속 호환성 정리 항목**으로 유지한다.

### Python Patch Version 검증 결함

초기 `bootstrap.sh`는:

```text
python3.12
```

Major/Minor만 확인했다.

하지만 프로젝트가 요구한 것은 정확히:

```text
3.12.13
```

이었다.

따라서 `sys.version_info[:3]`로 정확한 Patch까지 검증하도록 수정했다.

## 제한사항

이 작업의 핵심 구성과 역할별 회귀검증은 통과했지만, 이후 Issue #84에서 진행한 **빈 환경 재현 검증 / 전체 Smoke / Rollback**은 아직 완전히 닫히지 않았다.

2026-09-01 기준으로는 이전보다 검증 범위가 넓어졌다.

- Kubernetes 주요 Playbook의 빈 환경 Smoke: PASS
- MariaDB/MaxScale/NFS Smoke: PASS
- Data/Storage 영역 `failed=0`, `unreachable=0`: 확인
- Network/LB 영역: TS-019의 코드 결함 발견 후 보완 필요
- Argo CD 영역: TS-018의 공용 kubeconfig 실행 경계 문제로 재검증 필요
- System Ansible Rollback: 최종 Evidence 대기

따라서 현재 제한사항은 "전 영역의 빈 환경 재현을 하지 않았다"가 아니라, **빈 환경 재현 과정에서 드러난 Network/LB와 Argo CD 자동화 결함 및 Rollback Evidence가 아직 남아 있다**는 것이다.

Issue #38/#41의 최초 체크리스트는 과거 Snapshot이므로, 현재 진행상태 판단은 후속 Issue #84의 실제 Smoke 결과를 우선한다.

## 담당 역할 및 영향

- 이유빈: Controller/venv/Lock/Bootstrap
- 정태훈: Kubernetes/Calico/`kubernetes.core`/Argo CD 검증
- 김상희: MariaDB/MaxScale/NFS 검증
- 최유준: Jenkins/Observability 범위 확인

## 관련 근거

- 협업 Issue #38: https://github.com/seokpan/seokpan-infra/issues/38
- Owner Issue #41: https://github.com/seokpan/seokpan-infra/issues/41
- Version Lock PR #46: https://github.com/seokpan/seokpan-infra/pull/46
- Python Exact Check Issue #73: https://github.com/seokpan/seokpan-infra/issues/73
- Python Exact Check PR #74: https://github.com/seokpan/seokpan-infra/pull/74
- Deprecated Fact Issue #66: https://github.com/seokpan/seokpan-infra/issues/66
- Deprecated Fact PR #77: https://github.com/seokpan/seokpan-infra/pull/77
- 빈 Project 재현/Smoke/Rollback Issue #84: https://github.com/seokpan/seokpan-infra/issues/84

## 회귀검증에서 추가로 드러난 기존 환경 Gap

버전 고정 변경 자체가 원인은 아니지만, 전체 팀 회귀검증이 기존 환경의 숨은 전제도 드러냈다.

### DB 관리자 계정의 모니터링 권한 누락

Data/Storage 담당 검증에서 `db_admin` 계정으로 다음 명령을 실행했을 때 권한 부족이 확인됐다.

```sql
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G
```

검증을 위해 실제 Master에서:

```sql
GRANT BINLOG MONITOR, SLAVE MONITOR ON *.* TO 'db_admin'@'%';
```

를 적용했고, Replica에도 권한이 복제된 것을 확인했다.

이것은 **버전 고정 때문에 새로 생긴 문제는 아니다.** 다만 공용 실행환경을 바꾼 뒤 담당자별 실제 명령을 다시 수행했기 때문에 기존 `db_admin` 권한 기준이 검증 작업에 충분하지 않았다는 사실이 드러난 사례다. 따라서 별도 독립 TS로 부풀리지 않고 이 사례의 교차 회귀검증 Evidence로 남긴다.

### Argo CD 검증에서 확인된 Ready 대기 범위 누락

Argo CD/CRD 회귀검증에서는 ApplicationSet CRD 누락 장애 자체는 재현되지 않았고 전체 Runtime은 정상으로 확인됐다. 하지만 `argocd_bootstrap`의 주요 Ready 검증 목록에 `argocd-application-controller` StatefulSet 대기가 빠져 있다는 지적이 추가로 나왔다.

현재 Controller 자체는 정상 동작 중이므로 TS-007과 같은 장애 원인은 아니지만, 향후 초기 구성 자동화(Bootstrap) 완료 판정을 더 엄격하게 만들기 위한 **Validation hardening 항목**으로 남는다.
