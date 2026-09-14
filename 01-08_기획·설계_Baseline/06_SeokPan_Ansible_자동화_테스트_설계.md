# 石나가는 판단 - Ansible 자동화·테스트 설계

On-premise 원 목표 구조를 반복 구축·재실행·복구·검증 가능한 실행 설계로 변환한다.

> **문서 기준** Physical Server 4대·VM 18개의 원 목표 구조를 기준으로 한다. 07에서 도출할 MVP 축소안은 본문 자동화 기준에 함께 사용하지 않으며, 실제 구현·측정 전의 값은 시작값 또는 검증 기준으로만 표현한다.

이 문서는 최신 서비스 계약과 논리·물리 아키텍처를 Inventory, Variable, Role, Playbook, Validation Gate, Recovery Procedure와 Evidence로 연결하여 두 개발자가 같은 입력으로 같은 결과를 재현할 수 있게 하는 실행 설계다.

## 목차

1. [문서 목적·범위·완료 조건](#section-1)  
2. [선행 설계 입력과 관리 책임 경계](#section-2)  
3. [자동화 대상·Inventory·Failure Domain](#section-3)  
4. [Variable·Secret·인증서 소유권](#section-4)  
5. [Version·설치 자산·Repository 정책](#section-5)  
6. [Ansible 프로젝트 구조·Role·Playbook](#section-6)  
7. [수동 공통 기반·CentOS Guest·VRouter·Firewall 자동화](#section-7)  
8. [Common VIP·HAProxy·Keepalived·TLS 자동화](#section-8)  
9. [Kubernetes·Calico·Gateway·Cluster Add-on 자동화](#section-9)  
10. [MariaDB·MaxScale·Redis·ANALYSIS·NFS 자동화](#section-10)  
11. [Jenkins·Harbor·Argo CD CI/CD·GitOps](#section-11)  
12. [Prometheus·Loki·Grafana·Alloy·Alertmanager](#section-12)  
13. [멱등성·부분 재실행·Drift·Recovery](#section-13)  
14. [설치 후 Validation Gate](#section-14)  
15. [장애·Failure Domain·복구 검증](#section-15)  
16. [부하·HPA·실시간·ANALYSIS 검증](#section-16)  
17. [Backup/Restore·Before/After·Evidence](#section-17)  
18. [추적 Matrix·구현 체크리스트·07 이관 경계](#section-18)  

<a id="section-1"></a>

## 1. 문서 목적·범위·완료 조건

자동화의 목적은 18개 VM에 Command를 빠르게 실행하는 것이 아니라, 동일한 물리·논리 구조를 반복 일치시키고 부분 실패 후 필요한 범위만 재실행하며, 서비스 경로와 장애 복구 결과를 정량 증거로 남기는 데 있다.


<a id="section-1-1"></a>

### 1.1 해결할 문제

- 수동 공통 기반 이후 CentOS Guest·Network·Kubernetes·Data·Delivery 설정의 편차와 누락을 감소시킨다.

- Stable API Endpoint, Routing, Storage, DB, GitOps 등 선행관계를 실행 순서로 고정한다.

- 정상 일치와 위험한 Recovery를 분리하여 자동화가 데이터를 임의 삭제하거나 Cluster를 초기화하지 않게 한다.

- HA·HPA·DR·CI/CD·관측성·실시간 처리·ANALYSIS 비권위 경계를 같은 Test Case 형식으로 검증한다.

- 수동 구축과 자동 구축을 동일 조건에서 비교하고 Raw Evidence와 Git Commit을 연결한다.


<a id="section-1-2"></a>

### 1.2 자동화 출발점과 경계

| **구분**            | **본 문서 기준**                                                                           | **판정**                                  |
|---------------------|--------------------------------------------------------------------------------------------|-------------------------------------------|
| Physical Server     | Windows 10 Host·VMware Workstation·VMnet·VM Hardware·CentOS 설치·초기 SSH의 수동 공통 기반 | Runbook·검증 증거 대상, Ansible 실행 제외 |
| VM 생성             | VMware에서 VM Hardware·Thin Disk·vNIC를 수동 생성하고 실제 VMnet·Interface 값을 기록       | Ansible 자동화 범위 제외                  |
| VM OS 설치          | SSH 가능한 CentOS Stream 9 상태를 기본 출발점으로 사용                                     | PXE·Kickstart는 기본 제외                 |
| Kubernetes Workload | Argo CD가 지속 Desired State 소유                                                          | Ansible 직접 상시 배포 제외               |
| Recovery            | reset·promotion·restore는 별도 Playbook과 승인 Flag                                        | 정상 site.yml에서 제외                    |


<a id="section-1-3"></a>

### 1.3 완료 조건

> **완료 판정** Playbook 종료 코드 0만으로 완료하지 않는다. Component·Integration·E2E Validation Gate가 통과하고, 재실행·부분 실패·Drift·복구 결과와 Evidence가 남아야 Experiment Ready로 판정한다.

<a id="section-2"></a>

## 2. 선행 설계 입력과 관리 책임 경계

| **선행 문서**                | **06으로 변환하는 핵심 계약**                                                                                                              |
|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| 서비스 요구사항 및 기능 명세 | Room·Game·Turn·Vote·Move·Result의 서버 권위, 재접속 Snapshot, 중복 요청 멱등성, 실시간 비율와 공식 착수                                    |
| 핵심 문제 및 검증 목표       | HA·HPA·DR·CI/CD·Monitoring/Logging·실시간 처리·Ansible Before/After                                                                        |
| 논리 역할 및 서비스 목록     | WEB·Application·ANALYSIS·STATE·DATA·MONITORING·LOGGING·AUTOMATION 책임 분리                                                                |
| 기술 비교 및 논리 아키텍처   | kubeadm·Calico·Gateway API·MariaDB/MaxScale·Redis·NFS·Jenkins/Argo CD·Harbor·Prometheus/Loki/Alloy                                         |
| 물리 아키텍처                | Physical Server 4대·VM 18개, Windows Host·VMware 수동 기반, Common VIP, 사설 Subnet·Static Route, Storage·Failure Domain·외부 지원 Host/VM |


<a id="section-2-1"></a>

### 2.1 도구별 단일 소유권

| **Owner**  | **소유 영역**                                                                                                                         | **소유하지 않는 영역**                                              |
|------------|---------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| Ansible    | SSH 가능한 CentOS Guest OS, VM 내부 Network, Kubernetes Bootstrap, External Infra, CA/TLS 주입, Argo CD Bootstrap, Validation·Rebuild | Windows Host·VMware·VM 생성과 GitOps 이후 Application Desired State |
| Jenkins    | Source Checkout, Test, Rootless Image Build, Harbor Push, GitOps 변경 제안                                                            | Cluster 직접 kubectl/helm 배포                                      |
| Argo CD    | Gateway, Storage Client, Runtime, Application, Observability의 선언 상태·Sync·Self-Heal                                               | Host OS·외부 DB/NFS 구성                                            |
| Kubernetes | Scheduling, Replica, Health Probe, Pod Restart, Service Endpoint                                                                      | VM·LB·DB 승격·NFS Disk 복구                                         |
| 운영자     | Fencing 확인, DB 승격 승인, 위험한 Restore 승인, 성공 기준 동결                                                                       | 반복 가능한 정상 설정의 수동 유지                                   |

```text
Ansible Bootstrap → Argo CD Desired State → Kubernetes Runtime Self-Healing
Jenkins Build/Test → Harbor Image → GitOps PR/Merge → Argo CD Sync
```

<a id="section-3"></a>

## 3. 자동화 대상·Inventory·Failure Domain


<a id="section-3-1"></a>

### 3.1 Node·Role·IP·Group 단일 기준표

| **Host**    | **배치**                                 | **관리 IP**                | **Inventory Group**                | **역할**                                      |
|-------------|------------------------------------------|----------------------------|------------------------------------|-----------------------------------------------|
| server-01   | Windows Host (Topology)                  | 10.1.93.70                 | topology_hosts / fd_server_01      | VMware·VMnet 수동 기반; Ansible 비실행        |
| server-02   | Windows Host (Topology)                  | 10.1.93.72                 | topology_hosts / fd_server_02      | VMware·VMnet 수동 기반; Ansible 비실행        |
| server-03   | Windows Host (Topology)                  | 10.1.93.74                 | topology_hosts / fd_server_03      | VMware·VMnet 수동 기반; Ansible 비실행        |
| server-04   | Windows Host (Topology)                  | 10.1.93.76                 | topology_hosts / fd_server_04      | VMware·VMnet 수동 기반; Ansible 비실행        |
| vrouter-01  | Server-01 VM                             | 10.1.93.71 / 192.168.51.10 | vrouters / fd_server_01            | Routing                                       |
| cp-01       | Server-01 VM                             | 192.168.51.20              | control_plane / fd_server_01       | K8s CP                                        |
| worker-01   | Server-01 VM                             | 192.168.51.30              | workers / fd_server_01             | K8s Worker                                    |
| mariadb-02  | Server-01 VM                             | 192.168.51.40              | mariadb_replica / fd_server_01     | Replica                                       |
| vrouter-02  | Server-02 VM                             | 10.1.93.73 / 192.168.52.10 | vrouters / fd_server_02            | Routing                                       |
| cp-02       | Server-02 VM                             | 192.168.52.20              | control_plane / fd_server_02       | K8s CP                                        |
| worker-02   | Server-02 VM                             | 192.168.52.30              | workers / fd_server_02             | K8s Worker                                    |
| mariadb-01  | Server-02 VM                             | 192.168.52.40              | mariadb_primary / fd_server_02     | Primary                                       |
| vrouter-03  | Server-03 VM                             | 10.1.93.75 / 192.168.53.10 | vrouters / fd_server_03            | Routing                                       |
| cp-03       | Server-03 VM                             | 192.168.53.20              | control_plane / fd_server_03       | K8s CP                                        |
| lb-01       | Server-03 VM                             | 10.1.93.78                 | load_balancers / fd_server_03      | VIP/L4                                        |
| maxscale-01 | Server-03 VM                             | 192.168.53.40              | maxscale / fd_server_03            | DB Proxy                                      |
| harbor      | Server-03 VM                             | 192.168.53.61              | harbor / fd_server_03              | Registry                                      |
| vrouter-04  | Server-04 VM                             | 10.1.93.77 / 192.168.54.10 | vrouters / fd_server_04            | Routing                                       |
| lb-02       | Server-04 VM                             | 10.1.93.79                 | load_balancers / fd_server_04      | VIP/L4                                        |
| maxscale-02 | Server-04 VM                             | 192.168.54.40              | maxscale / fd_server_04            | DB Proxy                                      |
| nfs         | Server-04 VM                             | 192.168.54.50              | nfs_server / fd_server_04          | Shared NFS                                    |
| ansible     | Server-04 VM                             | 192.168.54.70              | ansible_controller / fd_server_04  | Automation                                    |
| loadgen     | 5번째 Windows Host(.92)의 CentOS 지원 VM | 10.1.93.91                 | external_support / load_generators | k6·Backup·Evidence·긴급 SSH; 서비스 18VM 제외 |

> **수량 규칙** 서비스 실행 Physical Server 4대와 VM 18개를 별도로 계산한다. 5번째 Windows Host 10.1.93.92와 loadgen VM 10.1.93.91은 외부 지원 환경으로 제외하고, 10.1.93.93~99는 확장 여유로 보존하며 6번째 PC는 사용하지 않는다. Kubernetes의 Gateway·Frontend·Backend·Redis·ANALYSIS·Jenkins·Argo CD·Observability Pod는 VM 수에 포함하지 않는다.


<a id="section-3-2"></a>

### 3.2 Inventory 구조

```text
inventories/phase1_target/
├── hosts.yml
├── group_vars/
│   ├── all.yml
│   ├── kubernetes.yml
│   ├── load_balancers.yml
│   ├── mariadb.yml
│   ├── load_generators.yml
│   └── vault.yml              # encrypted
└── host_vars/                 # Guest Interface·IP·Gateway 등 실제 확인값
```

기능 Group과 Physical Server Failure Domain Group은 동시에 사용한다. 예를 들어 worker-01은 workers와 fd_server_01에 함께 속한다. Windows Host는 Ansible 실행 대상이 아니라 VM 배치와 Failure Domain을 표현하는 Topology 정보로만 둔다. loadgen은 5번째 Windows Host 10.1.93.92에서 실행하는 CentOS Stream 9 Bridged VM 10.1.93.91이며 external_support와 load_generators에 속하지만 서비스 Physical Server 4대·VM 18개 산정에는 포함하지 않는다.

<a id="section-4"></a>

## 4. Variable·Secret·인증서 소유권

| **위치**            | **소유 값**                                                 | **원칙**                           |
|---------------------|-------------------------------------------------------------|------------------------------------|
| role/defaults       | 범용·안전 기본값                                            | 주소·계정·Password를 두지 않는다.  |
| group_vars/all      | Domain, Common VIP, CIDR, 공통 Repository                   | 프로젝트 공통 단일 기준            |
| group_vars/\<role\> | Port, Package, Storage, Health Check                        | 기능 Group 공통값                  |
| host_vars           | Guest Interface, VM IP, Gateway, Route                      | 실제 확인한 CentOS Guest 고유값만  |
| Vault               | Password, Token, TLS Private Key, SMTP, Registry Credential | encrypted·no_log·최소 노출         |
| Runtime Fact        | kubeadm Token, Certificate Key, 임시 승인값                 | 실행 시 생성 후 장기 저장하지 않음 |

```text
common_vip: 10.1.93.90
service_fqdn: service.stone.test
k8s_api_fqdn: k8s-api.stone.test
db_fqdn: db.stone.test
pod_cidr: 10.244.0.0/16
service_cidr: 10.96.0.0/12
gateway_http_nodeport: 30080
gateway_https_nodeport: 30443
```

<a id="section-4-1"></a>

### 4.1 Secret 경계

- MariaDB Admin·Replication·Application 계정과 MaxScale Monitor 계정은 Vault가 소유한다.

- Harbor Robot Credential, Git Deploy Key, Jenkins Credential, SMTP Password와 TLS Private Key는 평문 Inventory·Evidence에 남기지 않는다.

- Kubernetes Secret의 실제 값은 Ansible이 최초 주입하거나 승인된 Secret 전달 절차가 주입하며, Argo CD는 Secret 참조만 관리한다.

- Internal Root CA와 Kubernetes Cluster CA는 분리한다. CA 재생성은 정상 재실행이 아니라 Rotation/Recovery 작업이다.

- Vault Password 파일은 Git에 저장하지 않고 Controller 외부의 팀 승인 경로로 전달한다.

<a id="section-5"></a>

## 5. Version·설치 자산·Repository 정책

모든 핵심 구성요소는 GA/Stable Release 또는 OS 저장소의 고정 Package로 동결한다. candidate·RC·alpha·beta·nightly·latest Tag는 사용하지 않는다. Container Image는 Tag뿐 아니라 Digest를 기록하고, RPM은 NEVRA와 Repository Snapshot 정보를 Evidence에 남긴다.

| **Component**          | **Baseline**                | **고정 자산**                   | **호환 대상**             | **선택 근거**                          | **업그레이드 기준**                 |
|------------------------|-----------------------------|---------------------------------|---------------------------|----------------------------------------|-------------------------------------|
| Kubernetes             | v1.36.2                     | pkgs.k8s.io minor + NEVRA       | Calico·Metrics Server     | 1.36 최신 Patch 기준                   | 보안/호환 Patch만 회귀 후           |
| Calico                 | v3.32.1                     | Operator Manifest + Digest      | Kubernetes 1.36           | 1.34~1.36 지원 범위                    | K8s minor 변경 시 재검증            |
| containerd             | CS9 검증 NEVRA              | Repo Snapshot + NEVRA           | Kubernetes·systemd cgroup | OS 통합 운영                           | 보안 Fix·K8s 요구 시                |
| HAProxy/Keepalived     | CS9 검증 NEVRA              | Repo Snapshot + NEVRA           | VRRPv3·4 Listener         | OS 패키지 재사용                       | 보안 Fix 후 Failover 회귀           |
| MariaDB Server         | 11.8.9 LTS                  | 공식 Repo + NEVRA               | MaxScale 24.02·동일 Patch | LTS·복제/Backup                        | Primary/Replica 동시 Patch          |
| MaxScale               | 24.02.10                    | 공식 Repo + NEVRA               | MariaDB 11.8              | GA·Monitor/Router                      | 라이선스·DB 호환 확인 후            |
| Redis                  | 8.10.1 GA                   | 공식 Image Digest               | AOF·Streams               | 8.10 Security Patch·단일 Stateful 경계 | AOF/PVC 회귀 후                     |
| Jenkins                | 2.568.2 LTS                 | Controller Digest + Plugin Lock | JDK 21·K8s Plugin         | LTS·JCasC                              | LTS/Plugin 보안 Fix 시              |
| Harbor                 | v2.15.2                     | Offline Installer + SHA256      | OCI·Internal TLS          | 외부 Registry                          | DB/Redis 포함 Upgrade 검증          |
| Argo CD                | v3.4.7                      | Release Manifest + Digest       | Kubernetes 1.36·GitOps    | 3.4 Patch·RC 제외                      | Minor는 Upgrade Guide 후            |
| NGINX Gateway Fabric   | v2.6.7                      | OCI Chart + Digest              | Gateway API Standard      | NodePort Data Plane                    | CRD/API 호환 후                     |
| NFS Subdir Provisioner | Chart 4.0.18 / Image v4.0.2 | Chart Lock + Image Digest       | NFSv4·Kubernetes 1.36     | 공식 Chart/Image 구분                  | Registry Pull·Digest·호환성 동결 후 |
| Metrics Server         | v0.9.0                      | components.yaml + SHA256        | Kubernetes 1.36.2         | HPA Resource Metrics                   | K8s minor 변경 시                   |
| Prometheus Stack       | Chart 88.5.4                | Chart Lock + Image Digests      | CRD·Prometheus Operator   | 공식 Stack 일괄 관리                   | CRD Backup·회귀 후                  |
| Loki                   | v3.7.6                      | Single Binary Digest            | Alloy 1.18·Local PV       | 3.7 Patch Fix                          | LogQL·Schema 회귀 후                |
| Grafana Alloy          | v1.18.1                     | Chart/RPM + Digest/NEVRA        | Loki 3.7·Linux Journal    | K8s/외부 Agent 통일                    | Pipeline 회귀 후                    |
| Grafana k6             | v2.1.0                      | Binary/Image Checksum           | 외부 loadgen              | HTTP/WSS/arrival-rate                  | Script 호환 회귀 후                 |

> **Freeze 절차** 구축 시작 시점에 공식 Release와 실제 다운로드 가능성을 다시 확인하고 version-lock.yml에 Tag, Digest/Checksum, RPM NEVRA, Chart Version, Source URL을 기록한다. 이 표의 Patch보다 새 Stable Patch가 존재해도 자동 변경하지 않으며, 보안·호환 필요성과 회귀 결과를 별도 PR로 남긴 뒤 동결한다.

<a id="section-6"></a>

## 6. Ansible 프로젝트 구조·Role·Playbook

```text
ansible/
├── ansible.cfg
├── requirements.yml
├── inventories/phase1_target/
├── roles/
├── playbooks/
│   ├── stages/
│   ├── components/
│   ├── recovery/
│   └── experiments/
├── validation/
├── templates/
├── files/
└── evidence/                 # Git에는 요약·metadata·checksum 중심
```

| **Role**                 | **주요 책임**                                                                                                                                                                                                                                   |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| preflight                | 수동 공통 기반 증거, SSH/become, CPU/RAM/Disk, IP 중복, DNS/NTP, Repository                                                                                                                                                                     |
| linux_base               | SSH 가능한 CentOS Guest의 Hostname, chrony, Package, 사용자·SSH, 공통 경로                                                                                                                                                                      |
| vrouter                  | Guest NIC, IP forwarding, 영구 Static Route, 외부망 전용 Masquerade, 사설망 Source IP·Return Path                                                                                                                                               |
| host_firewall            | Source·Destination·Protocol·Purpose 기반 최종 허용 상태                                                                                                                                                                                         |
| haproxy / keepalived     | Common VIP Listener·Backend·Health / VRRP·VIP Owner                                                                                                                                                                                             |
| container_runtime        | containerd, cgroup, Registry Trust                                                                                                                                                                                                              |
| kubernetes_prereq        | Kernel/sysctl, swap, kubeadm/kubelet/kubectl                                                                                                                                                                                                    |
| kubernetes_bootstrap     | First CP init, CP join serial, Worker join                                                                                                                                                                                                      |
| calico                   | Operator Bootstrap, IPPool, VXLAN, MTU, NetworkManager 예외                                                                                                                                                                                     |
| mariadb / maxscale       | DB 복제·Backup / Monitor·Router·Listener                                                                                                                                                                                                        |
| nfs_server               | NFSv4.2 Export, Permission, Backup Staging                                                                                                                                                                                                      |
| harbor                   | Offline Installer, TLS, Project·Robot·GC 기본                                                                                                                                                                                                   |
| internal_ca / tls_deploy | CA·SAN Certificate·Trust Store                                                                                                                                                                                                                  |
| argocd_bootstrap         | Argo CD 설치, Repository Credential, Root Application                                                                                                                                                                                           |
| alloy_linux              | CentOS Guest·외부 지원 Linux VM의 Journal·File 수집과 Loki 전송                                                                                                                                                                                 |
| node_exporter_linux      | CentOS Guest의 node_exporter 설치·고정 버전·systemd Service·9100/TCP Listen·Prometheus Scrape 대상 구성을 관리. Kubernetes Node의 node-exporter가 kube-prometheus-stack에 의해 관리되는 경우 중복 설치하지 않으며, Windows Host는 대상에서 제외 |
| backup_transfer          | MariaDB·etcd·Evidence SSH 전송, Checksum, Retention                                                                                                                                                                                             |
| validation               | Component·Integration·E2E Gate와 Evidence                                                                                                                                                                                                       |
| loadgen_support          | CentOS VM 10.1.93.91 /etc/hosts, k6 Checksum, Backup 제한 계정·별도 Virtual Disk·Evidence 경로, 긴급 SSH, 선택적 Alloy; Windows Host .92는 수동                                                                                                 |


<a id="section-6-1"></a>

### 6.1 전체 실행 순서

1. 00-preflight.yml: 수동 공통 기반 증거·접속·권한·주소·Repository·Disk·시간 동기화 사전 점검

2. 10-guest-baseline.yml: Windows Host·VMware 수동 공통 기반 완료 확인과 CentOS Guest Baseline

3. 20-network.yml: VM 내부 Network·VRouter·Static Route·Firewall·CentOS Guest/loadgen /etc/hosts·Network Gate

4. 30-endpoint.yml: HAProxy·Keepalived·Common VIP·CA/TLS·Endpoint Gate

5. 40-kubernetes.yml: Runtime·kubeadm Bootstrap/Join·Calico·CoreDNS Records·Cluster Gate

6. 50-data-storage.yml: MariaDB·MaxScale·NFS·Backup 준비

7. 60-delivery.yml: Harbor·Argo CD Bootstrap·Root Application

8. 70-platform.yml: Gateway·Provisioner·Redis·Jenkins·Kubernetes Observability Desired State Sync + 외부 CentOS Guest node_exporter_linux/alloy_linux 구성

9. 80-application.yml: Frontend·Backend·ANALYSIS·Route·HPA·NetworkPolicy Sync

10. 90-validation.yml: loadgen Support 동결→Component→Integration→E2E→Experiment Ready 판정

site.yml은 위 Stage를 import하는 단일 Entry Point다. Component Playbook은 담당 영역 개발과 부분 실패 재실행에 사용하고, Recovery·Experiment Playbook은 정상 구축과 분리한다.


<a id="section-6-2"></a>

### 6.2 Role/Playbook 의존성 단일 기준표

| **Stage**         | **선행 Gate**    | **Role/소유자**                                                      | **대상**                              | **실패·재시작 위치**                                                        | **Validation** |
|-------------------|------------------|----------------------------------------------------------------------|---------------------------------------|-----------------------------------------------------------------------------|----------------|
| 00 Preflight      | 없음             | preflight                                                            | CentOS Guest·loadgen + 수동 기반 증거 | Critical 실패 즉시 중단; 00부터                                             | G-01           |
| 10 Guest Baseline | G-01             | linux_base                                                           | SSH 가능한 CentOS Guest·loadgen       | 대표 대상 검증 후 역할 Group 적용; 실패 Host부터                            | G-01           |
| 20 Network        | 10 완료          | vrouter·host_firewall·loadgen_support                                | CentOS Guest·loadgen                  | 관리 경로 보존; 실패 Router/Guest부터                                       | G-02           |
| 30 Endpoint       | G-02             | haproxy·keepalived·internal_ca·tls_deploy                            | LB·대상 Host                          | Standby→Active; 30부터                                                      | G-03·F-01      |
| 40 Kubernetes     | G-03 :6443       | container_runtime·kubernetes_prereq·bootstrap·calico·coredns_records | CP·Worker·Cluster DNS                 | CP join serial:1; 실패 Node/Add-on부터                                      | G-04           |
| 50 Data/Storage   | G-02·G-04        | mariadb·maxscale·nfs_server·backup_transfer                          | DB·NFS·LB                             | Data 삭제 금지; Component부터                                               | G-05·DR-01~04  |
| 60 Delivery       | G-04·TLS         | harbor·argocd_bootstrap                                              | Harbor·Cluster                        | Harbor/Argo별 재개                                                          | G-06           |
| 70 Platform       | G-05·G-06        | Argo Root App · node_exporter_linux · alloy_linux                    | Cluster · 외부 CentOS Guest           | Cluster Application 단위 Sync 또는 실패 Guest의 Observability Role부터 재개 | G-05~07        |
| 80 Application    | Platform Healthy | Argo Application                                                     | Cluster                               | App/Route 단위 재개                                                         | G-08·RT-01~02  |
| 90 Validation     | G-01~08          | loadgen_support·validation                                           | loadgen·전체 경로                     | 실패 Gate 이후 중단; 해당 Gate부터                                          | G-09·F/L/DR/Q  |

<a id="section-7"></a>

## 7. 수동 공통 기반·CentOS Guest·VRouter·Firewall 자동화


<a id="section-7-1"></a>

### 7.1 Windows Host·VMware 수동 공통 기반과 Ansible 시작점

- Windows Host IP, C 드라이브 프로젝트 여유 500GB, VMware Workstation Pro, VMnet0 Bridged와 DHCP를 끈 Server별 Host-only VMnet은 공통 Runbook에 따라 수동 구성한다. 프로젝트 VM은 VMnet8 NAT를 사용하지 않는다.

- VM Hardware·vNIC, Thin Provisioning Disk, CentOS Stream 9 설치, 초기 계정과 SSH 준비도 수동 공통 기반이다. Ansible은 SSH와 become 검증을 통과한 CentOS Guest부터 관리하며 Windows Host와 VM 생성에는 실행하지 않는다.

- VRouter는 Bridged+Host-only vNIC 2장, 일반 내부 VM은 Host-only vNIC 1장, LB와 loadgen은 Bridged vNIC 1장인지 확인한다. VMnet 번호와 Guest Interface 이름은 추측하지 않고 실제 조회 후 host_vars 또는 Runbook에 동결한다.

- 역할별 대표 CentOS Guest 한 대에서 수동 1회 성공과 완료 Gate를 확보한 뒤 Role을 작성하고, 다른 대상에 적용한 후 동일 조건으로 재실행하여 멱등성을 확인한다.

- VMnet0 Bridged Guest의 교육장 Gateway·Internet 통신, Host-only DHCP 비활성화, Host Virtual Adapter의 Gateway·DNS 미설정, Windows Host·VMware 자원 상태와 임시 Snapshot 삭제·통합 여부를 수동 증거로 남긴다. Windows Host 재부팅과 VMware 변경은 Ansible Handler 대상이 아니다.


<a id="section-7-2"></a>

### 7.2 VRouter·Static Route

VRouter는 External vNIC와 Private vNIC 사이의 IP forwarding과 정적 경로를 소유한다. 일회성 ip route Command가 아니라 NetworkManager connection profile로 영구화한다. Masquerade는 Private Subnet에서 외부망으로 나가는 통신에만 적용하고 192.168.51.0/24~192.168.54.0/24 사이에는 NAT를 적용하지 않아 원본 Source IP를 유지한다. 각 Router를 순차 적용한 직후 외부·사설 양방향 Reachability, Return Path와 Source IP를 확인한다.

| **대상**   | **필수 경로**                                        |
|------------|------------------------------------------------------|
| vrouter-01 | 52/24 via 10.1.93.73 · 53/24 via .75 · 54/24 via .77 |
| vrouter-02 | 51/24 via 10.1.93.71 · 53/24 via .75 · 54/24 via .77 |
| vrouter-03 | 51/24 via .71 · 52/24 via .73 · 54/24 via .77        |
| vrouter-04 | 51/24 via .71 · 52/24 via .73 · 53/24 via .75        |
| lb-01/02   | 192.168.51~54.0/24를 각 VRouter 외부 IP로 전달       |


<a id="section-7-3"></a>

### 7.3 Firewall 관리 원칙

Component Role이 firewalld 규칙을 임의 추가하지 않고 host_firewall Role이 Source·Destination·Protocol·Purpose 데이터에서 최종 상태를 생성한다. Kubernetes Node의 iptables/nftables는 Calico·kube-proxy와 충돌하지 않도록 공식 요구사항에 맞춰 별도 정책으로 관리한다.

| **Source**             | **Destination**                    | **Port/Protocol**                 | **Purpose**                   | **적용 주체**                 | **Validation**    |
|------------------------|------------------------------------|-----------------------------------|-------------------------------|-------------------------------|-------------------|
| 관리자·Ansible         | CentOS Guest·loadgen               | 22/TCP                            | SSH·자동화                    | host_firewall                 | G-01              |
| Client/loadgen         | Common VIP                         | 80,443/TCP                        | HTTP·HTTPS·WSS                | host_firewall/LB              | G-03·G-08·L-01~02 |
| Admin·CP·Worker        | Common VIP                         | 6443/TCP                          | Kubernetes API                | host_firewall/LB              | G-03~04           |
| Backend·제한 Admin     | Common VIP                         | 3306/TCP                          | DB Stable Access              | host_firewall/NetworkPolicy   | G-03·G-05         |
| LB-01/02               | CP-01~03                           | 6443/TCP                          | API Backend                   | host_firewall/haproxy         | G-03·F-01~02      |
| LB-01/02               | Worker-01/02                       | 30080,30443/TCP                   | Gateway NodePort              | host_firewall/haproxy         | G-03·F-01·F-03    |
| LB-01/02               | MaxScale-01/02                     | 3306/TCP                          | DB Backend Pool               | host_firewall/haproxy         | G-03·F-05         |
| LB-01 ↔ LB-02          | VRRP Peer                          | 112/IP                            | Unicast VRRP                  | host_firewall/keepalived      | G-03·F-01         |
| CP-01~03               | CP-01~03                           | 2379-2380/TCP                     | stacked etcd                  | host_firewall                 | G-04·F-02         |
| CP·Metrics Server      | CP/Worker kubelet                  | 10250/TCP                         | Node·Resource Metrics         | host_firewall                 | G-04·G-07         |
| K8s Node               | K8s Node                           | 4789/UDP                          | Calico VXLAN                  | host_firewall                 | G-02·G-04         |
| MaxScale-01/02         | MariaDB-01/02                      | 3306/TCP                          | Query·Monitor                 | host_firewall/maxscale        | G-05·F-05~06      |
| MariaDB Replica        | MariaDB Primary                    | 3306/TCP                          | GTID Replication              | host_firewall/mariadb         | G-05·F-06         |
| Worker                 | NFS                                | 2049/TCP                          | NFSv4 PVC                     | host_firewall/nfs_server      | G-05·F-08~09      |
| Jenkins Agent          | Harbor·Git                         | 443/TCP                           | Push·Source/GitOps            | NetworkPolicy/egress          | G-06·F-10~11      |
| Worker                 | Harbor                             | 443/TCP                           | Image Pull                    | host_firewall/containerd      | G-06·F-10         |
| Prometheus             | kube-state/app                     | 8080·App metrics/TCP              | Object·App Scrape             | NetworkPolicy                 | G-07              |
| Prometheus             | CentOS Guest node_exporter·kubelet | 9100·10250/TCP                    | Guest/Node Metric Scrape      | NetworkPolicy/host_firewall   | G-07              |
| Prometheus             | Alertmanager                       | 9093/TCP                          | Alert 전달                    | NetworkPolicy                 | G-07·F-12         |
| Grafana                | Prometheus·Loki                    | 9090·3100/TCP                     | Metric·Log 조회               | NetworkPolicy                 | G-07              |
| Alloy K8s              | Loki                               | 3100/TCP                          | 내부 Log 전송                 | NetworkPolicy                 | G-07              |
| 외부 Alloy             | Worker Loki Endpoint               | 31000/TCP                         | 외부 Linux Log                | host_firewall                 | G-07·F-13         |
| Jenkins Controller     | Kubernetes API                     | 443/TCP                           | 동적 Agent 관리               | NetworkPolicy                 | G-06·F-11         |
| Alertmanager           | E-mail SMTP / 선택적 Discord 연계  | 587 또는 465/TCP; 선택 시 443/TCP | E-mail 필수·Discord 선택 알림 | NetworkPolicy/egress          | G-07·F-12         |
| 전체 CentOS Guest      | DNS·NTP                            | 53/TCP·UDP,123/UDP                | 이름·시간                     | host_firewall                 | G-01~02           |
| Ansible·CentOS Guest   | 승인 Repository                    | 443/TCP                           | Package·Image                 | host_firewall                 | G-01·G-06         |
| Backup Source          | loadgen                            | 22/TCP                            | SFTP/rsync+SSH                | host_firewall/backup_transfer | DR-01~04          |
| 관리·사설 CentOS Guest | 상호 대상                          | ICMP                              | Reachability·MTU·Source IP    | host_firewall                 | G-02              |

<a id="section-8"></a>

## 8. Common VIP·HAProxy·Keepalived·TLS 자동화

| **Common VIP Listener** | **Backend**       | **Health Check**         | **책임**         |
|-------------------------|-------------------|--------------------------|------------------|
| 10.1.93.90:80           | Worker :30080     | TCP + 최종 HTTP Redirect | Gateway 진입     |
| 10.1.93.90:443          | Worker :30443     | TCP + 최종 HTTPS/WSS     | 서비스 진입      |
| 10.1.93.90:6443         | CP :6443          | TCP 또는 API readyz      | kubeadm Endpoint |
| 10.1.93.90:3306         | MaxScale Listener | TCP + 최종 Transaction   | DB Stable Access |


<a id="section-8-1"></a>

### 8.1 Keepalived 기본값과 환경 Gate

| **항목**     | **설계 기본값**                       | **적용 전 확인**                    |
|--------------|---------------------------------------|-------------------------------------|
| Mode         | Unicast VRRP                          | LB 상호 IP 통신                     |
| VRID         | 51 시작값                             | 교육장 내 중복 VRID 없음            |
| Priority     | LB-01 110 / LB-02 100                 | Bootstrap 순서와 실제 Owner 확인    |
| Preemption   | nopreempt                             | 현재 Master 유지·자동 Failback 방지 |
| Interface    | 외부 Bridged vNIC 변수                | VMware에서 확인한 실제 Device Name  |
| Track        | HAProxy Process/Config와 LB 자체 상태 | Backend 하나의 장애와 분리          |
| Network      | VRRP Protocol 112 + Gratuitous ARP    | 추가 MAC·ARP 갱신 허용              |
| Version/Auth | VRRPv3·authentication block 미사용    | Unicast Peer·Firewall 최소 허용     |

VRRP·Gratuitous ARP가 교육장 L2에서 허용되지 않으면 원 목표 설계 자체를 잘못으로 성공 처리하지 않는다. 07 MVP에서 단일 LB 고정 IP로 축소하고, VIP Failover는 미구현 항목과 검증 제한으로 기록한다.


<a id="section-8-2"></a>

### 8.2 안전한 설정 적용

1. HAProxy/Keepalived Template을 임시 경로에 렌더링한다.

2. haproxy -c와 keepalived config-test 계열로 문법을 검증한다.

3. LB-01·LB-02는 모두 초기 state BACKUP으로 선언한다. Bootstrap 최초 적용은 LB-02를 먼저 준비한 뒤 Priority·기동 순서·실제 VRRP 상태로 LB-01의 VIP 소유를 확인한다. 이후에는 실제 VIP Owner를 식별하여 Standby부터 적용하고, nopreempt에 따라 현재 Master를 유지한다.

4. Standby Health와 Listener를 확인한 뒤 Active를 적용한다.

5. ARP Table·VIP Owner·네 포트 E2E를 검증하고 Evidence를 저장한다.


<a id="section-8-3"></a>

### 8.3 TLS

Common VIP의 :80/:443은 HAProxy L4 TCP 전달이며 TLS termination은 NGINX Gateway Fabric이 담당한다. Internal Root CA는 Gateway와 Harbor 인증서를 서명한다. SAN에는 service.stone.test와 harbor.stone.test를 포함하고, k8s-api.stone.test는 kubeadm API Certificate SAN과 일치시킨다. Private Key는 Vault에 보관하며 정상 재실행에서 CA를 재생성하지 않는다.

<a id="section-9"></a>

## 9. Kubernetes·Calico·Gateway·Cluster Add-on 자동화


<a id="section-9-1"></a>

### 9.1 kubeadm 실행 상태

| **감지 상태**              | **정상 Playbook 행동**                                   |
|----------------------------|----------------------------------------------------------|
| 미구축                     | Stable API Endpoint 확인 후 First CP init 또는 join 수행 |
| 정상 구성                  | init/join 제외, Version·Node·etcd·Certificate Validation |
| 파일만 존재·Cluster 비정상 | 자동 reset 금지, 실패 후 Recovery 분기                   |
| Node/VM 유실               | 새 OS Baseline→Prereq→살아 있는 Cluster에 join           |
| 전체 etcd 유실             | 일반 구축이 아닌 Snapshot Restore 절차                   |

1. Common VIP :6443과 HAProxy Backend가 준비된 상태에서 시작한다.

2. containerd·Kernel·swap·sysctl·Package Version을 일치시킨다.

3. cp-01에서 kubeadm Configuration File로 init한다.

4. cp-02, cp-03은 stacked etcd 변화를 고려해 serial: 1로 join한다.

5. worker-01, worker-02를 join하고 Pre-CNI 상태를 확인한다.

6. Calico 적용 후 Node Ready·CoreDNS·Cross-node Pod 통신을 확인한다.


<a id="section-9-2"></a>

### 9.2 Calico 결정

| **항목**      | **결정**                      | **이유·검증**                                         |
|---------------|-------------------------------|-------------------------------------------------------|
| Pod CIDR      | 10.244.0.0/16                 | 외부·사설·Service CIDR과 비중복 검사                  |
| Service CIDR  | 10.96.0.0/12                  | kubeadm 단일 기준                                     |
| Install       | Tigera Operator               | 공식 Lifecycle 자산 재사용                            |
| Encapsulation | VXLAN CrossSubnet             | 서로 다른 Routed Subnet 간 Overlay                    |
| natOutgoing   | true                          | Pod→외부 DB/Registry 경로 단순화                      |
| MTU           | min(Path MTU)-50              | 실제 Routed Underlay 측정 후 동결; 1500이면 1450 예상 |
| BGP/IPIP      | 기본 제외                     | Static Routing + VXLAN에서 불필요한 복잡도            |
| NetworkPolicy | 핵심 허용→Default Deny 단계화 | 임시 Deny/Allow Test로 Enforcement 확인               |


<a id="section-9-3"></a>

### 9.3 Gateway·Add-on

Argo CD가 Gateway API CRD, NGINX Gateway Fabric, GatewayClass/Gateway, HTTPRoute와 Application NetworkPolicy를 관리한다. Data Plane Service는 NodePort 30080/30443, externalTrafficPolicy: Cluster로 시작한다. Stage 20에서는 CentOS Guest VM·Kubernetes Node·loadgen의 /etc/hosts만 관리하고 Windows Host 이름 해석은 수동 공통 기반으로 구분한다. Stage 40에서 CoreDNS Ready를 확인한 후 Ansible의 coredns_records가 기본 kube-system/coredns ConfigMap의 Corefile에 제한적 hosts block을 멱등 관리하여 service·k8s-api·db를 Common VIP 10.1.93.90에, harbor를 192.168.53.61에 매핑한다. hosts block에는 fallthrough를 두고 ConfigMap Backup·Diff를 먼저 확인하며, 변경된 경우에만 CoreDNS를 Rollout한 뒤 Pod 내부 DNS Query Gate를 통과시킨다. 내부 DNS가 준비되면 이름과 인증서 SAN은 유지하고 관리하던 hosts block만 제거한다.

| **Add-on**         | **책임**                                  | **Validation**                  |
|--------------------|-------------------------------------------|---------------------------------|
| Metrics Server     | HPA·kubectl top Resource Metrics          | metrics.k8s.io, Node/Pod Metric |
| CoreDNS            | Cluster DNS + 제한적 stone.test 정적 항목 | Pod 내부 이름 해석              |
| kube-state-metrics | Kubernetes Object 상태 Metric             | Prometheus Target·Query         |
| NFS Provisioner    | 기존 NFS Share의 동적 PV/PVC              | PVC 생성·쓰기·삭제·재생성       |
| Argo CD            | Root Application과 Sync                   | Synced/Healthy·Drift Self-Heal  |

<a id="section-10"></a>

## 10. MariaDB·MaxScale·Redis·ANALYSIS·NFS 자동화


<a id="section-10-1"></a>

### 10.1 MariaDB·MaxScale

MariaDB는 GTID 기반 1 Primary + 1 Replica 비동기 복제를 기본으로 한다. Replica 한 대인 구조에서 Semi-sync를 기본값으로 추가하지 않고 실제 Replication Lag과 RPO를 측정한다. mariadb Role 하나가 Inventory Group에 따라 Primary/Replica 설정을 적용하며, 정상 재실행에서 Data Directory 삭제·자동 Seed·승격을 수행하지 않는다.

| **단계**         | **검증/보호**                                                          |
|------------------|------------------------------------------------------------------------|
| 초기 Seed        | mariadb-backup 형식으로 Replica Seed; Backup/Restore 체계와 재사용     |
| Replication      | GTID, IO/SQL Thread, Lag, Read-only, 계정 최소 권한                    |
| MaxScale Monitor | 전용 Monitor 계정·Topology 확인·두 MaxScale의 단일 권한 원칙           |
| DB Access        | Common VIP :3306→HAProxy→생존 MaxScale→Primary                         |
| Primary 장애     | 구 Primary Fencing→Replica/GTID 확인→승인→MaxScale failover→Write 검증 |
| 재편입           | Old Primary를 자동 복귀시키지 않고 재동기화 후 Replica로 편입          |

> **과장 금지** MaxScale·Replication은 Backup을 대체하지 않으며, 2-node MariaDB에서 완전 자동 승격을 기본값으로 사용하지 않는다. 서비스 복구시간과 실제 ACK 데이터 유실량을 따로 측정한다.


<a id="section-10-2"></a>

### 10.2 Redis·ANALYSIS

Redis는 단일 Stateful Workload, NFS PVC와 AOF appendfsync everysec을 사용한다. Sentinel/Cluster는 기본 범위가 아니다. Redis Pub/Sub Event는 복구 기준이 아니며, 장애 후 MariaDB의 공식 Move/Result와 비교해 상태를 일치시킨다. AOF/PVC가 복구 불가능하면 MariaDB의 영속 Move·GameResult에서 Board·종료 Game 등 재구성 가능한 파생 상태만 복원한다. GuestSession·Participant·Ready·Current Vote 등 영속 DATA에 없는 활성 상태는 전체 자동 복원 가능하다고 간주하지 않는다.

| **항목**      | **설계**                           | **성공 기준**                              |
|---------------|------------------------------------|--------------------------------------------|
| Runtime State | Session·Room/Game/Turn/Vote        | 재기동 후 허용 범위 내 복구·권위 DB와 일치 |
| Streams Job   | 공식 Move 이후 game_id+move_no     | Consumer Group에서 단일 논리 처리          |
| ACK/Pending   | 처리 완료 후 ACK, Pending 재처리   | 작업 유실 없이 중복 결과 격리              |
| Retry/DLQ     | 제한 Retry 후 실패 Stream 격리     | 게임 권위 경로 무중단                      |
| Stale Result  | 현재 move_no와 다른 결과 폐기/보관 | 최신 판세를 덮어쓰지 않음                  |
| ANALYSIS 장애 | 지연·중단·오류 허용                | Vote·Move·Turn·Result·Rating 진행 유지     |


<a id="section-10-3"></a>

### 10.3 NFS·Local Storage

| **소비자**             | **저장 방식**                                    | **장애·복구 경계**                          |
|------------------------|--------------------------------------------------|---------------------------------------------|
| Redis                  | NFS PVC + AOF                                    | NFS I/O 중단과 Redis Pod 재기동을 분리 검증 |
| Jenkins Controller     | NFS PVC                                          | Controller 재배치·PVC 재연결 전 CI 중단     |
| MariaDB Backup Staging | NFS Share                                        | Restore 실험용 임시 사본; 별도 DR 아님      |
| Prometheus             | 정적 Local PV, 7일 또는 20GiB                    | Node 종속·일부 Metric 유실 수용             |
| Loki                   | Single Binary + 정적 Local PV, 72시간 또는 10GiB | Node 종속·일부 Log 유실 수용                |
| Protected Evidence     | loadgen 별도 Disk                                | Checksum 검증; off-site·Site DR 아님        |

NFS Subdir External Provisioner 장애는 신규 PV/PVC 생성 제어 경로 장애이고, NFS Server/Share 장애는 기존 Volume I/O 장애다. 두 시나리오를 별도 Test ID로 관리한다.

<a id="section-11"></a>

## 11. Jenkins·Harbor·Argo CD CI/CD·GitOps

```text
Source Commit → Jenkins Test → Rootless BuildKit Image Build
→ Harbor Push (immutable tag + digest) → GitOps Branch/PR
→ Review/Merge → Argo CD Sync → Deployment Healthy
```

<a id="section-11-1"></a>

### 11.1 Jenkins in-cluster

- Controller는 Kubernetes 내부에서 실행하고 JCasC로 설정한다. Job·Plugin·설정 상태는 NFS PVC에 보존한다.

- Kubernetes Plugin으로 동적 Agent Pod를 생성하며 Build Workspace는 emptyDir를 기본으로 한다.

- Host Docker Socket Mount는 Worker Host 권한을 과도하게 노출하므로 제외한다.

- Rootless BuildKit을 기본 Image Builder로 사용하고 SecurityContext·Cache·Harbor TLS Trust를 검증한다.

- Worker 한 대 장애 시 신규 Build Agent를 제한하고 사용자 Runtime Resource를 우선한다.

- Plugin은 Core 호환 버전 집합을 Plugin Catalog/Lock 파일로 고정한다.


<a id="section-11-2"></a>

### 11.2 Harbor·GitOps 추적성

| **대상**       | **정책**                                           | **검증**                        |
|----------------|----------------------------------------------------|---------------------------------|
| Image Tag      | git-\<commit-sha\>, latest 금지                    | Source Commit과 Digest 연결     |
| Harbor Project | Private + Tag Immutability                         | 동일 Tag 덮어쓰기 거부          |
| Credential     | 최소 권한 Robot 계정                               | Admin Credential 미사용         |
| GitOps 변경    | 별도 Branch·PR·Review/Merge                        | 직접 main Push 기본 제외        |
| Argo CD        | Auto Sync + Self-Heal; Prune는 삭제 검증 후 활성화 | Synced/Healthy와 Drift 복구     |
| Rollback       | Git Revert→Merge→Argo CD Sync                      | 이전 Digest Healthy와 시간 측정 |

> **책임 경계** Jenkins는 kubectl apply, helm upgrade, argocd app sync로 최종 Cluster 배포를 수행하지 않는다. Jenkins의 마지막 책임은 검증된 Image와 GitOps 변경 제안이며, 실제 적용은 Argo CD가 소유한다.

<a id="section-12"></a>

## 12. Prometheus·Loki·Grafana·Alloy·Alertmanager

| **구성요소**   | **배치·저장**                           | **책임**                                                        |
|----------------|-----------------------------------------|-----------------------------------------------------------------|
| Prometheus     | in-cluster + Worker Local               | Metric 단기 저장·Alert Rule 평가                                |
| Alertmanager   | in-cluster, 기본 PVC 없음               | Grouping·Dedup·Inhibition·Silence·Routing                       |
| Grafana        | in-cluster, Dashboard/Datasource GitOps | Metric·Log 조회; UI 변경은 공식 상태 아님                       |
| Loki           | Single Binary + Worker Local            | Log 단기 저장·조회                                              |
| Alloy K8s      | DaemonSet                               | Pod·Node·Kubernetes Event Log 수집                              |
| Alloy Linux    | CentOS Guest·외부 지원 Linux VM systemd | Journal·File Log를 관리 대역 Endpoint로 전송; Windows Host 제외 |
| Metrics Server | in-cluster                              | HPA용 Resource Metrics API; Prometheus와 분리                   |

Application Metric은 Application/GitOps 영역이 노출 책임을 가지며, MariaDB·MaxScale 등 외부 Data 구성요소의 Metric은 해당 Data 구성 또는 별도 Exporter를 통해 Prometheus가 수집한다. 정확한 Exporter 방식과 버전은 구현 시 공식 지원·호환성·운영 복잡도를 검증한 뒤 동결하며, 06 단계에서 불필요하게 특정 Exporter 제품을 선확정하지 않는다.


<a id="section-12-1"></a>

### 12.1 Local PV와 Retention

Prometheus와 Loki는 Static Local PV, nodeAffinity와 명시적 Host Path를 사용한다. Node 종속성을 Scheduler가 인식하도록 단순 hostPath보다 Local PV를 선택한다. Node 유실 시 과거 관측 데이터 일부 유실과 재배치 지연을 수용하되, 해당 실험의 핵심 Metric·Log·Timestamp는 장애 주입 전후 즉시 외부 Evidence Disk로 Export한다.


<a id="section-12-2"></a>

### 12.2 Alertmanager

| **정책**   | **시작값**                                                    | **검증**                                               |
|------------|---------------------------------------------------------------|--------------------------------------------------------|
| Severity   | info=기록만 / warning=반복·가용성 저하 / critical=운영자 개입 | 일시 자동복구는 통보 없음; Warning/Critical Route 검증 |
| Grouping   | alertname·namespace·service                                   | 동일 장애 알림 다량 발생 제한                          |
| Inhibition | critical이 warning 제한                                       | 중복 통보 감소                                         |
| Receiver   | E-mail 기본                                                   | firing/resolved 수신·시간 측정                         |
| Discord    | 호환 중계 확인 시 선택                                        | 직접 Generic Webhook으로 단정하지 않음                 |
| 장애 경계  | 운영 기능 열화                                                | 게임 서비스 유지·능동 통보 공백 기록                   |

<a id="section-13"></a>

## 13. 멱등성·부분 재실행·Drift·Recovery


<a id="section-13-1"></a>

### 13.1 재실행 계약

| **상태**         | **정상 Playbook 행동**             | **금지**                    |
|------------------|------------------------------------|-----------------------------|
| 미구축           | 필요 구성 생성 후 Validation       | 숨은 임시값                 |
| 정상             | ok 중심, 필요한 검증만 수행        | 무조건 Restart              |
| Config Drift     | Template/Variable 기준으로 일치    | 수동 상태를 정답으로 채택   |
| 부분 실패        | Component Playbook + --limit + Tag | Foundation 전체 재실행 강제 |
| 상태 불명확      | Fail closed 후 Recovery 안내       | 자동 reset·wipe             |
| 데이터 상태 전이 | 승인된 Recovery Playbook           | 정상 site.yml에서 실행      |


<a id="section-13-2"></a>

### 13.2 멱등성 판정

1. Clean 상태에서 최초 실행의 changed/ok/failed와 총 시간을 저장한다.

2. 동일 Inventory·Commit·Version으로 즉시 2회차 실행한다.

3. 2회차 실행의 허용 changed 목록을 사전 정의하고 그 외 변경을 결함으로 판정한다.

4. Template 하나를 변경해 해당 Handler만 동작하고 무관한 Service가 재시작되지 않는지 확인한다.

5. 중간 Task에서 의도적으로 실패시킨 뒤 Component/Host 범위로 재실행한다.

6. 수동 Drift를 주입하고 Check Mode 가능 영역의 차이와 실제 재복원 결과를 저장한다.


<a id="section-13-3"></a>

### 13.3 위험 작업 Guard

| **작업**             | **필수 Guard**                                                                              |
|----------------------|---------------------------------------------------------------------------------------------|
| kubeadm reset        | recovery playbook + exact host + confirm_reset=true                                         |
| MariaDB Promotion    | Old Primary Fencing 증거 + GTID 확인 + confirm_promotion=true                               |
| etcd Restore         | Snapshot Checksum + target cluster + quorum 중단 절차 + 승인                                |
| NFS Destructive Test | 전용 Test Export/PVC + Production-like 데이터 격리                                          |
| CA Rotation          | 신 Trust 배포 설계안 + Rollback + confirm_ca_rotation=true                                  |
| VM Delete/Recreate   | Windows·VMware 수동 Recovery + 정확한 VMX/VMDK·Snapshot/Backup 확인; Ansible 자동 삭제 금지 |

<a id="section-14"></a>

## 14. 설치 후 Validation Gate

모든 테스트는 전제 조건 → 실행 절차 → 관측 지점 → 성공 기준 → 실패 판정 → 복구 → 증거의 동일 형식을 사용한다. 선행 Gate가 실패하면 후속 영역의 후속 오류를 만들지 않도록 중단한다.

```text
Foundation Gate → Network Gate → Endpoint Gate
→ Kubernetes Pre-CNI Gate → Calico/Post-CNI Gate
→ Data/Storage Gate + Delivery Gate → GitOps Platform Gate
→ Application Gate → E2E Gate → Experiment Ready
```

| **Gate**           | **관측**                                                      | **통과 기준**                             |
|--------------------|---------------------------------------------------------------|-------------------------------------------|
| G-01 Foundation    | 수동 기반 Checklist, SSH/become, Time, Repository, Guest Disk | 수동 증거와 모든 Critical Guest 항목 PASS |
| G-02 Network       | IP·Route·Return Path·DNS·NTP·Port                             | 중복·비대칭·MTU 오류 없음                 |
| G-03 Endpoint      | VIP Owner와 :80/:443/:6443/:3306                              | 포트별 실제 경로 PASS                     |
| G-04 Kubernetes    | etcd 3-member, Node, CoreDNS, Pod Network                     | Quorum·Ready·Cross-node 통신              |
| G-05 Data/Storage  | Replication, MaxScale, NFS, PVC, AOF                          | Read/Write·복제·PVC I/O                   |
| G-06 Delivery      | Harbor, Jenkins Agent, GitOps Sync                            | Commit→Digest→Healthy 연결                |
| G-07 Observability | Scrape, Log, Alert firing/resolved                            | Metric·Log 조회와 E-mail                  |
| G-08 Application   | HTTPS/WSS, Vote·Move·Persist·Reconnect                        | 권위 규칙·상태 일치                       |
| G-09 Experiment    | k6, Backup, Failure Guard, Evidence                           | 재현 가능한 실험 준비                     |


<a id="section-14-1"></a>

### 14.1 E2E 경로

- DB는 개별 MariaDB IP가 아니라 Common VIP :3306→HAProxy→MaxScale→Primary 실제 Transaction으로 검증한다.

- 서비스는 Worker NodePort 직접 접근이 아니라 service.stone.test→Common VIP→Gateway→Backend 경로를 사용한다.

- HTTP /health 200에 그치지 않고 Runtime State Write/Read와 MariaDB Persistent Write를 포함한다.

- WebSocket은 연결·메시지 송수신·유지·Pod 교체 후 Reconnect와 Snapshot 일치를 분리한다.

- 투표 종료 경합에서 공식 Move가 한 번만 확정되고 중복 요청이 동일 결과로 일치하는지 확인한다.

<a id="section-15"></a>

## 15. 장애·Failure Domain·복구 검증

```text
Precheck → Baseline Traffic/State → Failure Injection(T1)
→ Detection(T2) → Native Failover/Recovery(T3)
→ Service Restored(T4) → Full Recovery(T5)
→ Data/State Validation → Evidence
```

| **ID** | **장애**                      | **핵심 검증**                                       |
|--------|-------------------------------|-----------------------------------------------------|
| F-01   | Active LB 중단                | VIP Owner 이동·4 Listener 복구·실패 요청·Reconnect  |
| F-02   | Control Plane 1대 중단        | API 지속·etcd 2/3 quorum·Node 재join                |
| F-03   | Worker 1대 중단               | 필수 Pod 재배치·Build 제한·서비스 복구·Rebuild MTTR |
| F-04   | Gateway/Backend Pod 삭제      | Kubernetes Self-Healing·WSS 재연결                  |
| F-05   | MaxScale 1대 중단             | HAProxy Backend 제외·신규 DB Connection 복구        |
| F-06   | MariaDB Primary 중단          | Fencing·승격 승인·Write 복구·ACK 기반 RPO           |
| F-07   | Redis Pod 중단                | AOF/PVC 상태·Streams Pending·공식 DB와 일치         |
| F-08   | NFS Provisioner 중단          | 기존 PVC I/O 유지·신규 PVC 실패·복구                |
| F-09   | NFS Server 중단               | 기존 I/O 영향·Redis/Jenkins 열화·MTTR               |
| F-10   | Harbor 중단                   | 실행 Pod 유지 vs 신규 Pull/Scale/Deploy 실패 분리   |
| F-11   | Jenkins Controller/Agent 중단 | Runtime 무영향·CI 재개·PVC 재연결                   |
| F-12   | Alertmanager 중단             | 게임 유지·통보 공백·복구 후 resolved                |
| F-13   | Prometheus/Loki Node 유실     | 관측 공백·Local 데이터 유실·Evidence 보존           |
| FD-01  | Server-01 중단                | CP+Worker+Replica 복합 영향                         |
| FD-02  | Server-02 중단                | CP+Worker+Primary, DB 승격과 Runtime 동시 검증      |
| FD-03  | Server-03 중단                | CP+LB+MaxScale+Harbor 복합 영향                     |
| FD-04  | Server-04 중단                | LB+MaxScale+NFS+Controller, 관리 Plane 복구 선행    |


<a id="section-15-1"></a>

### 15.1 핵심 장애 Test Case


<a id="test-f-01"></a>

#### F-01 Active LB 중단

| **필드**  | **실행 설계**                                                                               |
|-----------|---------------------------------------------------------------------------------------------|
| 전제 조건 | G-03 PASS, LB-01이 현재 VIP Owner, 네 Listener Baseline과 지속 HTTPS/WSS·DB·API Probe 실행  |
| 실행 절차 | T1 기록 후 LB-01의 HAProxy/Keepalived를 정상 종료하고 VIP·ARP·연결 변화를 1초 간격으로 수집 |
| 관측 지점 | LB 양쪽 ip addr/VRRP log, Client ARP, :80/:443/:6443/:3306 Probe, WSS reconnect             |
| 성공 기준 | VIP가 LB-02로 이동하고 네 Listener가 복구되며 중복 VIP가 없고 실패 요청·전환시간이 산출됨   |
| 실패 판정 | VIP 미이동·동시 소유·일부 Listener만 복구·ARP 미갱신 또는 Stop Condition 초과               |
| 복구      | LB-01 재기동 후 nopreempt로 LB-02 Master 유지, 상태 확인 뒤 계획된 전환에서만 Owner 변경    |
| 증거      | T1~T5, VRRP/HAProxy log, ip/ARP, 포트별 Probe CSV, WSS reconnect, SHA256SUMS                |


<a id="test-f-03"></a>

#### F-03 Worker 1대 중단

| **필드**  | **실행 설계**                                                                                          |
|-----------|--------------------------------------------------------------------------------------------------------|
| 전제 조건 | G-04~08 PASS, 핵심 Workload 2 Worker 분산, Priority/Request와 Jenkins Build 제한 정책 확인             |
| 실행 절차 | Baseline 부하 중 Worker 전원 종료 또는 VM 중단, Node NotReady부터 재스케줄·신규 연결·Build 억제를 관측 |
| 관측 지점 | Node/Pod/Event, Endpoint, HPA, HTTP/WSS, PVC/Image Pull, Jenkins Agent 생성, 남은 Worker 자원          |
| 성공 기준 | 필수 Runtime이 남은 Worker에 수용되고 신규 서비스가 복구되며 비필수 Build가 Runtime을 침해하지 않음    |
| 실패 판정 | 필수 Pod Pending 지속·자원 고갈·PVC/Image Pull 실패·오류율 Stop Condition 초과                         |
| 복구      | Worker OS/Runtime/Prereq 재수렴 후 기존 Cluster Join; Scheduling·Endpoint 정상화 후 Cordon 해제        |
| 증거      | kubectl get/describe/events, Resource CSV, HTTP/WSS 결과, Jenkins Queue, Ansible Rebuild recap         |


<a id="test-f-06"></a>

#### F-06 MariaDB Primary 중단

| **필드**  | **실행 설계**                                                                                                             |
|-----------|---------------------------------------------------------------------------------------------------------------------------|
| 전제 조건 | G-05 PASS, GTID·Lag 정상, Backup/Checksum 확보, ACK 식별 Transaction과 Fencing 수단 준비                                  |
| 실행 절차 | T1 후 구 Primary 중단·격리, Replica GTID/Read-only 확인, 승인 Flag로 승격, MaxScale topology 갱신 후 VIP Transaction 수행 |
| 관측 지점 | MariaDB/MaxScale log, GTID·Lag, VIP :3306, ACK Transaction ID, Application Error, Fencing Evidence                        |
| 성공 기준 | 구 Primary가 쓰기 경로에서 격리되고 승인 후 Write가 재개되며 ACK 기준 RPO와 복구시간이 계산됨                             |
| 실패 판정 | 동시 Primary·승인 없는 승격·ACK Record 유실·GTID 불일치·MaxScale 오라우팅                                                 |
| 복구      | 구 Primary 자동 복귀 금지; 새 Primary에서 재동기화해 Replica로 편입하고 topology/Transaction 재검증                       |
| 증거      | 승인·Fencing log, GTID snapshot, Transaction CSV, MaxScale servers, T1~T5, Checksum                                       |


<a id="test-f-07"></a>

#### F-07 Redis Pod 중단

| **필드**  | **실행 설계**                                                                                                                                                                                                                                                                            |
|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 전제 조건 | G-05·08 PASS, AOF everysec와 NFS PVC 정상, Streams Consumer Group/Pending Baseline 저장                                                                                                                                                                                                  |
| 실행 절차 | 공식 Move와 분석 Job을 생성한 뒤 Redis Pod 삭제, 재기동 후 AOF/PVC·Pending 재처리·Snapshot 복원을 확인                                                                                                                                                                                   |
| 관측 지점 | Redis INFO/AOF log, PVC mount, Stream length/Pending/ACK, Backend Snapshot, MariaDB 공식 Move/Result                                                                                                                                                                                     |
| 성공 기준 | AOF/PVC로 복구한 범위가 권위 DB와 일치하고 처리 완료 Job이 중복 반영되지 않으며 stale 분석이 최신 표시를 덮지 않음                                                                                                                                                                       |
| 실패 판정 | AOF 손상·권위 경로 중단·작업 유실·중복 결과 반영·stale 오반영·비영속 활성 상태의 근거 없는 자동 복원                                                                                                                                                                                     |
| 복구      | PVC/AOF 검증 후 Pod 재생성; 복구 불가능 시 MariaDB Move·GameResult에서 Board·종료 Game 등 파생 상태만 재구성하고 Pending을 격리 재처리. GuestSession·Participant·Ready·Current Vote 등을 권위 있게 복구할 수 없는 활성 Room/Game은 SYSTEM_INVALID로 처리하고 전적·Rating에 반영하지 않음 |
| 증거      | AOF/PVC 상태, Stream Pending 전후, DB 비교 JSON, Pod Event/Log, RT-01 결과                                                                                                                                                                                                               |


<a id="test-f-09"></a>

#### F-09 NFS Server 중단

| **필드**  | **실행 설계**                                                                                          |
|-----------|--------------------------------------------------------------------------------------------------------|
| 전제 조건 | 전용 Test PVC와 Redis/Jenkins PVC 정상, DR-03~04 Backup·Checksum 확보, 파괴 시험 승인                  |
| 실행 절차 | NFS Export 또는 VM을 중단하고 기존 PVC I/O·Redis·Jenkins 영향을 관측한 뒤 NFS를 복구                   |
| 관측 지점 | NFS service/export, mount/I/O latency, Pod Event, Redis AOF, Jenkins Controller, 신규/기존 PVC         |
| 성공 기준 | 영향 범위를 Provisioner 장애와 구분하고 복구 후 기존 PVC I/O·Redis/Jenkins가 무결성 검사와 함께 정상화 |
| 실패 판정 | 무승인 데이터 손상·Mount 고착·복구 후 Checksum 불일치·영향 경계 미분리                                 |
| 복구      | NFS VM/Export→Network→Mount→PVC 소비자 순으로 복구; 손상 시 DR-03/04 Restore 수행                      |
| 증거      | NFS/journal, mount·I/O CSV, PVC/Pod Event, AOF/Workspace Checksum, T1~T5                               |


<a id="test-fd-02"></a>

#### FD-02 Server-02 복합 장애

| **필드**  | **실행 설계**                                                                                                   |
|-----------|-----------------------------------------------------------------------------------------------------------------|
| 전제 조건 | CP 3/Worker 2/DB 복제 정상, F-03·F-06 단독 절차 검증, 장애 전 Backup과 Stop Condition 승인                      |
| 실행 절차 | Server-02 전원 중단으로 CP-02·Worker-02·MariaDB Primary 동시 상실을 주입하고 quorum·Runtime·DB 복구를 순차 판정 |
| 관측 지점 | etcd quorum/API, Node/Pod, VIP HTTP/WSS/DB, MariaDB GTID, 사용자 결과·Rating, 남은 Worker 자원                  |
| 성공 기준 | API quorum 유지, Runtime 조건부 수용, 승인된 DB 승격 후 Write 복구, 시스템 장애 오패배 0건                      |
| 실패 판정 | quorum 상실·Runtime 자원 고갈·split-brain·오패배/Rating 반영·Stop Condition 초과                                |
| 복구      | DB Fencing/승격을 먼저 닫고 Server-02→CP/Worker→구 DB Replica 순으로 재편입                                     |
| 증거      | FD Timeline, etcd/Node/Pod/GTID, RT-02 결과, 자원·오류 CSV, Recovery Playbook recap                             |


<a id="section-15-2"></a>

### 15.2 RTO·RPO 판정

| **지표**              | **정의**                                                    |
|-----------------------|-------------------------------------------------------------|
| Detection Time        | T1 장애 발생부터 T2 감지까지                                |
| Failover Time         | T1부터 T3 전환 완료까지                                     |
| Service Recovery Time | T1부터 사용자 경로가 다시 성공하는 T4까지                   |
| MTTR                  | T1부터 장애 전과 동등한 정상 상태 T5까지                    |
| RPO                   | 장애 직전 ACK 성공 데이터와 복구 후 존재 데이터의 실제 차이 |

> **MariaDB 데이터 판정** ACK 성공 + Record 없음은 중요 Data Loss다. ACK 실패 + Record 없음은 정상 실패 가능성으로 분리한다. 예상값을 결과처럼 미리 쓰지 않고 최초 Baseline 후 목표 Threshold를 동결한다.


<a id="section-15-3"></a>

### 15.3 Failure Domain·Recovery·Test 단일 기준표

| **Failure Domain** | **영향**                   | **Native HA**                          | **Ansible Recovery**                                          | **Test ID**             | **Evidence**        |
|--------------------|----------------------------|----------------------------------------|---------------------------------------------------------------|-------------------------|---------------------|
| FD-01 Server-01    | CP·Worker·Replica          | etcd quorum·Replica 비권위             | Windows·VMware 수동 복구→CP/Worker/Replica Role 재적용        | FD-01·F-02~03           | API·Pod·GTID·MTTR   |
| FD-02 Server-02    | CP·Worker·Primary          | etcd quorum·Worker 조건부              | Fencing→승격→Windows·VMware 수동 복구→Node/DB 재편입          | FD-02·F-03·F-06·RT-02   | RPO·오패배·Timeline |
| FD-03 Server-03    | CP·LB·MaxScale·Harbor      | VIP/MaxScale Pair·etcd quorum          | Peer 경로 확인→Windows·VMware 수동 복구→Guest 서비스 재구축   | FD-03·F-01~02·F-05·F-10 | 포트·Pull·Sync·MTTR |
| FD-04 Server-04    | LB·MaxScale·NFS·Controller | VIP/MaxScale Pair; NFS/Controller SPOF | 긴급 관리 경로→Windows·VMware 수동 복구→NFS→Controller→나머지 | FD-04·F-01·F-05·F-08~09 | I/O·통보·복구 recap |

<a id="section-16"></a>

## 16. 부하·HPA·실시간·ANALYSIS 검증

부하는 서비스 Failure Domain 밖의 5번째 Windows Host 10.1.93.92에서 실행하는 CentOS Stream 9 Bridged VM loadgen 10.1.93.91에서 k6로 발생시킨다. Load Generator CPU·Memory·Network·dropped_iterations를 함께 수집하여 부하 발생기 자체가 성능 제한인 Run은 무효 또는 재시험으로 처리한다.

| **Profile**   | **Scenario**          | **시작 Stage**                                                     | **관측**                        | **판정·Stop**                                       |
|---------------|-----------------------|--------------------------------------------------------------------|---------------------------------|-----------------------------------------------------|
| L-01 HTTP     | 로그인·방 목록·입장   | Warm-up 2m@10rps → 5m씩 25/50/100rps → 2m Recovery                 | RPS·p95/p99·Error               | loadgen CPU\<85%; 성능 Threshold는 Baseline 후 동결 |
| L-02 WSS      | 연결·Broadcast·재연결 | 5m씩 50/100/200 연결, 5초당 1 message, 마지막 2m에 10% 강제 재연결 | Connect·Message 지연·Disconnect | Snapshot 불일치 0; 발생기 병목 Run 무효             |
| L-03 Vote     | 투표·변경·마감 경합   | 3m씩 20/50/100 events/s, 마감 ±1초 Burst, 10 Room 시작값           | events/s·단일 Move·충돌         | RT-01 정확성 기준 전부 충족                         |
| L-04 HPA      | Static vs CPU HPA     | L-01 동일 25→50→100rps, 각 5m; 두 구조 동일 순서                   | Replica·Ready·CPU·p95/p99       | 동일 Profile·자원·완료 Gate로 비교                  |
| L-05 ANALYSIS | Streams Job 누적      | 3m씩 5/10/20 jobs/s, 처리기 지연·중단 각 2m                        | Queue Lag·Pending·stale         | 게임 권위 경로 지연·오류를 함께 기록                |
| L-06 Resource | Build·Runtime 경합    | Runtime 50rps+100 WSS 유지, Build 1개 후 선택적으로 2개 동시       | Eviction·Throttle·SLA           | Runtime 우선; Build 제한 작동                       |
| L-07 Storage  | NFS/Backup I/O        | 전용 Test PVC 4KiB randrw 60s×3, Backup 1개 동시 비교              | Latency·IOPS·AOF/Jenkins        | 운영형 PVC 직접 파괴 금지                           |
| L-08 Failure  | 정상 부하 중 장애     | 50rps+100 WSS 10m, 3분에 주입, 8분까지 회복 관측                   | 실패 요청·Reconnect·T1~T5       | 선택 F/FD Test의 Stop Condition 적용                |


<a id="section-16-1"></a>

### 16.1 HPA 결정 절차

1. Backend Static Replica와 resources.requests.cpu를 먼저 정의한다.

2. Smoke→Step Load→Capacity Curve로 CPU·Latency·Error의 성능 변화 지점을 측정한다.

3. 성능 저하 전 구간을 근거로 targetUtilization과 min/maxReplicas를 동결한다.

4. 같은 Profile로 Static과 HPA를 재비교한다.

5. Trigger→새 Pod Ready→Traffic 참여→CPU 재분산→p95/p99/Error 변화까지 측정한다.

6. Scale-out 효과가 없으면 DB·Redis·Network 성능 제한 가능성을 결과로 기록한다.


<a id="section-16-2"></a>

### 16.2 서비스 권위·동시성 Test Case


<a id="test-rt-01"></a>

#### RT-01 Turn·Move·Result·Analysis 멱등성

| **필드**  | **실행 설계**                                                                                                                                                                                                                          |
|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 전제 조건 | G-08 PASS, 서버 권위 deadline·game_id+turn_no·Idempotency Key 적용, MariaDB 검증 Query 준비                                                                                                                                            |
| 실행 절차 | 정상 Turn, 0표 Pass, 동일 마감 중복 호출, 중복 Result/Rating 요청, 과거 turn_no Vote와 과거 move_no Analysis 결과를 L-03 Burst로 주입                                                                                                  |
| 관측 지점 | Turn/Move/GameResult/RatingHistory/BoardAnalysis, Streams Job·Event/Idempotency log, Client 응답·표시, Redis Runtime State                                                                                                             |
| 성공 기준 | 정상 Turn Move=1·Analysis 요청=1, Pass Turn Move=0·Analysis 요청=0, 중복 GameResult=0, 중복 Rating 반영=0, 동일 game_id+move_no 중복 표시=0, BLACK+WHITE=100%, stale 상태/분석 변경=0, 분석 지연·실패로 인한 Turn·Result·Rating 차단=0 |
| 실패 판정 | 위 불변식 중 하나라도 위반하거나 ACK 응답 간 결과가 불일치                                                                                                                                                                             |
| 복구      | 해당 Run 중단·쓰기 격리, DB Snapshot과 Event Log로 영향 game_id 식별 후 승인된 데이터 복구                                                                                                                                             |
| 증거      | 요청/응답·BoardAnalysis JSON, DB Count/Checksum, Event/Analysis Timeline·처리시간, k6 결과, Run ID                                                                                                                                     |


<a id="test-rt-02"></a>

#### RT-02 Backend 장애 후 상태 복원·오패배 방지

| **필드**  | **실행 설계**                                                                                      |
|-----------|----------------------------------------------------------------------------------------------------|
| 전제 조건 | 활성 Room/Game/Turn과 Board Snapshot 확보, 30초 재접속 유예, Rating 전후값 저장                    |
| 실행 절차 | 투표 중 Backend Pod를 삭제하고 Client 재연결, Snapshot 수신 후 Game 종료 여부와 결과/Rating을 비교 |
| 관측 지점 | HTTP/WSS, Room·Participant/Team·Game·Turn·Board, GameResult·RatingHistory, Pod Event               |
| 성공 기준 | 상태가 권위 Snapshot과 일치하고 장애 원인 몰수·공동 패배·Rating 반영은 0건                         |
| 실패 판정 | 상태 불일치·유효 시간 내 재연결 실패·시스템 장애를 사용자 패배로 확정                              |
| 복구      | 새 Backend에서 DB/Redis 기준 Snapshot 재구성; 정상 복구 불가 시 SYSTEM_INVALID와 Rating 미반영     |
| 증거      | 장애 전후 Snapshot JSON, WSS Timeline, GameResult/Rating Query, Pod Log/Event                      |

기존 WebSocket Connection은 새 Pod로 자동 Rebalance되지 않는다. 기존 연결 유지와 Scale-out 이후 신규 연결 분산을 별도로 측정한다. ANALYSIS의 지연·실패는 사용자 게임의 권위 경로를 차단하지 않아야 한다.

<a id="section-17"></a>

## 17. Backup/Restore·Before/After·Evidence


<a id="section-17-1"></a>

### 17.1 Backup·전송·Retention

| **대상** | **생성**                           | **보호 위치**                | **검증**                   |
|----------|------------------------------------|------------------------------|----------------------------|
| MariaDB  | mariadb-backup Full 1일 1회 시작값 | NFS Staging + loadgen Disk   | 최근 7개, 실제 Restore     |
| etcd     | Snapshot + status/hash             | loadgen Disk                 | 위험 실험 전 + 설계안 주기 |
| Redis    | AOF/PVC + MariaDB 권위 데이터      | NFS + 필요한 Evidence Export | Pod/NFS 장애별 복구        |
| Jenkins  | Controller PVC + JCasC/Plugin Lock | NFS + Git 선언               | 재배치·재생성              |
| Grafana  | Dashboard/Datasource GitOps        | Deployment Repository        | UI 변경은 비공식           |
| Evidence | Run 종료 즉시 Raw Export           | loadgen 별도 Disk            | Run ID+Checksum+요약 Git   |


<a id="section-17-2"></a>

### 17.2 Backup/Restore Test Case


<a id="test-dr-01"></a>

#### DR-01 MariaDB Backup/Restore

| **필드**  | **실행 설계**                                                                                               |
|-----------|-------------------------------------------------------------------------------------------------------------|
| 전제 조건 | mariadb-backup Full·prepare 완료, loadgen 보호본 SHA-256 일치, 격리 Restore VM/Schema와 승인 확보           |
| 실행 절차 | 빈 Data Directory에 Restore→권한 복원→DB 기동→Application Read-only 검증→필요 시 서비스 연결                |
| 관측 지점 | Backup/prepare log, DB startup, row count·FK·Checksum, Application Query, 시작/완료 시각                    |
| 성공 기준 | Restore 성공, Member·MemberStats·Move·GameResult·RatingHistory 표본 PK/FK/Count/Checksum 일치, RTO/RPO 산출 |
| 실패 판정 | Checksum 불일치·DB 기동 실패·표본 누락/중복·승인 없는 운영 연결                                             |
| 복구      | 격리 환경 폐기 후 보호본 재검증; 다른 보존본으로 재시도하고 원본은 변경하지 않음                            |
| 증거      | Backup/Restore log, SHA256SUMS, 표본 Query CSV, RTO/RPO Worksheet                                           |


<a id="test-dr-02"></a>

#### DR-02 etcd Snapshot Restore

| **필드**  | **실행 설계**                                                                                   |
|-----------|-------------------------------------------------------------------------------------------------|
| 전제 조건 | etcdutl snapshot status/hash PASS, 전체 Control Plane 중단 절차와 target cluster 승인           |
| 실행 절차 | 격리된 경로에 snapshot restore, 세 Member 설정·Manifest 복원, quorum/API·Kubernetes Object 확인 |
| 관측 지점 | Snapshot status, Member list, etcd health, API, Namespace/Deployment/Secret metadata            |
| 성공 기준 | 3 Member quorum·API 복구, 핵심 Object 표본 일치, Restore 시간 산출                              |
| 실패 판정 | 기존/복구 Cluster 동시 기동·Member 불일치·quorum/API 실패·Snapshot hash 오류                    |
| 복구      | 복구 Cluster 중단·격리, Snapshot/Member 설정 재검증 후 전체 절차 재실행                         |
| 증거      | snapshot status/hash, member/endpoint health, API Object export, Timeline                       |


<a id="test-dr-03"></a>

#### DR-03 Redis AOF/PVC Recovery

| **필드**  | **실행 설계**                                                                                                                                                                                                                                                    |
|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 전제 조건 | AOF/PVC 보호본과 MariaDB 권위 Snapshot, Stream Pending Baseline, 전용 복구 Namespace                                                                                                                                                                             |
| 실행 절차 | PVC/AOF 연결 후 Redis 기동, Runtime State·Stream 복원, MariaDB와 비교하고 Pending 재처리. AOF/PVC 복구 불가 상태에서 DATA로부터 파생 가능한 범위와 불가능한 활성 상태를 분리 판정                                                                                |
| 관측 지점 | AOF check/startup, Session·Room·Participant·Ready·Game·Turn·Current Vote, Board·Move·GameResult, Stream/Pending/ACK, stale 결과                                                                                                                                  |
| 성공 기준 | AOF/PVC로 복구한 범위가 권위 DB와 일치하고 작업 유실·중복 권위 반영·stale 오표시가 없으며, DATA에 없는 활성 상태를 전체 자동 복원했다고 과장하지 않음                                                                                                            |
| 실패 판정 | AOF 손상·DB 불일치·중복 결과·게임 권위 경로 중단·비영속 활성 상태의 근거 없는 복원                                                                                                                                                                               |
| 복구      | 손상 AOF 격리. MariaDB Move·GameResult에서 Board·종료 Game 등 파생 가능 상태만 재구성하고 Streams를 재발행/격리하며, GuestSession·Participant·Ready·Current Vote 등을 권위 있게 복구할 수 없는 Room/Game은 SYSTEM_INVALID로 처리하고 전적·Rating에 반영하지 않음 |
| 증거      | AOF Check, 상태 비교 JSON, Pending 전후 CSV, Pod Log/Event                                                                                                                                                                                                       |


<a id="test-dr-04"></a>

#### DR-04 NFS/PVC Restore

| **필드**  | **실행 설계**                                                                                   |
|-----------|-------------------------------------------------------------------------------------------------|
| 전제 조건 | 전용 Export/PVC Backup·SHA-256, UID/GID·Mount Option·StorageClass 기록, 소비자 중단 승인        |
| 실행 절차 | NFS Export/권한 복원→Provisioner 연결→PVC 재생성/바인딩→Test Pod·Jenkins/Redis 선택 소비자 검증 |
| 관측 지점 | Export/mount, PV/PVC, 파일 Owner/Mode, Checksum, Pod Event, 소비자 Health                       |
| 성공 기준 | PVC Bound·Read/Write·Checksum·UID/GID 일치, 소비자 정상화와 Restore 시간 산출                   |
| 실패 판정 | 잘못된 Export 덮어쓰기·Checksum/권한 불일치·PVC 미바인딩·소비자 데이터 손상                     |
| 복구      | 소비자 중단 유지, 잘못된 Volume 격리 후 보호본/StorageClass 입력을 재검증해 재실행              |
| 증거      | Export/PV/PVC YAML, ls/stat, SHA256SUMS, Test Pod I/O, Timeline                                 |

전송은 전용 제한 계정과 SSH/SFTP 또는 rsync over SSH를 사용한다. 임시 파일로 전송한 뒤 SHA-256을 비교하고 원자적으로 최종 이름으로 이동한다. Backup과 부하 시험은 동시에 실행하지 않는다. loadgen VM의 별도 Virtual Disk는 서비스 Physical Server 4대와 NFS에서 분리된 복구 단위지만 5번째 Windows Host 한 대에 의존하므로 off-site·Site DR·HA Storage로 표현하지 않는다.


<a id="section-17-3"></a>

### 17.3 Before/After

| **ID**         | **Before**                             | **Change/After**                  | **Metric**                                    |
|----------------|----------------------------------------|-----------------------------------|-----------------------------------------------|
| Q-01 구축      | 역할별 대표 CentOS Guest 수동 Baseline | 동일 초기상태의 Ansible Role 적용 | 총/단계 시간, Command·개입, 누락, Gate 통과율 |
| Q-02 재실행    | 최초 실행                              | 동일 조건 2회차                   | changed/ok/failed, Handler, 시간              |
| Q-03 Drift     | 수동 변경                              | Check/Apply                       | 감지 수, 재복원 시간, 남은 편차               |
| Q-04 Worker    | 정상                                   | Worker 장애·Rebuild               | Recovery Time, 실패 요청, MTTR                |
| Q-05 LB        | 정상 Owner                             | Active LB 장애                    | VIP 이동, 실패 요청, Reconnect                |
| Q-06 DB        | Primary 정상                           | 승격 복구                         | Write Recovery, RPO, 실패 Transaction         |
| Q-07 CI/CD     | 수동 Build/배포                        | Jenkins+Harbor+GitOps             | CI/CD/Rollback 시간, 수동 단계                |
| Q-08 HPA       | Static Replica                         | HPA                               | RPS, p95/p99, Error, Replica                  |
| Q-09 DR        | 정상 데이터                            | Backup/Restore                    | Backup·Restore 시간, 무결성, RPO              |
| Q-10 대상 추가 | 기존 Node 수동 추가                    | Inventory/host_vars 추가          | 변경 파일·명령·개입, Ready까지 시간           |

전체 18개 VM을 수동과 자동으로 각각 다시 구축하지 않는다. 역할별 대표 CentOS Guest에서 수동 1회 성공을 Before로 확보한 뒤 동일 OS Image, VM 자원, Network 상태와 완료 Gate에서 Role을 적용해 After를 측정하고, 이후 다른 대상 적용과 동일 조건 재실행으로 재현성과 멱등성을 검증한다. 가능하면 각 비교를 3회 반복해 Median과 편차를 기록하며 실제 측정 전에는 개선 수치를 기재하지 않는다.


<a id="section-17-4"></a>

### 17.4 Evidence 구조

```text
evidence/YYYYMMDD-HHMMSS-<scenario>/
├── metadata.yml        # Run ID, Commit, Version, Inventory, Operator
├── foundation/         # Windows·VMware 수동 Checklist와 검증 증거
├── ansible/            # recap, changed task, timing
├── network/            # route, reachability, VIP, ARP
├── kubernetes/         # nodes, events, workload, metrics
├── data/               # replication, transaction, checksum
├── observability/      # query export, alert timestamps
├── e2e/                # HTTP/WSS/Vote/Move result
└── SHA256SUMS
```

> **Evidence 원칙** TXT·JSON·CSV 등 Raw Evidence를 우선하고 Screenshot은 보조로 사용한다. Secret·Token·Private Key·Password는 수집하지 않는다. Git에는 요약·metadata·Checksum·외부 보관 위치를 남긴다.

<a id="section-18"></a>

## 18. 추적 Matrix·구현 체크리스트·07 이관 경계


<a id="section-18-1"></a>

### 18.1 Automation ID 단일 기준표

| **Automation ID** | **Stage**      | **Role/Owner**                                                          | **Output**                                    |
|-------------------|----------------|-------------------------------------------------------------------------|-----------------------------------------------|
| AUT-FND           | 00·10          | preflight·linux_base                                                    | 수동 기반 완료 증거·CentOS Guest Baseline     |
| AUT-NET           | 20             | vrouter·host_firewall·loadgen_support                                   | Route·Host DNS·Port 최종 상태                 |
| AUT-END           | 30             | haproxy·keepalived·internal_ca·tls_deploy                               | Common VIP·4 Listener·TLS                     |
| AUT-K8S           | 40             | container_runtime·kubernetes_prereq/bootstrap·calico·coredns_records    | CP/Worker·CNI·Pod DNS                         |
| AUT-PLT           | 70             | Argo Root App                                                           | Gateway·Provisioner·Cluster Add-on            |
| AUT-DATA          | 50             | mariadb·maxscale·nfs_server                                             | DB·Storage                                    |
| AUT-STATE         | 70·80          | Redis/ANALYSIS GitOps 선언                                              | Runtime State·Streams 격리                    |
| AUT-BKP           | 50·recovery    | backup_transfer·restore playbooks                                       | Backup·Checksum·Restore                       |
| AUT-DEL           | 60·70          | harbor·argocd_bootstrap·Jenkins GitOps                                  | Commit→Digest→Healthy                         |
| AUT-OBS           | 70             | Prometheus·Loki·Alertmanager GitOps · node_exporter_linux · alloy_linux | Kubernetes/외부 CentOS Guest Metric·Log·Alert |
| AUT-MET           | 90·experiments | loadgen_support·validation·evidence                                     | Load·Timeline·Raw Evidence                    |
| AUT-APP           | 80             | Application GitOps + E2E validation                                     | Frontend·Backend·ANALYSIS                     |

| **Requirement** | **01–02 Source ID** | **검증 목표**             | **Automation**                          | **Test/Evidence**      |
|-----------------|---------------------|---------------------------|-----------------------------------------|------------------------|
| REQ-AUTO        | P-05·M-05·A-01~04   | 반복 구축·편차·복구       | AUT-FND·AUT-NET·AUT-K8S·AUT-PLT·AUT-BKP | G-01~09·Q-01~05·Q-10   |
| REQ-HA          | P-03·M-03           | LB·CP·Worker·DB 복구      | AUT-END·AUT-K8S·AUT-DATA                | F-01~06·FD-01~04·RT-02 |
| REQ-DR          | P-06·M-04·A-04      | Backup/Restore·RTO/RPO    | AUT-BKP·AUT-DATA·AUT-STATE              | DR-01~04·Q-09          |
| REQ-CICD        | A-03                | Build·Registry·GitOps     | AUT-DEL·AUT-PLT                         | G-06·F-10~11·Q-07      |
| REQ-OBS         | 검증축 공통         | Metric·Log·Alert·Evidence | AUT-OBS·AUT-MET                         | G-07·F-12~13           |
| REQ-HPA         | P-04·M-02           | 단계 부하·Scale-out       | AUT-MET·AUT-APP                         | L-01·L-04·Q-08         |
| REQ-RT          | P-01~03·M-01·M-03   | WSS·Turn·Move·복원        | AUT-APP·AUT-STATE                       | G-08·L-02~03·RT-01~02  |
| REQ-AI          | P-04·M-02           | 공식 Move 후 비권위 분석  | AUT-STATE·AUT-APP                       | F-07·L-05·RT-01        |


<a id="section-18-2"></a>

### 18.2 구현 전 체크리스트

- 교육장 L2에서 Unicast VRRP, Protocol 112, Gratuitous ARP, 추가 MAC과 Common VIP 이동을 확인한다.

- Windows Host IP·VMnet 번호·VM vNIC 수량과 CentOS Guest의 실제 Interface 이름, SSH 계정·Key, become 정책과 긴급 접근 경로를 확인한다.

- CentOS Stream 9 Repository, Kubernetes minor Repository와 모든 Image/Chart 다운로드 가능성을 확인한다.

- Calico Path MTU, VXLAN 4789/UDP, CIDR 중복과 Return Path를 측정한다.

- MaxScale 라이선스·Repository와 두 Instance의 Monitor 권한 방식을 확인한다.

- NFS Export, Mount Option, UID/GID, Provisioner Chart/Image 조합을 검증한다.

- Jenkins Plugin Lock, Rootless BuildKit SecurityContext와 Harbor TLS Trust를 검증한다.

- Prometheus/Loki Local PV 경로·용량·Node Affinity와 Evidence Export 경로를 확정한다.

- SMTP Sender/Receiver와 선택적 Discord 중계 호환성을 확인한다.

- 위험 실험 전 Backup·Checksum·Rollback·Stop Condition과 승인자를 확인한다.


<a id="section-18-3"></a>

### 18.3 구현 시 실측 후 동결할 값

| **항목**                       | **동결 근거**                                     |
|--------------------------------|---------------------------------------------------|
| VMnet 번호·Guest Interface     | VMware와 CentOS Guest 실제 조회 및 관리 경로 시험 |
| Calico MTU                     | Worker 간 최소 Path MTU 측정                      |
| HPA target/min/max             | Static Capacity Curve                             |
| requests/limits                | Application·Jenkins·Observability 실사용량        |
| Prometheus/Loki 실제 Retention | 수집률·Disk 증가량                                |
| RTO/RPO Threshold              | 최초 장애·복구 Baseline                           |
| Alert Group/Repeat Interval    | 실제 중복·통보 지연 시험                          |
| Backup 주기·Retention          | 변경량·Backup/Restore 시간·Disk                   |
| Webhook vs Polling             | GitHub→On-prem Inbound Reachability               |


<a id="section-18-4"></a>

### 18.4 07 MVP 이관 경계

| **원 목표 설계**                   | **07에서 판단할 축소**         | **보존해야 할 검증 가치**                         |
|------------------------------------|--------------------------------|---------------------------------------------------|
| LB 2 + Common VIP                  | LB 1 고정 IP 가능              | 포트별 경로·장애 제한을 정직하게 기록             |
| MaxScale 2                         | MaxScale 1 가능                | DB Stable Endpoint·Primary 복구·RPO               |
| Windows Host·VMware 수동 공통 기반 | 동일 유지; VM 생성 자동화 제외 | Runbook·대표 수동/Role 비교·Guest Baseline 재현성 |
| 전체 장애 Matrix                   | 대표 장애 우선                 | Worker·DB·NFS·CI/CD·Evidence                      |
| 전체 관측 기능                     | 장기/HA 저장 제외              | 감지·Timeline·Metric/Log Export                   |
| 완전 자동 DB 승격 없음             | 동일 유지                      | Fencing·승인·무결성                               |
| 외부 보호 Disk                     | 동일 유지                      | Backup/Restore·Checksum; Site DR 과장 금지        |

> **07 원칙** MVP는 기술을 크게 제거한 데모가 아니라 원 목표 구조의 핵심 자동화·장애·복구·부하·배포 Evidence를 유지하면서 4명·4주 범위에 맞게 HA 복잡도를 감소한 최소 구현이다.


<a id="summary"></a>

## 요약

이 설계는 최신 서비스 계약과 물리 구조를 Ansible 코드·GitOps·Runtime 책임으로 나누고, 정상 일치·재실행·부분 실패·Drift·위험한 Recovery를 서로 다른 실행 경계로 정의한다. 또한 Native HA와 Ansible Rebuild를 구분하여 측정하고, 부하·HPA·CI/CD·Backup/Restore·ANALYSIS 결과를 동일 Run ID와 Raw Evidence에 연결한다. 최종 가치는 기술 개수가 아니라 같은 시작 조건에서 재현되고, 실패 시 원인과 복구 책임을 설명하며, 실제 측정값으로 개선 여부를 증명할 수 있다는 데 있다.


<a id="official-technical-sources"></a>

## 공식 기술 확인 자료

- Kubernetes Releases  
  <https://kubernetes.io/releases/>

- Kubernetes Patch Releases  
  <https://kubernetes.io/releases/patch-releases/>

- Calico Kubernetes Requirements  
  <https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements>

- MariaDB Server Releases  
  <https://mariadb.org/mariadb/all-releases/>

- MariaDB MaxScale Release Notes  
  <https://mariadb.com/docs/release-notes/maxscale>

- Redis Releases  
  <https://github.com/redis/redis/releases>

- Jenkins LTS Changelog  
  <https://www.jenkins.io/changelog-stable/>

- Jenkins Java Support Policy  
  <https://www.jenkins.io/doc/book/platform-information/support-policy-java/>

- Harbor Releases  
  <https://github.com/goharbor/harbor/releases>

- Argo CD Releases  
  <https://github.com/argoproj/argo-cd/releases>

- NGINX Gateway Fabric Releases  
  <https://github.com/nginx/nginx-gateway-fabric/releases>

- NFS Subdir External Provisioner  
  <https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner>

- Metrics Server Releases  
  <https://github.com/kubernetes-sigs/metrics-server/releases>

- Grafana Loki Releases  
  <https://github.com/grafana/loki/releases>

- Grafana Alloy Releases  
  <https://github.com/grafana/alloy/releases>

- Grafana k6 Releases  
  <https://github.com/grafana/k6/releases>

- BuildKit Rootless Mode  
  <https://github.com/moby/buildkit/blob/master/docs/rootless.md>

Release·Compatibility·다운로드 가능성은 구현 착수 시 다시 확인하고 Version Lock Sheet로 동결한다. 문서의 선택은 공식 자료만으로 결정되지 않으며, 최신 01–05의 요구·제약·물리 배치와 팀의 4주 구현 범위를 함께 기준으로 한다.
