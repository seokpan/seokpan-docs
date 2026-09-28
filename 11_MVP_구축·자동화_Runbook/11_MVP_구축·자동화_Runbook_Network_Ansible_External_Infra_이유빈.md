# MVP 구축·자동화 Runbook

## Network / Ansible / External Infra

## 1. 목적과 사용 범위

이 문서는 「石나가는 판단」 1차 프로젝트의 Network / Ansible / External Infra 영역을 점검·적용·재실행·복구·Rollback하기 위한 실행 Runbook이다. 09 문서는 책임과 Integration Gate를 정의하고, 본 문서는 실제 작업 순서와 중단 조건을 다룬다. 정식 Test Case와 PASS/FAIL 및 Evidence는 12 문서에 기록한다.

직접 기준은 [Network / Ansible / External Infra 09](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Network_Ansible_External_Infra_이유빈.md), [10 GitHub 협업 및 Repository 운영](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md), [PROJECT_CHANGES.md](../PROJECT_CHANGES.md), [MVP_IMPLEMENTATION_BASELINE.md](../MVP_IMPLEMENTATION_BASELINE.md), `seokpan-infra`의 실행 대상 Revision과 관련 Issue·PR·Runtime Evidence다. 과거 검증은 당시 시점·대상·Revision에 한정한다.

```text
Controller 사용자·checkout·Project Ansible 환경 확인
  → Inventory·대상 접근 확인
  → VRouter Route·Forwarding·Firewall·NAT
  → lb-01 VIP·Return Route·HAProxy
  → Endpoint·Chrony 확인
  → Kubernetes·Database·Observability Consumer 결과 연결
```

Windows Host의 VMware 구성, VM Hardware·vNIC 생성, OS 설치와 최초 SSH 준비는 Ansible 자동 적용 범위 밖이다. 현재 LB는 `lb-01` 한 대이며 `lb-02`·Keepalived 자동 VIP 전환은 운영 절차에 포함하지 않는다.

## 2. 실행 원칙

### 2.1 구현 기준과 작업 위치

Inventory·Role·Playbook·Endpoint 변수의 구현 기준은 `seokpan-infra/ansible`이다. 작업자는 자신의 checkout에서 실행한다.

```bash
cd /path/to/your/seokpan-infra/ansible
```

문서 저장소인 `seokpan-docs`에서 Infra Playbook을 실행하지 않는다. 이 문서의 예시 값과 실행 Revision의 Inventory·Role·변수가 다르면 실행 Revision을 대조하고 원인을 확인한다.

### 2.2 결과 판정

```text
Defined ≠ Implemented ≠ Merged ≠ Running ≠ Validated

Route 존재 ≠ 왕복 통신 ≠ 서비스 성공

HAProxy Listener ≠ Backend 정상 ≠ Application·DB·API 검증

Exporter 로컬 응답 ≠ Worker 접근 ≠ Prometheus Target UP
```

09에서 `Partial`인 Gate는 Runbook에 절차가 있다는 이유로 `Validated`가 되지 않는다. 과거 Actual Run 성공과 현재 Safe Runner에서의 재실행 가능성도 별도 판정한다.

### 2.3 Credential과 중단 조건

Vault·Become Password, Token, Private Key를 Git 파일, Issue·PR, 문서, 일반 명령행 인자 또는 일반 `-e` 인자에 기록하지 않는다. 민감한 추가 변수는 `ansible-safe-run`의 비표시 입력 경로를 사용하고 사용자별 Credential은 Kernel Keyring 경로를 따른다.

다음 상황에서는 Apply를 중단한다.

- Inventory의 예상 대상·대상 수, NIC 탐지 결과 또는 실행 Revision이 실제 작업 계획과 다르다.
- Project Python·Ansible 환경, 대상 접속, Credential 또는 Safe Runner 사전 점검이 실패한다.
- 미검토 `check_mode: false` Task가 차단되거나 Check Mode의 조회값 누락으로 후속 Task가 실패한다.
- Route·Firewall 변경 시 SSH 유지 또는 독립적인 복구 접근 경로가 없다.
- `lb-01` 관리 IP·VIP·Return Route의 차이를 설명할 수 없거나 checkout의 Source가 실행 중 변경됐다.

Safe Runner가 중단한 작업을 일반 `ansible-playbook` 직접 실행으로 우회하지 않는다.

## 3. 현재 실행 기준 상태

09의 Gate 상태와 현재 Runbook의 판단 경계는 다음과 같다.

| Gate | 영역 | 09의 검증 상태 | 실행 경계 |
| --- | --- | --- | --- |
| A — Project Runtime | 고정 환경·Inventory·최소 Smoke | `Validated` — 빈 Project 환경과 기존 VM 범위 | 사용자·checkout별 재확인 |
| B — VRouter Route | NIC·Forwarding·Route | `Validated` — 당시 4대·반복 Actual Run | Runner 경계는 아래 표 참조 |
| C — Firewall / NAT | zone·Policy·NAT·Source/Port | `Partial` | Runner 경계는 아래 표 참조 |
| D — Common Endpoint / LB | VIP·HAProxy·Endpoint·Consumer | `Partial` | 서비스·DB 결과와 API FQDN 보류 분리. Runner 경계는 아래 표 참조 |
| E — External Metric | VRouter :9100, LB :8404 | `Validated` — 당시 Target 범위 | 네트워크 접근과 Prometheus 결과 분리 |
| F — Common Time | Chrony 설정·서비스·동기화 | `Validated` — 당시 Harbor 포함 16대 | 이번 대상의 상태 재확인 |
| G — Safe Ansible | 사용자·Credential·차단·승인 | `Partial` | 사용자·Playbook별 결과 분리 |

`lb-02`·Keepalived는 `Deferred`, SSH Bootstrap 복구는 Infra Issue #27의 `Planned` 작업이다. Kubernetes API FQDN은 `pending_api_san`, 신규 VM부터의 전체 Clean Build는 `Not Tested`다. P4 전체 Acceptance는 역할별 12 문서의 결과로 판정한다.

현재 Revision의 Safe Runner 실행 경계는 다음과 같다.

| Playbook | 현재 실행 경계 |
| --- | --- |
| `vrouter_network` | 미등록 `check_mode: false` Evidence로 코드상 일반 `ars` Apply 차단 대상. 실제 실행 결과는 별도 기록 |
| `vrouter_firewall` | 자동 Check Mode 및 일반 `ars` Apply 통과 여부 미검증 |
| `lb_network` | 일부 조회 Evidence는 등록됐으나 Playbook 전체의 Check Mode·Apply 통과 여부 미검증 |
| `lb_haproxy` | 조회 Task·후속 Assert의 Check Mode 정합성과 Apply 결과 미검증 |

코드상 차단 판단은 Role과 Runner 로직의 대조 결과이며 실제 실패 로그를 대신하지 않는다. 미검증은 실패가 확인됐다는 뜻이 아니다. 각 명령의 진행·중단 조건은 6~9장에 둔다.

## 4. 공통 Pre-check

### 4.1 Repository와 Revision

실행할 개인 `seokpan-infra/ansible` checkout에서 Branch·Commit·작업 상태를 확인한다.

```bash
cd /path/to/your/seokpan-infra/ansible
git status --short
git branch --show-current
git rev-parse HEAD
git log --oneline --decorate -n 5
```

필요하면 `git fetch --prune`으로 원격 추적 참조를 갱신한다. 이는 현재 Branch를 최신 `main`으로 이동시키지 않는다. 실행 SHA와 미병합 변경의 영향을 확인하기 전에는 Apply하지 않는다.

### 4.2 실행 위치와 계정

`ars ...`와 `.venv/bin/...`은 Ansible Controller의 개인 Infra checkout에서 `ansible`, `jth`, `ksh`, `cyj` 중 현재 일반 사용자로 실행한다. Controller에서 root·sudo·setuid 실행은 금지한다. VRouter의 `ip`·`nmcli`·`sysctl`·`firewall-cmd`, `lb-01`의 `ip`·`nmcli`·`haproxy`·`systemctl`·`ss`는 각 대상 VM에서 필요한 조회·관리 권한으로 실행한다. Metric 원격 `curl`은 `worker-01` 등 실제 Consumer 출발지에서 실행한다.

Inventory의 `ansible_user: root`는 관리 대상의 원격 SSH 계정이며 Controller에서 `ars`를 root로 실행한다는 뜻이 아니다.

### 4.3 Project Runtime과 Inventory

고정 버전은 Python 3.12.13, ansible-core 2.20.8, kubernetes.core 6.5.0, ansible.mariadb 6.0.2, Python kubernetes 36.0.3이다.

```bash
./.venv/bin/python --version
./.venv/bin/ansible --version
./.venv/bin/ansible-galaxy collection list kubernetes.core
./.venv/bin/ansible-galaxy collection list ansible.mariadb
./.venv/bin/python -c 'import kubernetes; print(kubernetes.__version__)'
```

`.venv`가 없거나 버전이 다르면 System Ansible로 대신 진행하지 않고 5장의 설정 절차를 확인한다.

### 4.4 Inventory Parse

```bash
./.venv/bin/ansible-inventory \
  -i inventory/hosts.yml \
  --graph
```

대상 그룹과 대상 수가 예상과 일치하는지 확인한다. VRouter별 주소는 6.1절, `lb-01` 주소는 8.1절에서 확인한다.

### 4.5 변경 전 Runtime 상태

VRouter에서 다음을 조회한다.

```bash
ip -br address
sysctl net.ipv4.ip_forward
ip -4 route
firewall-cmd --get-active-zones
firewall-cmd --zone=external --list-all
firewall-cmd --zone=internal --list-all
```

`lb-01`에서 다음을 조회한다.

```bash
ip -br address
ip -4 route
ss -lnt
systemctl is-active haproxy
```

Route·Firewall 변경 전에는 현재 SSH 세션 외에 VMware Console 등 독립적인 복구 접근 경로도 확인한다.

## 5. Project 실행환경과 안전 실행 — Gate A·G

### 5.1 사용자별 설정

각 사용자는 자신의 Linux 계정·checkout·Project `.venv`·Collection·Kernel Keyring Credential·`ars` alias를 사용한다. 최초 설정은 일반 계정으로 개인 Infra checkout에서 실행한다.

```bash
cd /path/to/your/seokpan-infra/ansible
./tools/ars-setup
```

`ars-setup`은 Python exact lock과 `requirements.txt`·`requirements.yml`, 사용자별 Credential 및 현재 checkout의 `ansible-safe-run`을 가리키는 alias를 확인·구성한다. 새 Shell을 열거나 다시 로그인해 확인한다.

```bash
type ars
```

기존 `.venv`의 Python 버전이 다르면 원인을 확인한다. `.venv`를 임의 삭제하거나 다른 Python으로 우회하지 않는다.

### 5.2 구조 확인과 실제 실행

```bash
ars playbooks/vrouter_network.yml --inspect-only
```

`--inspect-only`는 구조·Syntax와 필요한 Vault 해석을 확인하지만 대상 접속, Become 실행·Runtime 판정, Check Mode, Apply는 수행하지 않는다. Vault가 필요한 경우 사용자별 Keyring 경로를 사용한다.

일반 실행 `ars <Playbook> [허용된 옵션]`은 Playbook 구조 → 대상·작업·권한 → 필요한 Vault·Become Credential → 대상 접속 → Check Mode 또는 안전 정책의 사전 판정 → 계획 출력 → 사용자 Apply 승인 → Actual Run의 순서로 진행한다. 출력된 대상·Credential 필요 여부·변경 예상이 작업 계획과 다르면 승인하지 않는다.

미검토 `check_mode: false` Task의 Evidence가 현재 Source와 다르면 차단될 수 있다. 해당 Task가 Check Mode에서 실제로 어떤 조회나 변경을 수행하는지 확인하고, 현재 Repository Snapshot과 일치하는 Evidence만 정합화한다. Evidence를 맞춘 사실도 Actual Run 성공과는 별개로 기록한다.

Controller에서 root·sudo·setuid로 `ars`를 실행하는 것은 금지한다. 정상 일반 사용자가 실행한 Playbook의 `localhost`·`connection: local` 대상은 별도 연결 정책에 따라 취급하며 원격 SSH 접속 결과로 기록하지 않는다.

한 사용자의 환경 준비와 Apply 결과를 네 허용 계정 전체 또는 다른 Playbook에 확대하지 않는다. 사용자별 checkout·Keyring·Credential 요구·Check Mode·Actual Run을 구분한다. Project 환경 재현은 신규 VM부터의 전체 Clean Build 결과가 아니다.

## 6. VRouter Route·Forwarding — Gate B

### 6.1 대상과 목표

`playbooks/vrouter_network.yml`은 Inventory `vrouters` 그룹에 `vrouter_network` Role을 적용한다.

| VRouter | 외부 관리 IP | 내부 Gateway IP |
| --- | --- | --- |
| `vrouter-01` | `10.1.93.71` | `192.168.51.10` |
| `vrouter-02` | `10.1.93.73` | `192.168.52.10` |
| `vrouter-03` | `10.1.93.75` | `192.168.53.10` |
| `vrouter-04` | `10.1.93.77` | `192.168.54.10` |

Role은 `ansible_host` 주소의 외부 NIC와 담당 사설망 `.10` 주소의 내부 NIC를 탐지한다. 후보는 각각 하나이며 서로 달라야 한다. 다른 세 사설망의 목표 Route와 다음 홉은 실행 Revision의 `inventory/host_vars/vrouter-01.yml`부터 `vrouter-04.yml`까지 확인한다.

NIC 이름을 모든 VRouter에서 같은 값으로 가정하지 않는다. 자동 탐지 결과가 실제 `ip -br address`와 다르면 Apply를 중단한다. Route 목록은 이 문서의 과거 표를 복사하지 않고 현재 `host_vars`와 대조한다.

### 6.2 변경 전 확인

```bash
ip -br address
sysctl net.ipv4.ip_forward
ip -4 route
nmcli -g GENERAL.CONNECTION device show <외부_NIC>
nmcli -g ipv4.routes connection show <Connection_Name>
```

외부·내부 NIC, IPv4 Forwarding, Runtime·Persistent Route, 목표 다음 홉 및 SSH 복구 접근을 대조한다.

Runtime Route가 맞아도 NetworkManager의 영구 Route가 빠졌다면 재부팅 후 유지된다는 근거가 없다. 반대로 영구 Route만 있어도 현재 통신이 된다는 근거가 없다. 두 상태와 실제 왕복 통신을 따로 기록한다.

### 6.3 현재 Safe Runner 경계

```bash
ars playbooks/vrouter_network.yml --inspect-only
```

현재 Role에는 NetworkManager Connection·Forwarding·Route 상태 조회를 위한 명시적 `check_mode: false` Task가 있으나 Runner의 `REVIEWED_CHECK_FALSE` Evidence에는 `vrouter_network/tasks/main.yml`이 등록되지 않았다. 다음 명령은 현재 확정된 Apply 절차가 아니다.

```bash
ars playbooks/vrouter_network.yml --limit vrouter-01
ars playbooks/vrouter_network.yml
```

Task의 안전성과 현 Source를 검토해 Runner Evidence를 정합화한 뒤 제한 대상에서 일반 `ars` 사전 점검·Actual Run을 확인하고 확대한다. 과거 PR #134의 네 대 반복 Actual Run 성공을 현재 Runner 호환성의 근거로 쓰지 않는다.

### 6.4 적용 후·재실행

Role은 Forwarding의 영구·Runtime 상태와 Runtime·NetworkManager Persistent Static Route를 관리한다. Runtime Route는 목표와 다를 때 `ip route replace`로 정합화하며, Persistent Route는 비교 후 차이가 있을 때 수정한다. 저장 후 `nmcli connection up`을 실행하지 않는 경계를 유지한다.

현재 Runner 정합화와 실제 Apply가 끝나면 다음을 확인한다.

```bash
ip -br address
sysctl net.ipv4.ip_forward
ip -4 route
nmcli -g ipv4.routes connection show <Connection_Name>
```

Forwarding 1, 의도한 NIC, 다른 세 사설망의 Runtime·Persistent Route, Inventory의 다음 홉을 확인한다. 필요한 출발지 VM에서 상대 VM까지 왕복 경로를 별도로 확인한다.

```bash
ping -c 3 <검증할_상대_IP>
```

ICMP 결과는 서비스 Port 검증을 대신하지 않는다. Runner 정합화와 첫 Actual Run이 완료된 뒤 같은 Revision·Inventory·대상에서 두 번째 Actual Run을 수행한다.

```bash
ars playbooks/vrouter_network.yml --limit <대상_VRouter>
```

목표 상태가 이미 맞다면 `changed=0`, `failed=0`, `unreachable=0`을 기대한다. 누락 Route를 의도적으로 복원하는 시험은 정상 상태의 2회차 멱등성과 구분해 기록한다.

## 7. Firewall·NAT — Gate C

### 7.1 대상과 정책

`playbooks/vrouter_firewall.yml`은 `vrouters`에 `vrouter_firewall` Role을 적용한다. 외부·내부 NIC는 탐지 후 각각 `external`·`internal` zone에 배치하며 양방향 정책의 Target은 `ACCEPT`다. `private_network`는 `192.168.0.0/16`이다.

Role은 `external` zone의 전체 Masquerade를 제거하고 목적지가 이 private 대역이 아닌 경우의 선택적 Masquerade Rich Rule을 관리한다.

```text
destination not address="192.168.0.0/16" masquerade
```

규칙 존재와 실제 통신에서 원본 Source IP가 보존됐다는 측정은 구분한다. 이 규칙을 “인터넷 목적지에만 NAT”라고 축약하지 않는다.

`192.168.0.0/16` 목적지는 이 선택적 규칙에서 제외되며, 그 밖의 목적지는 규칙의 대상이다. 실제 포워딩 결과는 출발 Route, 목적지 정책, 응답 Route 및 수신지에서 본 Source IP까지 확인한다.

### 7.2 변경 전·후 확인

SSH 세션과 별도 Console 접근을 확보하고 Runtime·Permanent 상태를 조회한다.

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --zone=external --list-all
firewall-cmd --zone=internal --list-all
firewall-cmd --permanent --zone=external --list-all
firewall-cmd --permanent --zone=internal --list-all
firewall-cmd --zone=external --list-rich-rules
firewall-cmd --permanent --zone=external --list-rich-rules
firewall-cmd --permanent --get-policies
firewall-cmd --permanent --info-policy=int-to-ext
firewall-cmd --permanent --info-policy=ext-to-int
```

Policy 이름과 ingress·egress zone·Target, NIC→zone 배치, SSH 허용, 전체 Masquerade 제거, 선택적 Rule, 필요한 Source·Port를 대조한다. 원본 IP 보존의 실제 관찰은 12의 별도 시험에 인계한다.

이 명령은 적용 전과 후에 같은 대상에서 실행한다. `--permanent` 결과는 저장된 설정이고 기본 조회는 현재 Runtime 설정이다. 하나만 맞는 상태를 완료로 기록하지 않는다.

### 7.3 Safe Runner와 적용 순서

```bash
ars playbooks/vrouter_firewall.yml --inspect-only
ars playbooks/vrouter_firewall.yml --limit vrouter-01
```

일부 `firewall-cmd` 조회는 `ansible.builtin.command`와 후속 `rc`·`stdout` 조건을 사용하면서 명시적 `check_mode: false`가 없다. `--inspect-only`만으로 자동 Check Mode나 Apply 성공을 선언하지 않는다. 제한 대상 명령은 사전 점검 진입점이며, Check Mode 결과·대상·변경 내용을 확인한 후에만 Apply 여부를 결정한다. 조회값 누락 등으로 실패하면 Role·Runner 정합화 후 재검증한다.

첫 대상의 Apply, SSH 유지, 정책·Rich Rule 확인 후에만 다른 VRouter로 확대한다.

```bash
ars playbooks/vrouter_firewall.yml
```

이 실행도 별도 사전 점검과 승인이 필요하다. 예상과 다른 NIC 배치나 SSH 단절이 발생하면 확대를 중단한다.

같은 Revision·Inventory·대상에서 재실행할 때는 Recap의 변경 수뿐 아니라 NIC zone, Policy, SSH 허용, Masquerade 및 `9100/tcp` Rich Rule의 Runtime·Permanent 출력을 함께 비교한다. Check Mode 통과만으로 실제 정책의 멱등성을 선언하지 않는다.

### 7.4 VRouter 9100/tcp

현재 Observability의 VRouter Endpoint는 `192.168.51.10:9100`부터 `192.168.54.10:9100`까지의 내부 Gateway 주소다. 다른 사설망 Worker의 요청은 대상 VRouter 외부 NIC·zone으로 유입될 수 있다. Role은 Worker Source `192.168.51.0/24`·`192.168.52.0/24`에 대해 `external` zone `9100/tcp`를 Rich Rule로 허용한다. 실제 유입 zone과 Source를 확인하고, 원격 결과는 11장과 연결한다. Gate C는 여전히 `Partial`이다.

## 8. lb-01 Network·VIP — Gate D

### 8.1 대상과 Return Route

`playbooks/lb_network.yml`은 `loadbalancer` 그룹의 `lb-01`에 `lb_network` Role을 적용한다. 관리 IP는 `10.1.93.78`, 고정 VIP는 `10.1.93.90/24`다.

현재 Return Route는 `192.168.51.0/24 → 10.1.93.71`, `192.168.52.0/24 → 10.1.93.73`, `192.168.53.0/24 → 10.1.93.75`, `192.168.54.0/24 → 10.1.93.77`이다. 실제 적용 목표는 실행 Revision의 `lb-01` 변수에서 재확인한다.

### 8.2 변경 전 확인과 적용

```bash
ip -br address
ip -4 route
nmcli -g GENERAL.CONNECTION device show <관리_IP가_있는_NIC>
nmcli -g ipv4.addresses connection show <Connection_Name>
nmcli -g ipv4.routes connection show <Connection_Name>
```

관리 IP·VIP·Return Route의 Runtime·Persistent 상태와 SSH 접근을 확인한다.

```bash
ars playbooks/lb_network.yml --inspect-only
ars playbooks/lb_network.yml --limit lb-01
```

Connection 조회의 `check_mode: false` Evidence는 등록됐으나 이것만으로 전체 Playbook의 자동 Check Mode·Apply 성공을 보장하지 않는다. 제한 대상 일반 명령의 사전 점검을 확인하고 예상 변경과 일치할 때만 Apply한다. 조회값 누락 시 중단해 Role·Runner를 정합화한다.

### 8.3 적용 후 확인

위 조회 명령으로 VIP의 주소·Prefix, 각 영구 Route의 목적지·다음 홉과 Runtime 경로를 함께 대조한다. 목적지만 같고 다음 홉이 다르거나 VIP 주소만 같고 Prefix가 다른 경우 Role 재실행만으로 자동 교정된다고 가정하지 않는다. VIP가 있어도 HAProxy와 Backend가 준비되지 않았다면 Consumer 경로는 완료가 아니다. 현재 `lb-01`은 단일 장애점이며 자동 VIP 인계는 없다.

Runtime과 Persistent 어느 한쪽만 목표에 맞는 경우에도 다른 쪽의 결과를 적는다. 확인하지 않은 값에 대해서는 “정상”으로 표시하지 않는다. VIP·Route가 이미 존재해 재실행 시 `changed=0`이더라도 잘못된 기존 다음 홉이나 Prefix가 그대로 남아 있을 수 있으므로 두 필드를 직접 대조한다.

## 9. HAProxy 전달 — Gate D

### 9.1 Listener와 Backend

`playbooks/lb_haproxy.yml`은 `lb-01`에 `lb_haproxy` Role을 적용하고 L4 TCP를 전달한다.

```text
10.1.93.90:80   → Worker NodePort 30080
10.1.93.90:443  → Worker NodePort 30443
10.1.93.90:6443 → cp-01/02/03:6443
10.1.93.90:3306 → maxscale-01:3306
```

실행 Revision의 변수와 다음 현재 값을 대조한다: Worker `192.168.51.30`·`192.168.52.30`, Control Plane `192.168.51.20`·`192.168.52.20`·`192.168.53.20`, MaxScale `192.168.53.40`.

Worker는 `worker-01/02`, Control Plane은 `cp-01/02/03`, DB Backend는 `maxscale-01`을 뜻한다. HAProxy의 `:80`·`:443`은 Worker NodePort로, `:6443`은 Control Plane API로, `:3306`은 MaxScale로 전달한다. 이 표는 Listener 계약이며 실제 Backend 기능 결과는 해당 Consumer에게서 받아야 한다.

### 9.2 변경 전 확인과 Runner 경계

```bash
systemctl is-active haproxy
haproxy -c -f /etc/haproxy/haproxy.cfg
ss -lnt
ip -br address
ip -4 route
ars playbooks/lb_haproxy.yml --inspect-only
```

현재 Role의 적용 후 `systemctl is-active haproxy`·`ss -lnt` 조회는 명시적 `check_mode: false`가 없고 후속 Assert가 조회 출력을 사용한다. 자동 Check Mode에서 조회가 skip되면 Assert가 실패할 수 있다. 아래 명령은 현재 검증된 Apply 절차로 확정하지 않는다.

```bash
ars playbooks/lb_haproxy.yml --limit lb-01
```

현재 Revision의 Check Mode 동작을 확인하고, 중단되면 Role·Runner를 정합화한 후 Actual Apply로 진행한다. 과거 HAProxy Runtime Evidence는 현재 Runner 호환성을 증명하지 않는다.

### 9.3 적용 후·Consumer 확인

사전 점검과 Apply가 완료됐다면 설정 문법, 서비스 active, `:80`·`:443`·`:6443`·`:3306` Listener를 다시 확인한다. `/`·`/api/v1`·`/ws/v1`의 Route는 Kubernetes Gateway·HTTPRoute 결과로, DB TLS·Query는 Database·Application 결과로 연결한다. Kubernetes API FQDN Client TLS는 SAN·게시 보류와 분리한다. Repository에서 `nc` 설치를 보장하지 않으므로 필수 명령으로 지정하지 않는다.

```bash
systemctl is-active haproxy
haproxy -c -f /etc/haproxy/haproxy.cfg
ss -lnt
```

Listener가 하나 빠져 있거나 서비스가 비활성화됐다면 다른 Backend 성공으로 이를 덮지 않는다. 변경 전 정상 설정과 현재 Template을 대조하고, HAProxy Config 복귀와 이전에 개방한 Firewall Port의 제거 여부를 각각 확인한다.

## 10. Endpoint·Client 연결 — Gate D

### 10.1 Endpoint 계약

Source는 실행 Revision의 `inventory/group_vars/all/vars.yml`이다.

| Endpoint | 주소·Port | Host | CoreDNS | Client | 상태 |
| --- | --- | ---: | ---: | ---: | --- |
| `harbor.seokpan.soldesk.store` | `192.168.53.61:443` | Yes | Yes | No | `active` |
| `db.seokpan.soldesk.store` | `10.1.93.90:3306` | Yes | Yes | No | `active` |
| `game.seokpan.soldesk.store` | `10.1.93.90:80/443` | Yes | Yes | Yes | `active` |
| `grafana.seokpan.soldesk.store` | `10.1.93.90:443` | Yes | Yes | Yes | `runtime_pending` |
| `k8s-api.seokpan.soldesk.store` | `10.1.93.90:6443` | No | No | Yes | `pending_api_san` |
| `jenkins.seokpan.soldesk.store` | `10.1.93.90:443` | No | No | Yes | `deferred_route` |
| `argocd.seokpan.soldesk.store` | `10.1.93.90:443` | No | No | Yes | `deferred_external_ui` |

변수 정의, 게시, 이름 해석과 실제 접속은 서로 다른 단계다.

### 10.2 common_hosts 적용 범위

`playbooks/common_hosts.yml`은 `hosts: all`의 `common_hosts` Role을 적용한다. 기존 Legacy Harbor mapping 정리, 운영 Alias인 `common_host_entries`, `host_publish=true` Endpoint를 함께 관리한다. 적용 전 대상과 `/etc/hosts` 전체 변경 범위를 확인한다.

운영 Alias에는 `vr1`, `cp1`, `wk1`, `lb1`, `ans` 같은 이름이 포함될 수 있다. 따라서 `common_hosts.yml`을 `game`·`db`·`harbor`만 추가하는 Playbook으로 해석하지 않는다. 현재 Alias와 Endpoint의 정확한 IP 목록은 실행 Revision의 `common_host_entries`·`project_endpoints`에서 확인한다.

```bash
ars playbooks/common_hosts.yml --inspect-only
ars playbooks/common_hosts.yml --limit <대상_VM_또는_그룹>
```

일반 실행은 Safe Runner 사전 점검의 변경 범위를 확인한 뒤 승인한다. `grafana`는 `runtime_pending`이지만 `host_publish=true`이므로 `/etc/hosts` 게시 대상에 포함될 수 있다. 게시 성공은 Grafana Runtime 검증이 아니다.

대상 VM에서 확인한다.

```bash
cat /etc/hosts
getent hosts game.seokpan.soldesk.store
getent hosts db.seokpan.soldesk.store
getent hosts harbor.seokpan.soldesk.store
getent hosts grafana.seokpan.soldesk.store
```

`k8s-api.seokpan.soldesk.store`는 API Server 인증서 SAN 정합화 전이라 Host·CoreDNS 게시가 보류된다. VIP `:6443` Listener로 FQDN Client TLS를 PASS 처리하지 않는다. Windows Host의 수동 `hosts` 설정은 해당 Client의 이름 해석 수단이며 Public DNS 게시를 의미하지 않는다.

검증 순서는 Endpoint 변수 → 해당 Host·CoreDNS 게시 → Client 이름 해석 → VIP·Route → HAProxy Listener → Backend·Application 응답이다. 각 단계의 성공과 다음 단계의 성공을 구분해 기록한다.

## 11. VRouter·LB Metric Network — Gate E

### 11.1 VRouter :9100

Exporter의 로컬 응답과 실제 Worker 출발지의 응답을 구분한다.

대상 VRouter에서:

```bash
curl -fsS http://127.0.0.1:9100/metrics >/dev/null
```

실제 Consumer 출발지에서 실행한다.

```bash
curl -fsS http://192.168.51.10:9100/metrics >/dev/null
curl -fsS http://192.168.52.10:9100/metrics >/dev/null
curl -fsS http://192.168.53.10:9100/metrics >/dev/null
curl -fsS http://192.168.54.10:9100/metrics >/dev/null
```

이 주소는 사설 Gateway이며 VRouter 외부 관리 IP `10.1.93.7x`가 아니다. 출발지·Route·대상 유입 NIC와 zone·Source Rich Rule을 함께 확인한다.

`curl`이 로컬에서 성공하고 Worker에서 실패하면 exporter 설치 상태만 다시 확인하지 않는다. Worker Source IP와 대상의 실제 유입 NIC·zone을 확인하고, `external` zone의 `9100/tcp` 허용 Source가 해당 패킷과 일치하는지 본다. Worker에서 HTTP 응답을 받은 후에도 Prometheus Target·Query 결과는 Observability 담당의 별도 확인을 기다린다.

### 11.2 lb-01 :8404와 인계

HAProxy 네이티브 Metric은 서비스 VIP 대신 관리 IP `10.1.93.78:8404`를 사용한다. `lb-01`과 필요한 원격 출발지에서 확인한다.

```bash
curl -fsS http://10.1.93.78:8404/metrics >/dev/null
```

EndpointSlice·NetworkPolicy·Prometheus Target·Query·Dashboard·Alert는 Delivery / Observability 경계다. 과거 `node-exporter-external 7/7 UP`은 VRouter 4대·MaxScale·NFS·Ansible Controller의 당시 결과이며, `lb01-haproxy-exporter UP`은 별도 결과다. 현재 재측정으로 표현하지 않는다.

LB에서 로컬 Metric이 응답해도 실제 Consumer가 `10.1.93.78:8404`에 도달한다는 근거는 아니다. 원격 출발지·Port·응답 코드와 당시 설정 Revision을 기록해 Observability 영역으로 인계한다.

## 12. Chrony — Gate F

`playbooks/chrony.yml`은 `hosts: all`에 `chrony` Role을 적용한다. 공통 Source는 `pool 2.centos.pool.ntp.org iburst`다. RPM Patch Version의 일치 대신 설정, `chronyd` active/enabled, `NTPSynchronized=yes`, Leap status Normal, 선택된 Source를 확인한다.

대상 VM에서 변경 전후를 조회한다.

```bash
rpm -q chrony
systemctl is-active chronyd
systemctl is-enabled chronyd
grep -E '^[[:space:]]*(server|pool)[[:space:]]+' /etc/chrony.conf
timedatectl show --property=NTPSynchronized --value
chronyc tracking
chronyc sources -v
```

```bash
ars playbooks/chrony.yml --inspect-only
ars playbooks/chrony.yml --limit <대상_VM_또는_그룹>
```

현재 Role의 명시적 `check_mode: false` 조회 Evidence는 Runner에 등록돼 있다. 일반 실행에서도 대상·Credential·사전 점검·Apply 승인 후 진행한다. Role은 문제 유형에 맞춰 설치·설정·서비스 상태를 보정하고 동기화를 재확인한다. 과거 Harbor 포함 16대의 결과를 이번 대상의 PASS로 복사하지 않는다.

적용 후에는 위 명령을 다시 실행해 설정의 NTP Source, 서비스 active/enabled, `NTPSynchronized=yes`, `chronyc tracking`의 Leap status 및 `chronyc sources -v`의 선택된 Source를 함께 확인한다. 서비스만 active이고 실제 동기화가 되지 않았다면 Source 접근·설정·tracking을 점검한다. 재실행 시 정상 VM에서 Package·Config 변경 또는 불필요한 Service Restart가 반복되는지 Recap과 서비스 상태를 함께 기록한다.

## 13. 장애 유형별 Recovery

| 증상 | 먼저 확인 | 기본 조치 |
| --- | --- | --- |
| `ars` 사용자·Credential 차단 | 일반 사용자, sudo 환경, Keyring | 허용된 계정과 사용자별 등록 상태 확인. 값 노출 금지 |
| Python 버전 불일치 | 3.12.13, `.venv` | 임의 삭제·우회 없이 `ars-setup` 경로 확인 |
| 미검토 `check_mode: false` 또는 Check Mode 조회 누락 | Task, SHA, Evidence, 후속 조건 | Apply 중단, Role·Runner 정합화 |
| VRouter NIC 탐지·사설망 통신 실패 | `ansible_host`, `.10`, Forwarding, 양방향 Route | VM NIC·IP와 Runtime·Persistent Route 대조 |
| Route가 매번 변경 | 표현 형식, Inventory 목표 | 목표·현재 값 비교 |
| Firewall 이후 SSH 위험 | 유입 NIC·zone, SSH Rule, Console | 확대 중단, 독립 접근 확보 |
| 원본 Source IP가 NAT된 것으로 관찰 | 양방향 경로, Masquerade | 규칙과 수신 측 Source 비교 |
| VRouter :9100 로컬 성공·원격 실패 | Worker→사설 Gateway, zone, Source | external Rich Rule·실제 경로 확인 |
| `lb-01` VIP 실패·영구값 불일치 | 관리 IP, VIP Prefix, Route 다음 홉 | 자동 교정 가정 없이 수정 절차 결정 |
| HAProxy Check 실패·Listener 없음 | 조회·Assert, 문법, 서비스, VIP | 정합화 후 문법·서비스·로그 확인 |
| Listener만 정상, 서비스 실패 | Backend·Consumer | Kubernetes·Database 담당 결과 연결 |
| 이름 해석 또는 API FQDN TLS 실패 | 게시 값, `common_hosts`, SAN | 보류 상태와 실제 게시 경계 확인 |
| Chrony 동기화 실패 | Source·설정·서비스·tracking | 원인 확인 후 제한 대상 재적용 판단 |

장애 시 VIP TCP → Listener → Backend → Application HTTP·API·WebSocket과 DB TCP → TLS → Login → Query → Replication 중 어디까지 확인됐는지 기록한다. SSH Bootstrap의 사용자·VM별 상태 점검 및 누락 공개키 선택적 복구는 Infra Issue #27의 Planned 작업이다. 현재 Runbook이 Snapshot Rollback이나 재구성 뒤 SSH를 자동 복구한다고 기록하지 않는다.

장애를 좁힐 때는 같은 Endpoint를 두 출발지에서 비교할 수 있다. 예를 들어 VRouter 로컬 `:9100`은 응답하고 Worker에서 실패하면 대상 서비스뿐 아니라 Worker Route·Source CIDR·유입 zone을 확인한다. HAProxy Listener는 열렸지만 `game` 경로가 실패하면 Backend NodePort와 Kubernetes Gateway·Application 결과를 분리한다. DB `:3306` TCP가 성공해도 TLS·Login·Query·Replication을 각각 확인해야 한다.

원격 변경으로 SSH가 끊긴 경우에는 다른 VRouter나 LB로 Apply를 확대하지 않는다. 독립 Console 접근이 확보되기 전에는 반복 원격 실행을 시도하지 않고 15장의 변경 전 상태·현재 목표 대조 순서를 따른다.

## 14. 재실행·멱등성 기준

### 14.1 공통 판정

동일 Revision·Inventory·대상에서 정상 상태의 2회차 Actual Run과, 의도적으로 관리 값을 제거한 뒤 재실행해 복원하는 시험은 별개다. 각 실행마다 `ars`의 사전 점검·승인을 다시 거친다. Recap만 보지 않고 해당 설정의 Runtime·Persistent 상태와 Consumer 결과를 필요한 범위에서 대조한다.

### 14.2 Gate B — VRouter Network

과거 PR #134에서는 이미 목표 상태인 VRouter 네 대의 Actual Run 1·2차에서 `changed=0`, `failed=0`, `unreachable=0`을 확인했다. 별도로 `vrouter-01`의 누락 Persistent Route 복원도 확인됐다. 이 기록은 당시 Revision의 결과다.

현재는 6.3절의 Runner Evidence를 정합화하고 제한 대상 첫 Actual Run을 확인한 후 재실행한다.

```bash
ars playbooks/vrouter_network.yml --limit <대상_VRouter>
```

Forwarding·Runtime Route·Persistent Route가 목표와 맞고 재실행 때 변경이 없다면 해당 대상의 멱등성을 기록한다. 누락 Route 복원은 별도 Test Run으로 남긴다.

### 14.3 Gate C — VRouter Firewall

현재 Revision에서 자동 Check Mode와 제한 대상 첫 Apply가 통과한 뒤 같은 대상에 재실행한다.

```bash
ars playbooks/vrouter_firewall.yml --limit <대상_VRouter>
```

NIC zone, 두 Policy, SSH 허용, Masquerade, `9100/tcp` Worker Source Rule의 불필요한 반복 변경 여부를 확인한다. Apply Recap과 `firewall-cmd`의 Runtime·Permanent 결과를 같이 기록한다. Check Mode 성공이나 `changed=0` 하나만으로 실제 정책 검증을 마치지 않는다.

### 14.4 Gate D — lb-01 Network·HAProxy

`lb_network` 전체의 Safe Runner 사전 점검과 첫 Actual Run을 확인한 뒤에만 재실행한다.

```bash
ars playbooks/lb_network.yml --limit lb-01
```

VIP·Route가 이미 목표 상태라면 불필요한 변경이 없어야 한다. `changed=0`이어도 영구 Route의 다음 홉과 VIP의 Prefix가 목표와 같은지 8.2절의 조회 결과로 확인한다. Role이 필요한 Route·VIP를 추가하는 방식은 잘못된 값의 자동 교정이나 제거·Rollback을 보장하지 않는다.

`lb_haproxy`는 9.2절의 조회·Assert Check Mode 정합화 전까지 다음 명령을 검증된 재실행 절차로 확정하지 않는다.

```bash
ars playbooks/lb_haproxy.yml --limit lb-01
```

정합화 이후 첫 Actual Run과 정상 상태 2회차, 문법·서비스·Listener 결과를 나누어 기록한다.

### 14.5 Gate F — Chrony

```bash
ars playbooks/chrony.yml --limit <대상_VM_또는_그룹>
```

정상 VM에서 불필요한 Package·Config 변경이나 Service Restart가 반복되지 않는지 확인하고, 재실행 당시의 실제 시간 동기화도 기록한다. 과거 Harbor 포함 16대와 Metric Target 결과를 이번 실행의 Runtime 결과로 복사하지 않는다.

## 15. Recovery / Rollback Matrix

Recovery는 현재 Desired State로의 정상화, Rollback은 이전 정상 상태로의 복귀다. Playbook 재적용은 해당 Revision의 Safe Runner 사전 점검을 통과한 경우에만 진행한다.

| 변경 대상 | 현재 자동화·Recovery 범위 | Rollback 시 주의사항 | 복구 후 확인 |
| --- | --- | --- | --- |
| Project `.venv`·`ars` | 고정 버전·alias·checkout 정합 확인 | 기존 `.venv` 임의 삭제 금지 | Python·Ansible 버전, `type ars` |
| VRouter IPv4 Forwarding | 현재 목표값으로 정합화. Runner Evidence 정합화 후 재적용 | 이전 값은 당시 정상 Revision과 대조 | `sysctl net.ipv4.ip_forward` |
| VRouter Runtime Route | 현재 목표 Route를 replace. Runner 정합화 후 재적용 | 목표에서 제거된 Route의 자동 삭제를 보장하지 않음 | `ip -4 route` |
| VRouter Persistent Route | 목표 Route 목록과 비교·정합화 | 변경 전·후 전체 Route 목록과 다음 홉을 대조 | `nmcli -g ipv4.routes connection show <Connection_Name>` |
| VRouter Firewall zone·Policy | 현재 Role의 정책 정합화. Check Mode·Apply 확인 후 재적용 | 이전 Role에서 사라진 Rule의 자동 제거를 보장하지 않음 | Runtime·Permanent zone, Policy 상세 |
| VRouter 9100 Rich Rule | 필요한 Source Rule 관리 | Task 제거·과거 Revision 재적용만으로 기존 Rule 자동 삭제를 보장하지 않음 | Runtime·Permanent `--list-rich-rules` |
| `lb-01` Route | 현재 필요한 Route 추가·관리 | `+ipv4.routes` 항목 자동 제거와 잘못된 다음 홉의 자동 교정을 보장하지 않음 | `ip -4 route`, `nmcli`의 목적지·다음 홉 |
| `lb-01` VIP | 현재 필요한 VIP 추가·관리 | `+ipv4.addresses` 항목 자동 제거와 잘못된 Prefix의 자동 교정을 보장하지 않음 | `ip -br address`, `nmcli`의 주소·Prefix |
| HAProxy Config | Check Mode·Apply 정합화 후 정상 Template 재배포 | Config 복귀와 과거 Firewall Port 제거는 별도 작업 | `haproxy -c`, 서비스, `ss -lnt` |
| `/etc/hosts` 관리 Block | 현재 Alias·Endpoint Block 정합화 | Legacy mapping 정리와 전체 관리 Block 영향 대조 | `/etc/hosts`, `getent hosts` |
| Chrony | 현재 공통 Source·서비스 정합화 | 이전 설정 복귀는 당시 정상 Revision·백업 확인 | active/enabled, Sync, Tracking |

### 15.1 Route·Firewall 장애 시

원격 변경으로 SSH가 끊기면 다른 대상의 Apply를 멈춘다. VMware Console 등 독립 접근으로 현재 NIC·Route·Firewall의 Runtime·Persistent 값을 읽고, 변경 전 기록 및 실행 Revision의 목표와 대조한다. 원인을 확인한 뒤 해당 변경을 복구하거나 검증된 제거 절차로 이전 상태에 돌린다. 마지막으로 SSH와 필요한 Consumer 경로를 다시 확인한다. Console 접근이 없다면 안전한 원격 복구 경로를 확보하기 전까지 같은 명령을 반복하지 않는다.

### 15.2 Git Revert와 Runtime Rollback

Git에서 이전 Revision으로 돌아가도 추가된 firewalld Rich Rule, LB Persistent Route·VIP, HAProxy Firewall Port가 자동 삭제됐다고 판단하지 않는다. 특히 `+ipv4.routes`·`+ipv4.addresses`로 추가한 값과 현재 Revision에서 사라진 Rule의 제거 여부를 별도로 확인한다. Rollback 후 Runtime과 Persistent 상태를 모두 조회하고 실제 Client·Consumer 접속을 재확인한다.

## 16. 12 검증·측정 계획으로의 인계

11은 실행·확인 순서를 제공하고 정식 Test Case·PASS/FAIL은 12가 관리한다. 매 실행에 시각·사용자·Repository·Branch·SHA·Playbook·`--limit` 대상·사전 상태·Safe Runner/Check Mode·Apply 여부·첫 실행/재실행 Recap·변경 후 Runtime/Persistent 상태·Consumer 결과·미검증 범위·관련 Issue/PR을 기록한다. Credential 값은 제외한다.

VRouter Route·Forwarding·Firewall·NAT의 출발지→목적지 통신, Source·Port·원본 IP 결과는 Kubernetes·Database·Delivery의 Consumer 결과와 연결한다. VIP `:80/:443`은 HTTPS·Backend API·WebSocket, `:3306`은 DB FQDN·TLS·Query, `:6443`은 SAN 정합화와 이름 게시 이후 Kubernetes API Client 결과로 인계한다. VRouter 사설 Gateway `:9100`과 LB 관리 IP `10.1.93.78:8404`의 원격 접근은 Delivery / Observability의 Target·Query와 연결한다. Chrony는 실행 대상 VM의 실제 동기화 결과를 남긴다.

이 Runbook만으로 Clean Build, 모든 사용자×Playbook Apply, SSH Bootstrap #27, LB 자동 전환, API FQDN Client TLS, 3장에 기록한 현재 Safe Runner 미완료 경로, P4 전체 Acceptance를 PASS 처리하지 않는다.

## 17. Runbook 완료 기준

- 실제 Infra Inventory·Role·Playbook과 실행 대상·명령·변수가 일치한다.
- 명령의 실행 위치·계정·대상, 사전 점검·승인·중단 조건이 명확하다.
- 설정과 Consumer 응답, 과거 Evidence와 현재 실행, 첫 적용과 멱등성·Drift 복원·Rollback을 구분한다.
- 장애 시 중단·독립 접근·Recovery·Rollback 판단 경로가 있다.
- 미완료 SSH Bootstrap·Clean Build·LB 자동 전환·API FQDN·Runner 경계를 완료로 표현하지 않는다.
- Revision·대상·시각·전후 결과를 12 문서로 인계한다.

문서 작성 완료와 Network / Ansible / External Infra 전체 Acceptance는 별개다. Revision이나 구성이 바뀌면 해당 Test Run을 12에 기록하고 본 Runbook의 실행 절차를 현행화한다.
