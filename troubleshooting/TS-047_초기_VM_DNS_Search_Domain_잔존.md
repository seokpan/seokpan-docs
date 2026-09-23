[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-047 — 초기 VM의 과거 DNS Search Domain 설정이 Pod DNS Search Path까지 남아 있던 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-09 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 프로젝트 VM Resolver 설정, Kubernetes Node·ClusterFirst Pod DNS Search Path |

## 최초 문제

프로젝트 VM 생성 초기에 수동으로 입력했던 `stone.test`, `example.com` Search Domain이 일부 VM에 남아 있었다.

현재 프로젝트의 외부·플랫폼 Endpoint는 `*.seokpan.soldesk.store` Canonical FQDN을 직접 사용하는데, 과거 Search Domain은 현재 계약과 관계없는 설정이었다.

대표적으로 cp-01의 NetworkManager Connection Profile에는 다음 값이 남아 있었다.

```text
ipv4.dns-search: stone.test
```

Host의 `/etc/resolv.conf`에도 같은 Search Domain이 반영됐고, Kubernetes Node에서는 이 값이 신규 ClusterFirst Pod의 DNS Search Path까지 전달될 수 있었다.

```text
NetworkManager Connection Profile
→ Node /etc/resolv.conf
→ kubelet
→ ClusterFirst Pod DNS Search Path
```

실제 Pod에서 다음과 같이 `stone.test`가 Kubernetes 기본 Search Path 뒤에 추가된 상태를 확인했다.

```text
search <namespace>.svc.cluster.local svc.cluster.local cluster.local stone.test
```

## 원인

`stone.test`, `example.com`은 VM 생성 초기 단계에서 사람이 수동으로 입력했던 값이었다.

이후 공식 Endpoint 정책이 `*.seokpan.soldesk.store`로 바뀌었지만 VM의 NetworkManager Profile 또는 일부 Hostname에 초기 설정이 남아 설정 불일치가 발생했다.

일부 Host에서는 NetworkManager Profile의 `ipv4.dns-search`에 값이 남아 있었고, `ansible.example.com`, `nfs.example.com`처럼 초기 FQDN Hostname에서 Search Domain이 파생되는 경우도 확인했다.

## 조치

VM Resolver 설정을 정리하기 위해 `resolver_policy` Role을 추가했다.

자동화는 다음 원칙으로 제한했다.

- IPv4 Default Route가 연결된 NetworkManager Connection만 대상으로 선택
- `stone.test`, `example.com`만 확인된 과거 값으로 제거
- 알 수 없는 제3의 Search Domain은 자동 삭제하지 않음
- `seokpan.soldesk.store`를 새로운 Search Domain으로 추가하지 않음
- Connection `down/up` 대신 `nmcli device reapply` 사용
- IP·Prefix·Gateway·Route·DNS Server는 이번 작업에서 변경하지 않음

초기 FQDN Hostname은 확인된 두 값만 조건부로 정리했다.

```text
ansible.example.com → ansible
nfs.example.com     → nfs
```

모든 Hostname을 일괄적으로 `inventory_hostname`으로 바꾸지는 않았다.

## Check Mode 보완

초기 구현에서는 조회용 `command`/`shell` Task가 Check Mode에서 Skip돼 후속 `register.stdout` 검증이 실패하는 문제가 추가로 확인됐다.

조회 Task는 `check_mode: false`로 실제 상태를 읽도록 하고, NetworkManager Profile이나 Hostname을 변경하는 Task는 Check Mode에서 실행하지 않도록 분리했다.

Check Mode에서는 현재 과거 설정 불일치가 존재할 경우 실제 상태를 바꾸지 않고 `changed=1`로 예상 변경만 보고하도록 보완했다.

## 검증

NetworkManager Profile에 실제 설정 불일치가 있던 Host에서 제거 동작을 확인했다.

대표 검증:

```text
vrouter-01   stone.test 제거
vrouter-03   example.com 제거
```

의도적으로 `stone.test` 설정 불일치를 다시 만든 상태에서 Check Mode를 실행했을 때:

```text
changed=1
failed=0
```

이었고, 실행 후에도 실제 Profile과 `/etc/resolv.conf`에는 설정 불일치가 그대로 남아 있어 Check Mode가 설정을 변경하지 않았음을 확인했다.

이후 Actual Run:

```text
1차   changed=1 / failed=0
2차   changed=0 / failed=0
```

으로 수렴했다.

최종 전체 Inventory 재실행에서는 16개 Host가 모두:

```text
changed=0
unreachable=0
failed=0
```

상태였고, MariaDB·MaxScale과 Kubernetes Control Plane·Pod DNS에도 회귀가 없음을 확인했다.

## Before → After

```text
Before
초기 수동 Search Domain이 일부 VM에 잔존
→ Node resolv.conf에 반영
→ 일부 ClusterFirst Pod Search Path까지 전파

After
확인된 과거 Domain만 Ansible로 제거
→ 제3 Domain과 Network 설정은 보존
→ Check Mode에서 설정 불일치만 보고
→ Actual Run 후 재실행 changed=0
```

## 관련 근거

- Infra Issue #164: https://github.com/seokpan/seokpan-infra/issues/164
- Infra PR #165: https://github.com/seokpan/seokpan-infra/pull/165
