[← 전체 트러블슈팅](../README.md) · [네트워크·Ansible 자동화 목차](README.md)

# NET-005 — Network Role의 NIC Fact 탐색에서 IPv4 자료형을 잘못 가정해 실패

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 |
| **상태** | **해결** |
| **주 담당** | **이유빈 — 네트워크 및 공통 인프라·Ansible 통합** |
| **영향 범위** | `vrouter_network`, `vrouter_firewall`, `lb_network`의 NIC Fact 탐색 및 Check Mode 검증 |

## 문제 개요

빈 Project 환경에서 Network 관련 Playbook을 Smoke Test하던 중 NIC Fact 탐색 단계에서 다음 오류가 발생했다.

```text
object of type 'list' has no attribute 'address'
```

문제가 확인된 대상은 다음 Network Role이었다.

```text
vrouter_network
vrouter_firewall
lb_network
```

실제 VRouter의 NIC와 IPv4 주소 자체는 정상적으로 수집되고 있었다.

예를 들어 `vrouter-01`에서는 다음 정보가 정상적으로 확인됐다.

```text
ens160 = 10.1.93.71
ens224 = 192.168.51.10
```

따라서 실제 NIC 또는 IP 설정 장애가 아니라, Ansible에서 수집한 Fact를 탐색하는 로직의 문제로 범위를 좁혔다.

## 원인

Network Role은 `ansible_facts`를 순회하면서 `ipv4` 정보가 있는 항목을 NIC 후보로 판단한 뒤 다음과 같은 형태로 IPv4 주소에 접근하고 있었다.

```text
value.ipv4.address
```

하지만 `ansible_facts` 안의 모든 `ipv4` 값이 동일한 자료형을 갖는 것은 아니었다.

일부 Fact의 `ipv4` 값은 NIC 정보처럼 Mapping 형태가 아니라 List 형태일 수 있었고, 이 값을 Mapping이라고 가정한 채 `.address`에 접근하면서 오류가 발생했다.

즉 원인은 실제 NIC 정보 누락이 아니라 다음과 같은 자료형 가정이었다.

```text
잘못된 가정

ipv4 값이 존재한다
→ 항상 Mapping이다
→ .address로 접근할 수 있다

실제

ipv4 값이 존재하더라도
→ 일부 Fact에서는 List일 수 있다
→ 자료형 확인 없이 .address 접근 시 실패
```

## 조치

NIC 후보를 탐색할 때 `ipv4` 값이 존재하는지만 확인하지 않고, 해당 값이 Mapping 형태인지 확인한 뒤 주소 정보에 접근하도록 보완했다.

이 기준을 다음 Network Role에 공통 적용했다.

```text
vrouter_network
vrouter_firewall
lb_network
```

같은 PR에서는 Check Mode에서 현재 상태를 읽기 위한 `command` Task가 Skip되면서 이후 검증 Task가 빈 값을 받아 실패하던 부분도 함께 보완했다.

조회 목적의 Task는 Check Mode에서도 현재 상태를 확인할 수 있도록 실행하고, 실제 변경 이후에만 의미가 있는 Assert는 Check Mode에서 제외하도록 정리했다.

이 Check Mode 보완은 NIC Fact 자료형 오류와는 별개의 원인이며, 기존 Kubernetes 영역의 Check Mode 검증 사례인 [K8S-004](../kubernetes-platform-integration/K8S-004_Kubernetes_Check_Mode_검증_Skip.md)의 후속 적용 사례로 연결한다.

## 검증

수정 후 Network 관련 Playbook을 Check Mode로 다시 검증했다.

### vrouter_network

```text
vrouter-01 : failed=0 unreachable=0
vrouter-02 : failed=0 unreachable=0
vrouter-03 : failed=0 unreachable=0
vrouter-04 : failed=0 unreachable=0
```

다음 항목을 정상적으로 확인했다.

- External/Internal NIC 탐지
- NetworkManager Connection 조회
- IPv4 Forwarding 조회
- Runtime/Persistent Static Route 조회
- Check Mode PASS

### vrouter_firewall

```text
vrouter-01 : failed=0 unreachable=0
vrouter-02 : failed=0 unreachable=0
vrouter-03 : failed=0 unreachable=0
vrouter-04 : failed=0 unreachable=0
```

NIC Fact 탐지가 정상적으로 수행됐고 기존 List 자료형 관련 오류가 재발하지 않았다.

### lb_network

```text
lb-01 : failed=0 unreachable=0
```

LB NIC 탐지와 NetworkManager Connection 조회가 정상적으로 수행됐고 기존 List 자료형 관련 오류가 재발하지 않았다.

이번 변경은 기존 Network 설정 의도를 변경하지 않고 NIC Fact 탐색 안정성과 Check Mode 검증 재현성을 보완한 범위다.

## Before → After

```text
Before

ansible_facts 전체 탐색
        ↓
ipv4 값 존재 여부 확인
        ↓
항상 Mapping이라고 가정
        ↓
value.ipv4.address 접근
        ↓
List 형태 Fact에서 실패
```

```text
After

ansible_facts 탐색
        ↓
ipv4 값의 자료형 확인
        ↓
Mapping 형태인 항목만 NIC 후보로 사용
        ↓
IPv4 주소 확인
        ↓
VRouter / LB Network Role Check Mode PASS
```

## 관련 근거

- Infra Issue #101: https://github.com/seokpan/seokpan-infra/issues/101
- Infra PR #103: https://github.com/seokpan/seokpan-infra/pull/103
- Parent Smoke Test Issue #84: https://github.com/seokpan/seokpan-infra/issues/84
- 관련 Check Mode 선례: [K8S-004 — Kubernetes 통합 Playbook의 Ansible Check Mode에서 검증 명령이 Skip되어 Assert가 실패](../kubernetes-platform-integration/K8S-004_Kubernetes_Check_Mode_검증_Skip.md)
