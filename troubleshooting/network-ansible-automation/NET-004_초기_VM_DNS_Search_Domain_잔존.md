[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-047 — 초기 VM의 DNS Search Domain 설정이 Pod까지 남아 있던 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-09 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 프로젝트 VM DNS 설정, Kubernetes Node와 일부 Pod의 DNS Search Path |

## 최초 문제

프로젝트 초기에 VM을 만들면서 수동으로 입력했던 `stone.test`, `example.com` Search Domain이 일부 VM에 그대로 남아 있었다.

현재 프로젝트에서는 다음 **Canonical FQDN**(프로젝트에서 기준으로 사용하는 전체 도메인 이름)을 직접 사용한다.

```text
harbor.seokpan.soldesk.store
db.seokpan.soldesk.store
game.seokpan.soldesk.store
grafana.seokpan.soldesk.store
```

따라서 `stone.test`, `example.com`은 현재 구성에서 사용할 이유가 없는 과거 설정이었다.

cp-01에서는 NetworkManager 설정에 다음 값이 실제로 남아 있었다.

```text
ipv4.dns-search: stone.test
```

이 값은 `/etc/resolv.conf`에도 반영됐고, Kubernetes Node에서는 Pod의 DNS 검색 목록까지 전달됐다.

```text
NetworkManager 설정
→ Node /etc/resolv.conf
→ kubelet
→ Pod DNS Search Path
```

실제 Pod에서도 다음처럼 `stone.test`가 남아 있는 것을 확인했다.

```text
search <namespace>.svc.cluster.local svc.cluster.local cluster.local stone.test
```

즉 **Configuration Drift**(초기에 넣은 설정이 현재 구성과 달라진 뒤에도 일부 서버에 그대로 남아 있는 상태)가 발생했다.

## 원인

`stone.test`, `example.com`은 VM 생성 초기에 사람이 직접 입력했던 값이었다.

프로젝트에서 사용하는 도메인 이름은 이후 변경됐지만 NetworkManager 설정이나 일부 Hostname에 기존 값이 계속 남아 있었다.

일부 서버에서는 `ipv4.dns-search`에 값이 남아 있었고, 다음처럼 초기 Hostname 때문에 `example.com`이 사용되는 경우도 있었다.

```text
ansible.example.com
nfs.example.com
```

## 조치

남아 있는 DNS Search Domain을 Ansible로 정리할 수 있도록 `resolver_policy` Role을 추가했다.

작업 범위는 다음과 같이 제한했다.

- IPv4 Default Route가 연결된 NetworkManager Connection만 수정
- `stone.test`, `example.com`만 제거
- 다른 Search Domain이 발견되면 자동으로 지우지 않음
- `seokpan.soldesk.store`를 새로운 Search Domain으로 추가하지 않음
- Connection을 `down/up`하지 않고 `nmcli device reapply` 사용
- IP·Prefix·Gateway·Route·DNS Server는 변경하지 않음

초기 Hostname도 실제로 확인된 두 경우만 변경했다.

```text
ansible.example.com → ansible
nfs.example.com     → nfs
```

다른 서버의 Hostname은 일괄 변경하지 않았다.

## Check Mode 보완

처음 만든 자동화에서는 현재 상태를 읽는 `command`와 `shell` Task가 Check Mode에서 실행되지 않았다.

이 때문에 다음 Task에서 사용할 값이 비어 검증이 실패했다.

현재 상태를 조회하는 Task는 Check Mode에서도 실행되도록 `check_mode: false`를 지정했다.

반대로 실제 NetworkManager 설정이나 Hostname을 바꾸는 Task는 Check Mode에서 실행하지 않도록 분리했다.

따라서 Check Mode에서는 설정을 실제로 바꾸지 않고, 제거할 값이 있으면 `changed=1`만 표시하도록 했다.

## 검증

실제로 과거 Search Domain이 남아 있던 서버에서 제거를 확인했다.

```text
vrouter-01   stone.test 제거
vrouter-03   example.com 제거
```

이후 시험을 위해 `stone.test`를 다시 넣고 Check Mode를 실행했다.

```text
changed=1
failed=0
```

Check Mode가 끝난 뒤에도 NetworkManager 설정과 `/etc/resolv.conf`에는 `stone.test`가 그대로 남아 있었다. 따라서 Check Mode가 실제 설정을 변경하지 않은 것을 확인했다.

그다음 실제로 Ansible을 실행했다.

```text
1차 실행   changed=1 / failed=0
2차 실행   changed=0 / failed=0
```

마지막으로 전체 16개 관리 대상에 다시 실행했을 때 모두 다음 상태였다.

```text
changed=0
unreachable=0
failed=0
```

MariaDB·MaxScale과 Kubernetes Control Plane·Pod DNS도 기존처럼 정상 동작하는 것을 확인했다.

## Before → After

```text
Before
VM 생성 초기에 넣었던 Search Domain이 일부 서버에 남아 있음
→ Node /etc/resolv.conf에 반영
→ 일부 Pod의 DNS 검색 목록에도 추가됨
→ 현재 구성과 다른 Configuration Drift 발생

After
stone.test / example.com만 Ansible로 제거
→ 다른 DNS 설정은 그대로 유지
→ Check Mode에서는 변경 예정 여부만 표시
→ 실제 적용 후 다시 실행하면 changed=0
```

## 관련 근거

- Infra Issue #164: https://github.com/seokpan/seokpan-infra/issues/164
- Infra PR #165: https://github.com/seokpan/seokpan-infra/pull/165
