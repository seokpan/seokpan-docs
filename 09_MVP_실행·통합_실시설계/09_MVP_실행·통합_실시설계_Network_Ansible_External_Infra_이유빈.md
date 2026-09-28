# 09_MVP_실행·통합_실시설계

## Network / Ansible / External Infra

---

## 목차

1. [목적과 범위](#1-목적과-범위)
2. [선행 기준과 상태 판정](#2-선행-기준과-상태-판정)
3. [역할과 책임 경계](#3-역할과-책임-경계)
4. [Network / External Infra Current State](#4-network--external-infra-current-state)
5. [Ansible Current State](#5-ansible-current-state)
6. [Provider / Consumer Contract](#6-provider--consumer-contract)
7. [Integration 흐름](#7-integration-흐름)
8. [Integration Gate](#8-integration-gate)
9. [Gate 요약](#9-gate-요약)
10. [남은 Blocker와 Gap](#10-남은-blocker와-gap)
11. [Critical Path](#11-critical-path)
12. [완료 기준](#12-완료-기준)
13. [Traceability](#13-traceability)

---

# 1. 목적과 범위

## 1.1 목적

본 문서는 「石나가는 판단」 1차 프로젝트에서 **Network / Ansible / External Infra** 영역의 책임, Current State, Provider/Consumer 관계, Integration Gate와 완료 경계를 실제 구현·검증 결과에 맞춰 정리한다.

이 영역의 통합은 다음 흐름을 기준으로 판단한다.

```text
Ansible 실행환경과 관리 대상 확인
    ↓
VRouter의 사설망 Route·Forwarding·Firewall 구성
    ↓
lb-01의 Common VIP·HAProxy 전달 경로 구성
    ↓
Kubernetes·Database·Delivery·Observability 영역에 연결 조건 제공
    ↓
각 Consumer의 실제 접속·응답 결과 확인
```

VRouter에 Route가 정의되어 있다는 사실만으로 Application 통신 완료를 선언하지 않는다. HAProxy Listener가 존재한다는 사실만으로 Gateway Route나 DB Query까지 검증됐다고 판단하지도 않는다.

- 상세 실행 명령, 적용·재실행·장애 복구·Rollback 절차: **11 MVP 구축·자동화 Runbook**
- 공식 Test Case, 측정값, PASS/FAIL 및 실행별 Evidence: **12 MVP 검증·측정 계획**
- Repository·Branch·Issue·PR·Review 운영 기준: [10 GitHub 협업 및 Repository 운영](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)

## 1.2 범위

| 영역 | 현재 주요 구성 | 이 문서에서 확인하는 경계 |
| --- | --- | --- |
| VRouter | 네 사설망과 VRouter 4대 | NIC·Forwarding·Static Route·왕복 통신 조건 |
| Firewall / NAT | firewalld `external`·`internal` zone | 실제 유입 zone의 Source·Port 허용과 사설망 간 원본 IP 보존을 위한 NAT 제외 조건 |
| LB Network | `lb-01`, `10.1.93.90` | VIP와 네 사설망으로의 Return Route |
| HAProxy | HTTP·HTTPS·Kubernetes API·DB Listener | Port별 Backend 전달과 Consumer 연결 |
| Project Endpoint | `game`, `db`, `harbor` 등의 이름·주소 기준 | Host·CoreDNS·외부 Client별 게시 범위 |
| Ansible Controller | Inventory, Project `.venv`, 공용 실행기 | 실행환경·Credential·사전 점검·승인 경계 |
| 공통 시간 | Chrony NTP 기준 | VM의 공통 설정과 실제 동기화 상태 |

Windows Host의 VMware 설정, VM Hardware·vNIC 생성, CentOS 설치 및 최초 SSH 준비는 수동 공통 기반이다. Ansible의 관리 범위는 SSH 접근이 준비된 Guest VM부터 시작한다.

# 2. 선행 기준과 상태 판정

## 2.1 선행 기준

직접 상위 기준은 다음 문서다.

- [05 물리 아키텍처](../01-08_기획·설계_Baseline/05_SeokPan_물리_아키텍처.md) — 물리망, VMnet, VRouter 및 Common VIP의 원 목표
- [06 Ansible 자동화·테스트 설계](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md) — 수동 공통 기반, 자동화 시작점 및 검증 경계
- [07 확장 호환형 MVP 도출](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md) — 단일 LB를 사용하는 1차 MVP 범위
- [PROJECT_CHANGES.md](../PROJECT_CHANGES.md) — Baseline 이후 확정된 Endpoint·실행환경·Chrony·안전 실행 기준
- [MVP_IMPLEMENTATION_BASELINE.md](../MVP_IMPLEMENTATION_BASELINE.md) — App·Infra·GitOps가 함께 사용하는 구현 기준
- [CURRENT_STATE.md](../CURRENT_STATE.md) — 1차 종료 시점의 전체 구현·검증 및 남은 범위

01~08은 당시 설계 기준이다. **05의 원 목표에 있는 LB 2대와 Keepalived 자동 VIP 전환을 현재 구현 상태로 옮기지 않는다.** 현재 MVP는 `lb-01` 한 대가 `10.1.93.90`을 고정 보유한다.

실제 구성값은 `seokpan-infra`의 현재 `main` Inventory·Role·Playbook을 기준으로 확인한다. 실행 성공이나 통합 완료 주장은 해당 Issue·병합 PR·Runtime Evidence에서 확인된 대상과 조건으로 한정한다.

[CURRENT_STATE.md](../CURRENT_STATE.md)는 **2026-09-23 종료 스냅샷**이며, 2026-09-27 후속 현행화는 GitHub 기록·Source 기준으로 수행됐다. 본문의 Runtime 판정은 기록된 실행 시점의 Evidence를 뜻하며, 운영 서버를 이 문서 작성 시점에 새로 접속해 재측정했다는 뜻은 아니다.

## 2.2 상태 표현 기준

| 상태 | 의미 |
| --- | --- |
| `Defined` | 설계 또는 구성 방법이 정의됨 |
| `Implemented` | 코드나 설정으로 구현됨 |
| `Merged` | 변경이 Repository `main`에 반영됨 |
| `Running` | 대상 환경에서 동작이 관찰됨 |
| `Validated` | 명시한 조건에서 실제 실행·응답을 확인함 |
| `Partial` | 일부 대상이나 조건만 검증됨 |
| `Not Tested` | 해당 시험의 실제 결과가 없음 |
| `Not Implemented` | 해당 기능이 현재 코드·설정에 구현되지 않음 |
| `Planned` | 후속 작업의 목표는 정해졌으나 완료 근거가 없음 |
| `Deferred` | 현재 1차 MVP 범위에서 후속으로 분리함 |

### 중요한 판정 원칙

```text
Implemented ≠ Merged ≠ Running ≠ Validated

Route 정의
≠ VM 간 왕복 통신
≠ 서비스 Port 응답
≠ Application 기능 검증

HAProxy Listener
≠ Gateway Route
≠ HTTPS/Backend API/WebSocket Runtime 검증
≠ 전체 MVP Acceptance
```

이 문서의 Gate는 역할 간 **실행·통합 진행 상태**를 요약한다. 12 문서의 공식 Test Case와 실행별 결과를 대체하지 않는다.

# 3. 역할과 책임 경계

## 3.1 주요 책임

Network / Ansible / External Infra 영역은 **VM 간 통신의 기반, 공통 외부 접근 경로, Infra Playbook의 재현 가능한 실행 경계**를 담당한다.

- Ansible Inventory·공통 변수 및 Controller 실행환경
- VRouter의 NIC 판정, IPv4 Forwarding, Static Route
- VRouter firewalld zone·정책 및 사설망 목적지 NAT 제외
- `lb-01` 관리 주소, Common VIP와 사설망 Return Route
- HAProxy의 서비스·Kubernetes API·DB Port별 L4 전달
- Host·CoreDNS·Client별 Project Endpoint 게시 기준
- Chrony 공통 시간 설정과 상태 기반 확인
- 공용 안전 실행기의 Credential·사전 점검·승인 기준

## 3.2 직접 소유하지 않는 영역

| 영역 | 주요 Owner 및 이 문서와의 경계 |
| --- | --- |
| Kubernetes Cluster·Gateway·Application Runtime | Kubernetes / Application Integration. 이 문서는 Node·Gateway까지의 네트워크 조건을 제공한다. |
| MariaDB·MaxScale·NFS·Backup/Restore | Database / Storage / Recovery. 이 문서는 DB Endpoint까지의 접근 경로를 제공한다. |
| Exporter·Prometheus·Log·Alert·Dashboard | Delivery / Observability. 이 문서는 VRouter·LB의 Route·Firewall·Metric Port 경계를 연결한다. |
| GitHub Governance | 10 문서 |
| 단계별 실행·복구 절차 | 11 문서 |
| 공식 시험·측정·Evidence | 12 문서 |

VRouter나 LB에 영향을 주는 변경이라도 다른 담당자가 구현한 PR은 해당 구현자의 작업으로 기록한다. 네트워크 원인 분석, 설정 경계 검토 및 통합 판정의 기여와 PR 구현 기여를 혼동하지 않는다.

# 4. Network / External Infra Current State

## 4.1 사설망과 VRouter

현재 [`seokpan-infra` Inventory](https://github.com/seokpan/seokpan-infra/blob/main/ansible/inventory/hosts.yml)는 VRouter 네 대를 관리한다. 각 VRouter는 `10.1.93.0/24` 외부망과 담당 `192.168.51.0/24`~`192.168.54.0/24` 사설망 사이의 Gateway 역할을 한다.

| VRouter | 외부 관리 IP | 내부 Gateway | 다른 사설망의 목적지와 다음 홉 |
| --- | --- | --- | --- |
| `vrouter-01` | `10.1.93.71` | `192.168.51.10` | `192.168.52.0/24 → 10.1.93.73`<br>`192.168.53.0/24 → 10.1.93.75`<br>`192.168.54.0/24 → 10.1.93.77` |
| `vrouter-02` | `10.1.93.73` | `192.168.52.10` | `192.168.51.0/24 → 10.1.93.71`<br>`192.168.53.0/24 → 10.1.93.75`<br>`192.168.54.0/24 → 10.1.93.77` |
| `vrouter-03` | `10.1.93.75` | `192.168.53.10` | `192.168.51.0/24 → 10.1.93.71`<br>`192.168.52.0/24 → 10.1.93.73`<br>`192.168.54.0/24 → 10.1.93.77` |
| `vrouter-04` | `10.1.93.77` | `192.168.54.10` | `192.168.51.0/24 → 10.1.93.71`<br>`192.168.52.0/24 → 10.1.93.73`<br>`192.168.53.0/24 → 10.1.93.75` |

개별 VRouter의 목표 Route는 [`seokpan-infra`의 `inventory/host_vars`](https://github.com/seokpan/seokpan-infra/tree/main/ansible/inventory/host_vars)에 정의되어 있다.

[`vrouter_network` Role](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/vrouter_network/tasks/main.yml)은 Inventory의 `ansible_host`가 붙은 NIC를 외부 NIC로, 담당 사설망의 `.10` 주소가 붙은 NIC를 내부 NIC로 탐지한다. 후보가 각각 하나이고 서로 다른지 확인한 뒤 IPv4 Forwarding과 Runtime·Persistent Route를 관리한다. NIC 이름을 모든 VRouter에 동일한 값으로 고정하지 않는다.

[`seokpan-infra` PR #103](https://github.com/seokpan/seokpan-infra/pull/103)은 일부 NIC Fact의 자료형으로 인한 탐색 오류와 Check Mode의 조회 누락을 보완했다. [`seokpan-infra` Issue #132](https://github.com/seokpan/seokpan-infra/issues/132)와 이를 해결한 [PR #134](https://github.com/seokpan/seokpan-infra/pull/134)는 목표 Route가 이미 적용됐는데도 반복 실행에서 `changed=1`이던 문제를 수정했다.

PR #134의 당시 검증에서는 네 VRouter의 Actual Run 2회차가 모두 `changed=0`, `failed=0`, `unreachable=0`이었다. [`seokpan-infra` PR #134의 추가 검증 댓글](https://github.com/seokpan/seokpan-infra/pull/134#issuecomment-5536619923)에서는 `vrouter-01`의 영구 Route 한 건을 임시로 제거한 뒤 Actual Run에서 복원하고, 재실행 시 `changed=0`인 경로도 확인했다. 이 결과는 해당 Route와 당시 검증한 Gateway 통신 범위의 근거이며, 모든 서비스 Port의 전수 통신 결과는 아니다.

## 4.2 firewalld와 NAT

[`vrouter_firewall` Role](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/vrouter_firewall/tasks/main.yml)은 외부 NIC를 `external`, 내부 NIC를 `internal` zone에 배치한다. 두 zone 간 정책과 SSH 접근 보호를 관리한다.

현재 Role의 [`private_network` 값](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/vrouter_firewall/defaults/main.yml)은 `192.168.0.0/16`이다. `external` zone에 설정하는 선택적 Masquerade 규칙은 목적지가 이 대역에 속하는 패킷을 제외하고, 그 외 목적지에 적용되도록 정의되어 있다. 네 프로젝트 사설망 간에는 원본 Source IP를 보존하도록 구성한다. 이 규칙을 “인터넷 목적지에만 적용된다”라고 해석하지 않는다.

Port 허용 여부는 기본 zone만 보고 판정하지 않는다. 실제 패킷이 유입되는 NIC·zone과 Source 대역을 함께 확인해야 한다. 이 차이로 발생한 VRouter Metric 수집 문제는 7.3절에 기록한다.

## 4.3 `lb-01`과 HAProxy

현재 `lb-01`의 관리 주소는 `10.1.93.78`, Common VIP는 `10.1.93.90/24`다. [`lb_network` Role](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_network/defaults/main.yml)에는 네 사설망으로 돌아가는 Static Route가 정의되어 있다.

[`lb_haproxy` 설정](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_haproxy/templates/haproxy.cfg.j2)은 Port별 TCP 전달을 사용한다.

| VIP Listener | HAProxy Backend | 역할 |
| --- | --- | --- |
| `10.1.93.90:80` | Worker NodePort `30080` | HTTP 진입 |
| `10.1.93.90:443` | Worker NodePort `30443` | HTTPS·WSS 진입 |
| `10.1.93.90:6443` | `cp-01/02/03:6443` | Kubernetes API 진입 |
| `10.1.93.90:3306` | `maxscale-01:3306` | DB 접근 지점 |

HAProxy는 L4 전달을 맡는다. `/`, `/api/v1`, `/ws/v1`의 선택은 Kubernetes Gateway·HTTPRoute의 책임이다. DB의 TLS 세션과 Query 성공 여부는 Database·Application 영역의 실제 검증 결과로 판정하고, MariaDB 복제 상태는 Database / Storage / Recovery 영역에서 별도로 판정한다.

현재는 **`lb-01` 한 대의 고정 VIP 구조**다. Keepalived에 의한 LB 간 자동 전환은 현재 구현·검증 결과가 아니다.

## 4.4 Project Endpoint와 이름 해석

[`project_endpoints`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/inventory/group_vars/all/vars.yml)는 서비스 이름·주소·Port와 Host·CoreDNS·외부 Client 게시 여부를 구분한다.

| Endpoint | 현재 정의 | 게시·접근 경계 |
| --- | --- | --- |
| `game.seokpan.soldesk.store` | `10.1.93.90:80/443` | Host·CoreDNS·Client 대상 서비스 경로 |
| `db.seokpan.soldesk.store` | `10.1.93.90:3306` | 내부 Host·CoreDNS의 DB 접근 기준 |
| `harbor.seokpan.soldesk.store` | `192.168.53.61:443` | 내부 Registry 접근 기준 |
| `k8s-api.seokpan.soldesk.store` | `10.1.93.90:6443` | Client용 이름 계약은 있으나 API 인증서 SAN 정합화 전 Host·CoreDNS 게시 보류 |
| Jenkins·Argo CD 외부 UI | 이름 계약 존재 | 외부 Route 보류 |

이름이 `project_endpoints`에 정의된 것, Host·Pod에서 해석되는 것, 실제 서비스에 접속해 응답을 받는 것은 각각 다르게 판정한다.

`game.seokpan.soldesk.store`의 Windows Host 접속은 Host 측 수동 `hosts` 설정을 사용했다. [`seokpan-infra` Issue #188](https://github.com/seokpan/seokpan-infra/issues/188)은 Let’s Encrypt Production 인증서 적용 후 Windows Host Browser와 프로젝트 Linux VM의 HTTPS 접속을 확인하고, A-10의 Backend API·WebSocket 경로 검증을 연결한다. 이는 Public A Record 게시나 공인 인터넷 Inbound 공개를 뜻하지 않는다.

## 4.5 외부 VM Metric의 Network 경계

Prometheus가 VRouter와 LB의 Metric을 수집할 때는 사설망 Route뿐 아니라 실제 유입 zone, 대상 Port와 Observability 측 수집 설정이 함께 맞아야 한다.

VRouter의 `node_exporter`는 `9100/tcp`를 사용한다. 당시 `node-exporter-external` Target `7/7 UP`에는 VRouter 4대·MaxScale·NFS·Ansible Controller가 포함된다. 이를 VRouter 7대의 검증으로 표현하지 않는다.

`lb-01`의 HAProxy 네이티브 Metric은 서비스 VIP가 아닌 관리 IP `10.1.93.78:8404`에 바인딩된다. [`seokpan-infra` PR #208](https://github.com/seokpan/seokpan-infra/pull/208)은 LB Listener와 당시 로컬·클러스터 내 접근 결과를 기록한다. `lb01-haproxy-exporter` Prometheus Target UP은 Observability 측 수집 경로까지 연결된 별도 결과다.

Exporter 설치·Prometheus Target·Query와 Dashboard의 최종 판정은 [Delivery / Observability 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md) 및 해당 12 문서와 연결한다.

# 5. Ansible Current State

## 5.1 Controller와 고정 Project 실행환경

현재 Inventory의 Ansible Controller는 `ansible` (`192.168.54.70`, local connection)이다. Project 실행환경은 [`requirements.txt`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/requirements.txt), [`requirements.yml`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/requirements.yml), [`version_lock_bootstrap.sh`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/bootstrap/version_lock_bootstrap.sh)를 기준으로 구성한다.

| 구성 | 현재 고정 버전 |
| --- | --- |
| Project Python | `3.12.13` |
| ansible-core | `2.20.8` |
| Python Kubernetes client | `36.0.3` |
| `kubernetes.core` Collection | `6.5.0` |
| `ansible.mariadb` Collection | `6.0.2` |

[`seokpan-infra` Issue #84](https://github.com/seokpan/seokpan-infra/issues/84)는 기존 `.venv`와 Collection을 재사용하지 않는 빈 Project 환경에서 Version Matrix, Inventory Parse·Ping·Syntax·최소 Smoke 및 System 환경 복귀를 검증했다. [`seokpan-infra` Issue #94](https://github.com/seokpan/seokpan-infra/issues/94)와 [PR #96](https://github.com/seokpan/seokpan-infra/pull/96)은 MariaDB Collection을 공용 정의에 추가했다.

판정: **Project 환경 재현과 기존 VM 대상 최소 Smoke는 검증됨.** Python 자체 설치 자동화나 신규 VM부터 전체 인프라를 구축한 Clean Build 결과로 확대하지 않는다.

## 5.2 공용 안전 실행기와 Credential 경계

[`seokpan-infra` Issue #160](https://github.com/seokpan/seokpan-infra/issues/160)과 이를 구현한 [PR #184](https://github.com/seokpan/seokpan-infra/pull/184)는 최초 사용자 설정용 `ars-setup`과 공용 실행 명령 `ars`를 현재 운영 경로로 확정했다. `ars`는 각 사용자의 checkout에 있는 [`ansible-safe-run`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/ansible-safe-run)을 가리킨다.

각 사용자는 자신의 Linux 계정·checkout·`.venv`·Kernel Keyring을 사용한다. 허용 Controller 계정은 `ansible`, `jth`, `ksh`, `cyj`다. 실행기는 Playbook별 Vault와 Become Password의 필요 여부를 **별도로** 판단하고 필요한 Credential만 메모리에서 공급한다.

다음 경계를 적용한다.

- Controller에서 `ars`를 호출할 때 로컬 `root` 직접 실행, `sudo`·setuid 형태 및 허용되지 않은 계정 실행 차단
- 미검토 `check_mode:false` Task는 근거 없이 통과시키지 않고 중단
- 위험 Playbook에는 별도 사전 Gate 적용
- 실제 Apply 전 사용자 승인
- Credential의 Git·평문 파일·명령행 인자·일반 로그 노출 금지

Controller의 `root` 실행 차단과 관리 대상 VM의 SSH 사용자는 다른 경계다. 현재 [Inventory](https://github.com/seokpan/seokpan-infra/blob/main/ansible/inventory/hosts.yml)의 `ansible_user: root`는 **관리 대상에 접속할 원격 계정**을 뜻한다.

`--inspect-only`는 Ansible 구조·Syntax와 필요한 Vault decrypt를 확인하지만 SSH·Become 실행, Check Mode, Apply는 수행하지 않는다.

[PROJECT_CHANGES.md](../PROJECT_CHANGES.md)의 사용자별 Evidence에서 `ksh`는 실제 Vault/Become Apply까지, `jth`는 별도 checkout의 setup·inspect와 일반 실행 중 Check Mode 중단까지 확인됐다. `ansible` 계정의 회귀검증도 기록되어 있다. **네 허용 계정이 모두 같은 Playbook을 Apply한 결과로 합치지 않는다.**

판정: **실행기 구현·병합 완료, 사용자·Playbook별 검증 범위는 상이함.** 명령별 사전 확인·실행·실패 처리 순서는 11 Runbook에서 관리한다.

## 5.3 공통 시간 기준

[`chrony` Role](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/chrony/defaults/main.yml)은 `pool 2.centos.pool.ntp.org iburst`를 공통 NTP Source로 사용한다. 모든 VM의 Chrony RPM을 한 Patch Version으로 강제하는 대신 공통 설정, `chronyd` active/enabled와 실제 동기화 상태를 확인한다.

[`seokpan-infra` Issue #39](https://github.com/seokpan/seokpan-infra/issues/39)와 이를 구현한 [PR #40](https://github.com/seokpan/seokpan-infra/pull/40)의 당시 검증에는 Harbor를 포함한 16대 VM이 포함된다. Check Mode와 Actual Run 1·2차의 `failed=0`, `unreachable=0`, 반복 실행 `changed=0`이 기록됐다.

판정: **해당 16대·당시 구성 기준 검증 완료.** 이후 추가·변경된 모든 VM의 재검증을 뜻하지 않는다.

# 6. Provider / Consumer Contract

| 이 영역이 제공하는 조건 | 소비 영역 | 함께 확인해야 할 Consumer 조건 |
| --- | --- | --- |
| VRouter 네 대의 Route·Forwarding·Return Path | Kubernetes, DB·Storage, Delivery | 각 VM의 Gateway, 대상 Port·서비스 Listener |
| 사설망 간 원본 IP 보존을 위한 NAT 제외 정책 및 필요한 Firewall 허용 | 외부 VM·Kubernetes Node | 실제 출발지·목적지·응답 방향의 통신 |
| `10.1.93.90`의 Port별 HAProxy 전달 | Gateway·Kubernetes API·MaxScale | Worker NodePort, Control Plane API, MaxScale 응답 |
| Host·CoreDNS·Client별 Endpoint 게시 기준 | Application·GitOps·운영 Client | 이름 해석 이후 실제 TLS·Route·서비스 접속 |
| VRouter·LB Metric 접근 경로 | Observability | Exporter 기동, EndpointSlice·NetworkPolicy, Target·Query |
| 고정 Project Runtime과 공용 안전 실행 기준 | 각 역할의 Infra Playbook 운영자 | 자기 checkout·Credential과 Playbook별 실제 실행 결과 |
| Chrony 공통 시간 기준 | 관리 대상 VM | 설정·서비스·실제 동기화 상태 |

**Provider Ready ≠ Consumer Validated**를 유지한다. 네트워크 설정이 제공된 뒤에는 이를 소비하는 Kubernetes·DB·Observability 영역의 실제 결과를 연결해야 통합을 판정할 수 있다.

# 7. Integration 흐름

## 7.1 외부 사용자 서비스

```text
Windows Host Browser 또는 프로젝트 Linux VM
    ↓ game.seokpan.soldesk.store → 10.1.93.90
lb-01 HAProxy :443
    ↓
Worker NodePort 30443
    ↓
Gateway API / NGINX Gateway Fabric
    ├─ /          → Frontend
    ├─ /api/v1    → Backend API
    └─ /ws/v1     → Backend WebSocket
```

Network / External Infra는 VIP·LB·Worker 방향의 L3/L4 전달 경계를 제공한다. Gateway의 HTTPRoute와 Application 응답은 [Kubernetes / Application Integration 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)의 Runtime Gate와 교차 확인한다.

현재 A-10에서는 HTTPS·Backend API·WebSocket 경로가 검증됐고, Windows Host Browser와 프로젝트 Linux VM의 서비스 접속도 확인됐다. 이 결과를 P4의 동시성·재접속·성능·장애 복구 전체 Acceptance로 확대하지 않는다.

## 7.2 사설망 간 통신

```text
출발 사설망 VM
    ↓
출발지 VRouter
    ↓ 10.1.93.0/24를 통해 전달
목적지 VRouter
    ↓
목적 사설망 VM
    ↓
반대 방향 Route를 통해 응답
```

`192.168.0.0/16` 목적지는 현재 Masquerade 규칙에서 제외되어 프로젝트 사설망 간 원본 Source IP를 보존하도록 설정돼 있다. 실제 통신은 출발 Route, 목적지 접근 정책, 응답 Route를 함께 확인해야 한다.

## 7.3 Prometheus → VRouter 경로

VRouter에 `node_exporter`가 설치되고 로컬 `:9100/metrics`가 응답했어도 `vrouter-02/03/04`의 Prometheus Target은 처음에 Down이었다.

[`seokpan-infra` Issue #205](https://github.com/seokpan/seokpan-infra/issues/205)의 조사에서 Worker→VRouter 관리 IP의 패킷은 `ens160`의 `external` zone으로 유입되는데 `9100/tcp`는 `internal` zone에만 허용되어 거부되는 원인을 확인했다.

[`seokpan-infra` PR #206](https://github.com/seokpan/seokpan-infra/pull/206)은 당시 Worker 대역인 `192.168.51.0/24`·`192.168.52.0/24`에 한해 외부 zone의 `9100/tcp`를 허용하도록 `vrouter_firewall` Role에 반영했다. 당시 대상 HTTP `200`, `node-exporter-external` Target `7/7 UP`(VRouter 4대·MaxScale·NFS·Ansible Controller) 및 재실행 시 해당 방화벽 변경 없음이 기록됐다.

이 사례는 **Exporter 로컬 기동 → 실제 유입 zone 통과 → Prometheus Target UP**이 서로 다른 검증 단계임을 보여준다. PR의 구현, 네트워크 원인 진단과 리뷰, 최종 Target 검증의 담당 범위는 각각의 Evidence에 따른다.

## 7.4 DB VIP와 Kubernetes API 경계

확인된 DB 접속 흐름은 다음과 같다.

```text
Backend Pod
    ↓ db.seokpan.soldesk.store:3306
lb-01 Common VIP 10.1.93.90:3306
    ↓ HAProxy TCP 전달
maxscale-01:3306
    ↓
MariaDB
```

[Database / Storage / Recovery 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md)는 DB VIP/FQDN 접속과 MaxScale Provider 결과를 기록한다. [Kubernetes / Application Integration 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)는 A-10 Backend Runtime의 MaxScale TLS 세션 성공을 기록한다. 이 결과는 해당 DB Consumer 경로의 확인 근거이며, LB 자동 전환이나 모든 DB 장애 상황의 검증 결과는 아니다.

Kubernetes API의 VIP `10.1.93.90:6443` Listener 정의는 별도 경계다. `k8s-api.seokpan.soldesk.store`는 API Server 인증서 SAN 정합화 전이어서 Host·CoreDNS 게시가 보류돼 있다. 따라서 DB 접속 성공이나 HAProxy Listener 정의만으로 API FQDN의 인증서 검증·Client 접속까지 완료됐다고 판정하지 않는다.

# 8. Integration Gate

| Gate | 통과 조건 | 현재 판정 | 근거 |
| --- | --- | --- | --- |
| A — Project Runtime Ready | 고정 실행환경 재현, Inventory·최소 Smoke | `Validated` — 빈 Project 환경 및 기존 VM 최소 검증 범위 | `seokpan-infra` Issue [#84](https://github.com/seokpan/seokpan-infra/issues/84)·[#94](https://github.com/seokpan/seokpan-infra/issues/94), PR [#96](https://github.com/seokpan/seokpan-infra/pull/96) |
| B — VRouter Route Ready | NIC·Forwarding·Runtime/Persistent Route 및 재실행 | `Validated` — 당시 VRouter 4대 Route와 반복 Actual Run | `seokpan-infra` PR [#103](https://github.com/seokpan/seokpan-infra/pull/103)·[#134](https://github.com/seokpan/seokpan-infra/pull/134) |
| C — Firewall / NAT Boundary | 실제 NIC·zone, 사설망 NAT 제외, 필요한 Source·Port | `Partial` — Role의 목표 정책과 확인된 `9100/tcp` 경로. 필수 경로 전체의 검증은 별도 확인 필요 | [`vrouter_firewall`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/vrouter_firewall/tasks/main.yml), `seokpan-infra` Issue [#205](https://github.com/seokpan/seokpan-infra/issues/205) / PR [#206](https://github.com/seokpan/seokpan-infra/pull/206) |
| D — Common Endpoint / LB | VIP Listener·Backend와 각 Consumer의 실제 접속 경로 | `Partial` — A-10 외부 HTTPS·Backend API·WebSocket 경로, DB VIP/FQDN 접속 및 Backend→MaxScale TLS 연결은 해당 범위에서 `Validated`. `k8s-api.seokpan.soldesk.store`는 API 인증서 SAN 정합화 전 Host·CoreDNS 게시가 보류돼 있어 Kubernetes API FQDN의 Client 접속까지 Gate 전체를 완료로 판정하지 않음 | [`lb_network`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_network/defaults/main.yml)·[`lb_haproxy`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_haproxy/templates/haproxy.cfg.j2), `seokpan-infra` Issue [#188](https://github.com/seokpan/seokpan-infra/issues/188), [Database / Storage / Recovery 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md), [Kubernetes / Application Integration 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md) |
| E — External Metric Network | VRouter·LB Metric Port와 Observability Consumer 결과 | `Validated` — 당시 `node-exporter-external` Target `7/7 UP`(VRouter 4대·MaxScale·NFS·Ansible Controller)과 별도 `lb01-haproxy-exporter` Target UP 범위. 다른 외부 VM의 Metric·Log 경로 전체를 완료로 판정하지 않음 | `seokpan-infra` PR [#206](https://github.com/seokpan/seokpan-infra/pull/206)·[#208](https://github.com/seokpan/seokpan-infra/pull/208), [Delivery / Observability 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md) 및 해당 12 문서 |
| F — Common Time | 공통 설정·서비스·동기화 및 재실행 | `Validated` — 당시 Harbor 포함 16대 | `seokpan-infra` Issue [#39](https://github.com/seokpan/seokpan-infra/issues/39) / PR [#40](https://github.com/seokpan/seokpan-infra/pull/40) |
| G — Safe Ansible Execution | 사용자별 환경·Credential·차단·승인 경계 | `Partial` — 확인된 사용자·Playbook 경로는 `Validated`; 나머지 조합의 Apply 근거는 없음 | `seokpan-infra` Issue [#160](https://github.com/seokpan/seokpan-infra/issues/160) / PR [#184](https://github.com/seokpan/seokpan-infra/pull/184), [PROJECT_CHANGES.md](../PROJECT_CHANGES.md) |

Gate의 판정은 **해당 행에 적힌 대상·시점·절차**에 적용한다. 구성 또는 Revision이 바뀌면 새로운 Test Run을 12 문서에 기록하고 판정을 갱신한다.

# 9. Gate 요약

현재 근거로 확인할 수 있는 주요 통합 결과는 다음과 같다.

- VRouter 네 대의 목표 Static Route와 당시 반복 Actual Run의 멱등성
- `lb-01`의 고정 VIP `10.1.93.90` 및 Port별 L4 전달 구조
- A-10의 외부 HTTPS·Backend API·WebSocket 경로와 Windows Host·Linux VM 서비스 접속
- DB VIP/FQDN 접속 및 A-10 Backend→MaxScale TLS 연결
- 방화벽 보완 후 당시 `node-exporter-external` Target `7/7 UP`(VRouter 4대·MaxScale·NFS·Ansible Controller)
- 별도 `lb01-haproxy-exporter` Target UP
- Harbor 포함 16대의 당시 Chrony 설정·동기화·반복 실행
- Project Ansible 실행환경 재현과 안전 실행기의 사용자·Playbook별 검증 결과

각 결과의 대상과 한계는 8장 Gate의 판정 범위를 따른다.

# 10. 남은 Blocker와 Gap

| 항목 | 현재 상태 | 경계 |
| --- | --- | --- |
| `lb-02`·Keepalived 자동 VIP 전환 | `Deferred (Not Implemented)` | 원 목표 구조이며 현재 `lb-01`은 단일 장애점이다. |
| 사용자별 SSH Bootstrap 상태 점검·누락 공개키 선택적 복구 | `Planned` | [`seokpan-infra` Issue #27](https://github.com/seokpan/seokpan-infra/issues/27)의 Open 후속 작업이다. 최초 점검, 일부 공개키 누락, VM 재구성 및 Snapshot Rollback 후 복구를 포함한다. 구현·검증 완료로 표기하지 않는다. |
| 신규 VM부터 전체 인프라 구축 | `Not Tested` | 전체 Clean Build 기준의 판정이다. 빈 Project 실행환경 재현·기존 VM Smoke와 구분한다. |
| 필수 통합 경로 중 미검증 조합 | `Partial` | 확인된 Route·Listener·Consumer 경로만 해당 범위의 PASS로 기록한다. |
| P4 성능·장애·복구 Acceptance | `Partial` | 역할별로 확인된 시험 결과는 인정한다. 다만 이 문서만으로 P4 전체 Acceptance의 PASS를 선언하지 않으며 Application·Recovery 영역의 개별 결과와 연결한다. |

현재의 A-10 외부 Runtime Gate 완료와 P4 최종 Acceptance의 미완료 검증은 별개 상태로 유지한다.

# 11. Critical Path

```text
Windows Host·VM·초기 SSH 준비
    ↓
Inventory와 Project Ansible 실행환경
    ↓
VRouter Route·Forwarding·Firewall·Return Path
    ↓
lb-01 VIP·HAProxy Listener
    ↓
Kubernetes·Database·Delivery Provider/Consumer 연결
    ↓
외부 사용자 경로 및 Observability 결과 확인
```

앞 단계의 코드 Merge만으로 다음 단계가 자동 PASS가 되지는 않는다. 각 역할의 제공 조건과 Consumer 결과를 함께 연결해 Gate를 판정한다.

# 12. 완료 기준

## 12.1 Network / Ansible / External Infra 통합 완료 범위

다음 체크는 명시한 대상·시점·절차에서 확인된 결과를 뜻한다. 8장에서 `Partial`로 판정한 Gate 전체나 인프라 전체 Acceptance의 완료 표시는 아니다.

- [x] VRouter 4대의 목표 Static Route와 당시 반복 Actual Run 검증
- [x] `lb-01` 고정 VIP를 통한 A-10 외부 HTTPS·Backend API·WebSocket 경로 확인
- [x] DB VIP/FQDN 접속 및 A-10 Backend→MaxScale TLS 연결 확인
- [x] 당시 `node-exporter-external` Target `7/7 UP`(VRouter 4대·MaxScale·NFS·Ansible Controller)
- [x] `lb-01`의 별도 `8404` Metric 응답과 `lb01-haproxy-exporter` Target UP 확인
- [x] Harbor 포함 당시 16대의 Chrony 공통 기준 검증
- [x] 빈 Project Ansible 실행환경 재현과 기존 VM 대상 최소 Smoke
- [x] `ars` 구현·병합 및 기록된 사용자·Playbook 경로의 검증

다음 항목은 위의 완료 범위에 포함하지 않는다.

- **Partial:** 사용자·Playbook 조합 중 일부만 실제 Apply까지 검증된 상태와 필수 통합 경로 중 일부 조합만 확인된 상태
- **Not Tested:** 신규 VM부터 전체 인프라 Clean Build
- **Kubernetes API FQDN:** API Server 인증서 SAN 정합화와 Host·CoreDNS 게시 전이며, 해당 FQDN의 Client 접속 시험은 `Not Tested`
- **Planned:** 사용자별 SSH Bootstrap 상태 점검과 누락 공개키 선택적 복구 — [`seokpan-infra` Issue #27](https://github.com/seokpan/seokpan-infra/issues/27)
- **Deferred:** `lb-02`·Keepalived 자동 VIP 전환
- **P4 후속 검증:** 역할별 확인 결과는 인정하되 성능·장애·복구 전체 Acceptance는 완료로 판정하지 않음

## 12.2 문서 작성 기준

- 현재 Inventory·Role·Playbook과 01~08 원 목표·MVP 축소 범위를 구분한다.
- Network·Ansible·External Infra의 직접 책임과 다른 영역의 Consumer 경계를 명확히 한다.
- 4장 Current State → 6장 Provider/Consumer → 7장 Integration 흐름 → 8장 Gate에서 같은 대상을 추적할 수 있다.
- `Validated` 판정마다 대상·시점·검증 범위·근거가 있다.
- 상세 실행 절차와 Test Run은 11·12 문서의 책임으로 유지한다.
- 다른 담당자의 PR 구현을 이 영역의 단독 성과로 표현하지 않는다.
- Secret 값과 미실행 시험의 성공 결과를 기록하지 않는다.

문서의 작성 완료와 인프라 전체 Acceptance는 각각의 근거로 별도 판정한다.

# 13. Traceability

| 항목 | 주요 Source | 이 문서에서 사용하는 근거 |
| --- | --- | --- |
| 원 목표·MVP 축소 | [05](../01-08_기획·설계_Baseline/05_SeokPan_물리_아키텍처.md)·[06](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md)·[07](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md) | 수동 기반, VRouter 4대, 원 목표와 단일 LB MVP의 차이 |
| VRouter 주소·Route | [`seokpan-infra` Inventory](https://github.com/seokpan/seokpan-infra/blob/main/ansible/inventory/hosts.yml)·[`host_vars`](https://github.com/seokpan/seokpan-infra/tree/main/ansible/inventory/host_vars) | 현재 코드의 대상과 목표 다음 홉 |
| Network Role 검증 | `seokpan-infra` PR [#103](https://github.com/seokpan/seokpan-infra/pull/103)·[#134](https://github.com/seokpan/seokpan-infra/pull/134) | NIC Fact·Check Mode 보완, Route 반복 Actual Run의 멱등성 및 [PR #134 추가 검증 댓글](https://github.com/seokpan/seokpan-infra/pull/134#issuecomment-5536619923)의 누락 Route 복원 결과 |
| Firewall·Metric 경로 | [`vrouter_firewall`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/vrouter_firewall/tasks/main.yml), `seokpan-infra` Issue [#205](https://github.com/seokpan/seokpan-infra/issues/205) / PR [#206](https://github.com/seokpan/seokpan-infra/pull/206), [Delivery / Observability 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md) 및 해당 12 문서 | Masquerade 목적지 제외 규칙, 실제 유입 zone 진단, Worker→VRouter `9100/tcp` 복원 및 당시 `node-exporter-external` Target `7/7 UP` |
| VIP·HAProxy | [`lb_network`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_network/defaults/main.yml)·[`lb_haproxy`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/roles/lb_haproxy/templates/haproxy.cfg.j2), `seokpan-infra` PR [#208](https://github.com/seokpan/seokpan-infra/pull/208), [Delivery / Observability 12](../12_MVP_검증·측정_계획/12_MVP_검증·측정_계획_Delivery_Observability_최유준.md) | VIP·Backend·Port, LB `8404` 접근과 별도 Prometheus Target UP 결과 |
| Endpoint·외부 접속·DB Consumer | [`project_endpoints`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/inventory/group_vars/all/vars.yml), `seokpan-infra` Issue [#188](https://github.com/seokpan/seokpan-infra/issues/188), [Database / Storage / Recovery 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md), [Kubernetes / Application Integration 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md) | 이름 게시 경계, A-10 HTTPS·Backend API·WebSocket 경로와 Windows Host·Linux VM 접속, DB VIP/FQDN 접속과 Backend→MaxScale TLS 결과. Kubernetes API FQDN SAN·게시 보류와 구분 |
| Project 실행환경 | [`bootstrap`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/bootstrap/version_lock_bootstrap.sh), `seokpan-infra` Issue [#84](https://github.com/seokpan/seokpan-infra/issues/84)·[#94](https://github.com/seokpan/seokpan-infra/issues/94) / PR [#96](https://github.com/seokpan/seokpan-infra/pull/96) | 고정 버전, 빈 환경 재구성·최소 Smoke, MariaDB Collection 추가 |
| 공용 안전 실행 | [`ansible-safe-run`](https://github.com/seokpan/seokpan-infra/blob/main/ansible/ansible-safe-run), `seokpan-infra` Issue [#160](https://github.com/seokpan/seokpan-infra/issues/160) / PR [#184](https://github.com/seokpan/seokpan-infra/pull/184), [PROJECT_CHANGES.md](../PROJECT_CHANGES.md) | Controller에서 root·sudo·setuid 및 비허용 계정 실행 차단, local/ssh 연결 경로 구분, 사용자별 Credential·사전 점검·승인 계약과 검증 범위 |
| Chrony | `seokpan-infra` Issue [#39](https://github.com/seokpan/seokpan-infra/issues/39) / PR [#40](https://github.com/seokpan/seokpan-infra/pull/40) | 당시 16대의 공통 설정·동기화·재실행 |
| SSH Bootstrap 후속 | `seokpan-infra` Issue [#27](https://github.com/seokpan/seokpan-infra/issues/27) | 사용자·VM별 상태 점검과 누락 공개키 선택적 복구의 목표. Snapshot Rollback을 포함하지만 그 상황에만 한정되지 않으며 완료 Evidence로 사용하지 않음 |
