# 石나가는 판단

*물리 아키텍처*

논리 구조를 실제 On-premise Server·VM·Network·Storage·HA 배치로 구체화하고, 장애 검증과 자동화가 가능한 실행 환경을 정의한다.

> **설계 기준** 이 문서는 원 목표 구조를 정의한다. Physical Server 4대와 VM 18개를 기준으로 하며, 별도로 도출할 MVP 축소 구조는 포함하지 않는다. 기술 선택은 선행 문서에서 확정한 현재 구조를 따르고, 구현 후 측정되지 않은 수치는 최초 구축을 위한 계획값으로 구분한다.

## 목차

1. [문서 목적과 설계 원칙](#section-1)  
2. [물리 아키텍처 결정 요약](#section-2)  
3. [Physical Server와 VM 배치](#section-3)  
4. [VM 및 Kubernetes Workload 자원 계획](#section-4)  
5. [Network·IP·Static Routing](#section-5)  
6. [Common VIP·HAProxy·Keepalived](#section-6)  
7. [TLS·Firewall·NetworkPolicy](#section-7)  
8. [Kubernetes Node·Pod·Service Network](#section-8)  
9. [Data·Runtime State 물리 경계](#section-9)  
10. [External NFS와 Persistent Storage](#section-10)  
11. [CI/CD·GitOps·Image Delivery](#section-11)  
12. [Observability·Alerting·Automation](#section-12)  
13. [HA·SPOF·Failure Domain](#section-13)  
14. [Physical Server 장애 영향](#section-14)  
15. [검증 목표·확장 조건](#section-15)  
16. [자동화·테스트 설계 이관](#section-16)  
17. [기술적 사실 확인 근거](#section-17)

<a id="section-1"></a>

## 1. 문서 목적과 설계 원칙

서비스 요구사항 및 기능 명세, 핵심 문제 및 검증 목표, 논리 역할 및 서비스 목록, 기술 비교 및 논리 아키텍처에서 확정한 책임과 기술을 실제 서버·VM·네트워크·스토리지 배치로 변환한다. 이 문서에서는 Server/VM 수, OS·가상화 기반, 자원 시작값, IP/Subnet, Common VIP, 방화벽 경계, Failure Domain과 장애 영향 범위를 결정한다.

| **원칙** | **적용 기준** |
| --- | --- |
| 원 목표 구조 | Physical Server 4대, MaxScale 2대, LB 2대의 원 목표 구조를 사용한다. Common VIP HA는 교육장 L2에서 VRRP·VIP 이동이 확인될 때 적용하며, MVP 축소안은 07 이관 경계로만 기록한다. |
| 역할과 배포 단위 분리 | 논리 역할이 곧 VM을 의미하지 않는다. Frontend·Backend·Redis·ANALYSIS·Jenkins·Argo CD·Observability는 Kubernetes Workload다. |
| 계획값과 실측값 구분 | CPU·Memory·Disk는 최초 구축 시작값이다. 성능과 장애 실험 결과를 성과처럼 선기재하지 않는다. |
| 장애 경계 명시 | HA 구성뿐 아니라 NFS·Harbor·Ansible Controller 등 남는 SPOF와 복구 책임을 함께 기록한다. |
| 교차 일관성 | 배치, 자원, IP, Port, Storage, 장애 영향과 검증 항목이 동일한 기준값과 책임 경계를 사용한다. |

<a id="section-1-1"></a>

### 1.1 선행 설계 입력과 물리화 대상

| **선행 문서** | **확정 입력** | **물리화 결과** |
| --- | --- | --- |
| 서비스 요구사항 및 기능 명세 | Room·Game·Turn·Vote·Move·Result의 서버 권위 규칙, Member·Guest 경계, 재접속과 실시간 상태 수렴 | Backend를 특정 인스턴스의 Local State에 종속시키지 않고 Redis Runtime State와 MariaDB Persistent Data의 배치·복구 경계를 분리 |
| 핵심 문제 및 검증 목표 | HA·HPA·DR·CI/CD·관측성, On-premise 자동화, 장애·복구와 Before/After 정량 검증 | Control Plane·Worker·LB·DB·Storage Failure Domain과 장애 주입 지점, RTO/RPO·복구시간·자동화 측정 항목으로 변환 |
| 논리 역할 및 서비스 목록 | WEB·Application Roles·ANALYSIS·STATE·DATA·MONITORING·LOGGING·AUTOMATION의 책임 분리 | 논리 역할을 VM 수와 동일시하지 않고 Kubernetes Workload, 외부 VM, Node·Storage·Network 경계에 배치 |
| 기술 비교 및 논리 아키텍처 | kubeadm·Calico·Gateway API·MariaDB/MaxScale·External NFS·Prometheus/Loki Local Storage·Grafana GitOps·Jenkins/Argo CD·Harbor·Alertmanager·Alloy | Physical Server 4대·VM 18개, Common VIP, Private Subnet·Static Route, 공유 NFS와 관측 Local Storage, 외부 Linux 로그 수집 경로, 자원 시작값과 SPOF·HA 구조로 구체화 |

<a id="section-2"></a>

## 2. 물리 아키텍처 결정 요약

| **항목** | **결정** | **설계 이유** |
| --- | --- | --- |
| Physical Server | 4대 | Control Plane, Worker, DB, LB, Storage와 관리 기능의 Failure Domain을 분산하고 실제 장애를 주입할 수 있는 기준 규모다. |
| VM | 총 18개 | VRouter 4, Control Plane 3, Worker 2, LB 2, MariaDB 2, MaxScale 2, Harbor 1, NFS 1, Ansible Controller 1이다. |
| Kubernetes | Control Plane 3 + Worker 2 | stacked etcd quorum과 단일 Worker 장애 후 재스케줄·수용성을 검증한다. |
| 외부 진입 | HAProxy + Keepalived 2대(원 목표) | Common VIP 소유권 전환과 포트별 L4 전달·Health Check를 분리한다. 실제 VIP HA 적용은 교육장 L2 사전 검증을 통과해야 한다. |
| Common VIP | 10.1.93.90 | :80/:443 서비스, :6443 Kubernetes API, :3306 DB Access를 하나의 VIP에서 포트로 구분한다. |
| Persistent Data | MariaDB Primary/Replica + MaxScale 2 | DB 저장·복제 책임과 DB 접근·Routing 책임을 분리하고 MaxScale 자체 장애도 수용한다. |
| Shared Storage | External NFS 1대 | Redis AOF·Jenkins Controller 등 Shared File PVC를 제공하되 단일 NFS SPOF와 복구시간을 숨기지 않는다. Prometheus·Loki 저장에는 사용하지 않는다. |
| CI/CD | Jenkins·Argo CD in-cluster / Harbor external | Jenkins를 Kubernetes Workload로 실행해 특정 VM에 종속되지 않으며 Build, Registry, GitOps 책임을 분리한다. |
| Observability | Prometheus·Alertmanager·Grafana·Loki·Alloy in-cluster + 외부 Alloy Agent | Prometheus·Loki는 Worker Local Storage에 단기 보존하고, Grafana 설정은 GitOps로 복구한다. 외부 Linux 로그도 동일 Loki로 수집한다. |
| OS·Virtualization | Windows 10 Enterprise Host + VMware Workstation Pro 25H2 / CentOS Stream 9 Guest | 교육장의 Windows Host와 VMware Workstation을 유지하고, CentOS Guest가 SSH 준비된 이후부터 Ansible 관리 경계를 적용한다. |
| Management·Load | Ansible Controller + 외부 관리·부하 Server | SSH 준비된 CentOS Guest VM과 Node를 관리하고, k6 부하는 Cluster 밖의 별도 외부 지원 Server에서 발생시켜 측정 왜곡을 줄인다. 외부 지원 Server의 별도 Disk는 Backup·Raw Evidence 보관에 사용하되 부하 시험과 Backup 작업을 동시에 실행하지 않는다. |

<a id="section-3"></a>

## 3. Physical Server와 VM 배치

각 Physical Server는 Intel Core i5-13400 10 Core / 16 Thread, RAM 64GB DDR4-3200, Windows 10 Enterprise 64-bit Host를 기준으로 하며, 각 Host의 C 드라이브에 프로젝트용 여유 공간 500GB를 확보한다. VM 자원은 단순 설치 최소치에 맞추지 않고, 공식 권장치가 있는 구성요소는 권장치를 우선하며 실제 Workload와 단일 장애 수용 여유를 더해 결정한다. HA Pair와 quorum 구성요소를 동일한 Physical Server에 집중하지 않으며, 서버 추가 자체를 완성도로 간주하지 않는다.

<a id="section-3-1"></a>

### 3.1 OS·Virtualization·관리 경계

Physical Server 4대의 Host OS는 Windows 10 Enterprise 64-bit로 유지하고, VMware Workstation Pro 25H2(25.0.0.24995812)에서 실행하는 VM 18개의 Guest OS는 CentOS Stream 9로 통일한다. 팀의 RHEL 9 계열 학습 경험과 Package·운영 자료의 성숙도를 우선한 선택이다. CentOS Stream 9의 공식 EOL은 2027-05-31이므로 1차 프로젝트 기간에는 사용 가능하지만, 장기 운영 또는 2차 확장 전에는 CentOS Stream 10 등으로 전환 여부를 재검토한다. 세부 Package와 Component Release는 06에서 호환성을 확인한 뒤 고정한다.

가상화 기반은 교육장에 설치된 VMware Workstation Pro를 사용한다. VMnet0는 Intel Ethernet Connection I219-LM 물리 NIC에 Bridged로 고정하고, 각 Server의 Private Subnet은 DHCP를 끈 전용 Host-only VMnet으로 구성한다. VRouter는 외부 Bridged vNIC와 내부 Host-only vNIC 2장을 사용하고, 일반 내부 VM은 Host-only vNIC 1장, LB는 Bridged vNIC 1장을 사용한다. 프로젝트 VM은 VMnet8 NAT를 사용하지 않는다. 구축 전에는 VMnet0 Bridged Guest의 외부 통신과 교육장망의 추가 MAC 허용 여부를 실제 연결로 확인한다.

Windows Host IP 설정, Virtual Network Editor 구성, VM Hardware·vNIC 생성, CentOS 설치, 초기 계정과 SSH 준비는 공통 절차에 따라 수동으로 수행한다. Ansible Controller는 SSH Key와 privilege escalation을 사용해 SSH 준비가 끝난 CentOS Guest VM과 Kubernetes Node를 관리한다. Guest OS Baseline, VM 내부 Network 설정, VRouter·Route·Firewall, kubeadm과 외부 Infra는 자동화 대상으로 두되 Windows Host·VMware 설정과 VM 생성은 1차 프로젝트의 Ansible 자동화 범위에서 제외한다. 수동 절차는 Runbook과 검증 증거로 남기며 자동화 완료로 표현하지 않는다.

| **Server** | **배치 VM** | **주요 책임** | **Failure Domain** |
| --- | --- | --- | --- |
| Server-01 | VRouter-01<br>CP-01<br>Worker-01<br>MariaDB-02 Replica | Kubernetes Runtime<br>DB Replica | 장애 시 CP·Worker·DB Replica가 함께 손실되지만 CP quorum과 DB Primary는 다른 Server에 남는다. |
| Server-02 | VRouter-02<br>CP-02<br>Worker-02<br>MariaDB-01 Primary | Kubernetes Runtime<br>DB Primary | CP·Worker·DB Primary가 동시에 영향을 받으므로 Replica 승격과 Runtime 지속을 함께 검증하는 핵심 복합 장애 지점이다. |
| Server-03 | VRouter-03<br>CP-03<br>LB-01<br>MaxScale-01<br>Harbor | Control Plane<br>L4 Endpoint<br>DB Proxy<br>Registry | CP, LB, MaxScale의 상대 인스턴스를 다른 Server에 둔다. Harbor 장애는 재Pull·재배포·확장에 영향을 줄 수 있다. |
| Server-04 | VRouter-04<br>LB-02<br>MaxScale-02<br>NFS<br>Ansible Controller | L4 Endpoint<br>DB Proxy<br>Storage<br>Automation | LB·MaxScale의 두 번째 Failure Domain이다. NFS와 Ansible Controller는 복구를 검증할 단일 인스턴스다. |

> **VM 수 산정** Server-01 4개 + Server-02 4개 + Server-03 5개 + Server-04 5개 = 총 18개다. Kubernetes 내부의 Pod, Deployment, StatefulSet, DaemonSet은 VM 수에 포함하지 않는다.

<a id="section-4"></a>

## 4. VM 및 Kubernetes Workload 자원 계획

<a id="section-4-1"></a>

### 4.1 VM별 초기 자원

VM Resource는 최초 구축과 검증을 위한 시작값이다. 각 Host의 C 드라이브에 프로젝트용 여유 공간 500GB를 확보하고 Virtual Disk는 Thin Provisioning으로 생성한다. VMware Snapshot은 단기 작업 전 복구점으로만 사용하고 확인 후 삭제·통합하며 Backup으로 간주하지 않는다. 실제 Disk 사용량과 I/O를 관측하고, vCPU 합계와 Physical CPU saturation·ready time을 함께 확인한다.

| **VM 역할** | **수량** | **vCPU / RAM / Disk** | **근거와 조정 기준** |
| --- | --- | --- | --- |
| VRouter | 4 | 1 / 1GB / 20GB | IP forwarding, Static Route와 Firewall 중심의 경량 역할 |
| K8s Control Plane | 3 | 2 / 4GB / 50GB | kubeadm Control Plane과 stacked etcd 운영 시작값 |
| K8s Worker | 2 | 8 / 28GB / 140GB | Runtime, ANALYSIS, Jenkins, GitOps, Observability와 Prometheus·Loki 단기 Local Storage, 단일 Worker 장애 수용 여유 |
| HAProxy + Keepalived | 2 | 2 / 2GB / 20GB | Common VIP, L4 Listener, Backend Health Check |
| MariaDB | 2 | 4 / 12GB / 100GB | Primary/Replica와 성능·복구 실험 시작값 |
| MaxScale | 2 | 2 / 4GB / 30GB | DB Routing·Monitor·Failover 경계 |
| Harbor | 1 | 4 / 8GB / 160GB | 공식 권장 Hardware 수준과 Image·GC·Scanner I/O 고려 |
| NFS | 1 | 2 / 8GB / 200GB | Redis·Jenkins Shared PVC와 MariaDB Backup 복원 실험용 임시 보관 수요의 시작값 |
| Ansible Controller | 1 | 2 / 4GB / 40GB | Physical Server·VM Inventory, Role, Template, Artifact와 원격 실행 |

| **Server** | **vCPU 합계** | **RAM 합계** | **Disk 합계** | **수용성 판단** |
| --- | --- | --- | --- | --- |
| Server-01 | 15 | 45GB | 310GB | Worker와 DB Replica 경합을 관측하되 RAM·Disk 여유를 유지한다. |
| Server-02 | 15 | 45GB | 310GB | Worker와 DB Primary 경합이 성능 결과를 왜곡하는지 우선 확인한다. |
| Server-03 | 11 | 19GB | 280GB | Harbor Image·Scanner I/O와 CP/LB/MaxScale 영향 분리를 관측한다. |
| Server-04 | 9 | 19GB | 310GB | NFS 용량·I/O와 Backup 증가량을 우선 관측한다. |

> **자원 산정 원칙** 최소 요구치는 설치 가능 여부를 확인하는 참고값일 뿐 물리 배치의 목표값으로 사용하지 않는다. 공식 권장치가 제공되는 구성요소는 권장치를 우선하고, 권장치가 명확하지 않은 구성요소는 담당 Workload·동시 부하·영속 데이터·장애 시 재수용 예산을 기준으로 프로젝트 권장 시작값을 산정한다. 현재 Worker 8 vCPU / 28GB / 140GB와 Harbor 4 vCPU / 8GB / 160GB는 이 원칙을 반영하며, 실측 결과 없이 더 작은 값으로 낮추지 않는다.

<a id="section-4-2"></a>

### 4.2 Worker 수용 예산

정상 상태에서는 두 Worker에 필수 Replica와 운영 Workload를 분산한다. Worker 1대 장애 시 사용자 요청·상태·Gateway·복구 기능을 우선하고, Jenkins Build Agent와 비필수 배치 작업은 중지한다. 아래 값은 살아남은 Worker 한 대에서 유지할 상시 Request 예산의 상한이며 실제 requests/limits는 부하 시험으로 확정한다.

| **Workload 범주** | **CPU Request 예산** | **Memory 예산** | **장애 시 정책** |
| --- | --- | --- | --- |
| Frontend·Backend·Gateway | 2.0 vCPU | 4.0GB | 사용자 요청·WebSocket·외부 진입 우선 유지 |
| Redis·ANALYSIS | 1.0 vCPU | 3.0GB | Redis 우선. ANALYSIS는 지연 또는 축소 가능 |
| Jenkins Controller·Argo CD | 0.75 vCPU | 2.5GB | Controller 유지, 신규 Build Agent 중지 |
| Prometheus·Alertmanager·Grafana·Loki·Alloy | 1.25 vCPU | 4.5GB | 핵심 Metric·Log·Critical Alert 우선 |
| Metrics Server·CoreDNS·kube-state-metrics·NFS Provisioner | 0.5 vCPU | 1.5GB | Cluster 운영과 PVC 제어 경로 유지 |
| kubelet·CNI·OS·Eviction 여유 | 1.0 vCPU | 3.0GB | Node 안정성 보호 |
| 합계 | 6.5 vCPU | 18.5GB | 8 vCPU / 28GB Worker의 장애 수용 시작 기준 |

> **일시성 Build 자원** 정상 상태에서 Kubernetes Plugin 기반 Jenkins Agent Pod는 Build마다 동적으로 생성하며, 초기 상한은 총 1.5~2 vCPU와 3~4GB RAM 범위로 둔다. Controller 상태는 PVC로 보존하지만 Build Workspace는 기본적으로 emptyDir 등 일시 영역을 사용해 NFS I/O와 불필요한 영속화를 줄인다. 보존이 필요한 Artifact는 Harbor 또는 별도 보관 대상으로 이동한다.

<a id="section-5"></a>

## 5. Network·IP·Static Routing

<a id="section-5-1"></a>

### 5.1 Network 구조

공통 외부 L2는 10.1.93.0/24를 사용하고, Windows Host의 물리 NIC에 고정한 VMnet0 Bridged를 외부망으로 사용한다. 각 Physical Server 내부에는 DHCP를 끈 전용 Host-only VMnet으로 별도 Private Subnet을 둔다. Server-01은 현재 VMnet2 192.168.51.0/24를 사용하고 Server-02~04도 같은 방식으로 192.168.52~54.0/24를 구성한다. Host Virtual Adapter는 Host-local 관리에만 사용하며 Default Gateway와 DNS를 설정하지 않는다. VRouter는 외부 Bridged vNIC와 내부 Host-only vNIC 사이에서 IP forwarding, Static Routing, Masquerade와 L3/L4 정책을 담당한다. Masquerade는 Private Subnet에서 외부망으로 나가는 통신에만 적용한다. 192.168.51.0/24~192.168.54.0/24 사이의 사설망 간 통신에는 NAT를 적용하지 않고 원본 Source IP를 유지한다. 외부 Interface는 /24, Gateway 10.1.93.254, DNS 8.8.8.8을 사용하고, 내부 VM의 Default Gateway는 각 VRouter의 192.168.51~54.10으로 설정한다. 물리 NIC 추가 없이 VMware vNIC를 사용하되 Host·VRouter·LB·VIP의 외부 IP를 중복시키지 않으며 프로젝트 VM은 VMnet8 NAT를 사용하지 않는다.

| **비교 기준** | **선택: Server별 Private Subnet + VRouter** | **대안: Node 공통 Bridged L2** |
| --- | --- | --- |
| 외부 IP·MAC | Host·VRouter·LB·VIP 중심으로 제한 | CP·Worker마다 외부 IP·MAC 필요 |
| 구축 난이도 | Route·Firewall·MTU 검증 필요 | Node 연결은 상대적으로 단순 |
| 자동화 가치 | Routing과 Firewall을 Ansible 검증 범위로 포함 | 네트워크 자동화 범위가 축소 |
| Failure Domain | Server Subnet 격리가 명확 | Physical Server 장애 중심 |

<a id="section-5-2"></a>

### 5.2 외부 IP와 내부 Subnet

| **Server** | **Host 외부 IP** | **VRouter 외부 IP** | **Private Subnet** | **내부 Gateway** |
| --- | --- | --- | --- | --- |
| Server-01 | 10.1.93.70 | 10.1.93.71 | 192.168.51.0/24 | 192.168.51.10 |
| Server-02 | 10.1.93.72 | 10.1.93.73 | 192.168.52.0/24 | 192.168.52.10 |
| Server-03 | 10.1.93.74 | 10.1.93.75 | 192.168.53.0/24 | 192.168.53.10 |
| Server-04 | 10.1.93.76 | 10.1.93.77 | 192.168.54.0/24 | 192.168.54.10 |

| **영역** | **주소 배정** |
| --- | --- |
| LB Node | LB-01 10.1.93.78 / LB-02 10.1.93.79 |
| Common VIP | 10.1.93.90 |
| Server-01 Private | CP-01 192.168.51.20 / Worker-01 .30 / MariaDB-02 .40 |
| Server-02 Private | CP-02 192.168.52.20 / Worker-02 .30 / MariaDB-01 .40 |
| Server-03 Private | CP-03 192.168.53.20 / MaxScale-01 .40 / Harbor .61 |
| Server-04 Private | MaxScale-02 192.168.54.40 / NFS .50 / Ansible Controller .70 |
| 예약 여유 | 10.1.93.93~99는 현재 구성요소에 임의 할당하지 않고 확장 여유로 보존 |
| 외부 관리·부하 Server | 5번째 Physical Server의 Windows Host(10.1.93.92)에서 실행하는 CentOS Stream 9 지원 VM loadgen(Bridged, 10.1.93.91; 사용 전 ARP·중복 확인, 서비스 VM 18개 산정에서 제외). loadgen VM은 C 드라이브에 생성한 별도 Virtual Disk를 Backup·Raw Evidence 보관에 사용하며, 6번째 PC는 사용하지 않는다. |

<a id="section-5-2-1"></a>

### 5.2.1 Hostname·이름 해석

기존 구성요소 명칭을 소문자 Hostname으로 사용하고 내부 접미사는 stone.test로 통일한다. 예: server-01, vrouter-01, cp-01, worker-01, mariadb-01, maxscale-01, lb-01, harbor, nfs, ansible, loadgen. 서비스 Endpoint는 service.stone.test, k8s-api.stone.test, db.stone.test가 Common VIP 10.1.93.90을 가리키고 harbor.stone.test는 Harbor 주소를 사용한다. 초기에는 Ansible이 CentOS Guest VM·Node·외부 지원 Linux VM의 /etc/hosts를 동일하게 배포하고, Windows Host의 이름 해석은 수동 설정 대상으로 구분한다. Node의 /etc/hosts는 Pod에 자동 상속되지 않으므로, Kubernetes Pod에서 필요한 stone.test Endpoint는 CoreDNS hosts 정적 항목으로 제공한다. 내부 DNS가 준비되면 같은 이름을 유지한 채 이전하고 CoreDNS 정적 항목을 제거한다. CoreDNS의 기본 책임은 Cluster 내부 Service 이름 해석이며, 이 정적 항목은 MVP의 외부 Endpoint 이름 해석을 위한 제한적 보완이다.

<a id="section-5-3"></a>

### 5.3 Static Route

| **대상** | **필수 Route** |
| --- | --- |
| VRouter-01 | 192.168.52.0/24 via 10.1.93.73 / 192.168.53.0/24 via .75 / 192.168.54.0/24 via .77 |
| VRouter-02 | 192.168.51.0/24 via 10.1.93.71 / 192.168.53.0/24 via .75 / 192.168.54.0/24 via .77 |
| VRouter-03 | 192.168.51.0/24 via 10.1.93.71 / 192.168.52.0/24 via .73 / 192.168.54.0/24 via .77 |
| VRouter-04 | 192.168.51.0/24 via 10.1.93.71 / 192.168.52.0/24 via .73 / 192.168.53.0/24 via .75 |
| LB-01/02 | 192.168.51~54.0/24를 각 VRouter 외부 IP .71/.73/.75/.77로 전달 |

Kubernetes 설치 전에 CP·Worker Node IP 간 양방향 L3 도달성, Return Path, 이름 해석, NTP와 필수 Port를 검증한다. Pod에서 Common VIP :3306으로 나간 DB 연결이 LB를 거쳐 MaxScale로 되돌아오는 경로도 별도 확인하여 비대칭 Routing이나 Hairpin 경로 문제를 방지한다. 외부 loadgen은 Common VIP :443을 통해서만 서비스 부하를 발생시키며, 부하 발생기 자체 CPU·Network가 병목이 아닌지 함께 기록한다.

<a id="section-6"></a>

## 6. Common VIP·HAProxy·Keepalived

| **주소·Port** | **접근 주체** | **HAProxy Backend** | **역할** |
| --- | --- | --- | --- |
| 10.1.93.90:80 | 사용자 Client | Worker-01/02:30080 | HTTP 진입 후 HTTPS Redirect |
| 10.1.93.90:443 | 사용자 Client | Worker-01/02:30443 | HTTPS·WebSocket 서비스 진입 |
| 10.1.93.90:6443 | 관리자·Node·Automation | CP-01/02/03:6443 | kubeadm controlPlaneEndpoint |
| 10.1.93.90:3306 | Backend·제한된 관리 경로 | MaxScale-01/02 Listener | DB Access Endpoint |

Keepalived는 VMnet0 Bridged를 통해 동일한 외부 L2에 연결된 LB-01과 LB-02 사이에서 Common VIP 소유권을 전환한다. HAProxy는 포트별 TCP Listener와 Backend Health Check를 담당한다. 적용 전 교육장 Network에서 Unicast VRRP, Gratuitous ARP, VIP 이동과 추가 MAC 허용 여부를 확인한다. 지원되면 원 목표 LB Pair를 적용하고, 지원되지 않으면 07 MVP에서 단일 LB가 Common VIP를 고정 보유하며 VIP 자동 전환은 미구현·제외 범위로 기록한다.

NGINX Gateway Fabric Data Plane은 NodePort Service로 노출하고 HTTP 30080/TCP, HTTPS 30443/TCP를 사용한다. externalTrafficPolicy는 초기 Cluster로 설정해 어느 Worker로 들어온 요청도 Ready Endpoint로 전달되게 하며, 원본 Client IP 보존보다 Worker·Gateway 장애 시 진입 유지와 구현 단순성을 우선한다.

Health는 계층별로 분리한다. HAProxy는 Gateway NodePort와 MaxScale Listener의 TCP 가용성을 확인하고, Kubernetes Readiness는 Gateway Pod Endpoint를 관리하며, MaxScale Monitor는 MariaDB topology를 판단한다. 최종 서비스와 DB 정상성은 HTTPS/WSS 요청 및 Common VIP :3306 실제 Transaction으로 검증한다.

> **공통 Failure Domain** Common VIP 하나와 LB Pair를 서비스·Kubernetes API·DB Access가 공유하므로 LB 계층은 세 경로의 공통 Failure Domain이다. 주소는 공유하지만 Listener, Backend Pool, Health Check와 Firewall 정책은 포트별로 분리한다. LB 한 대 장애 시 VIP가 생존 LB로 이동해야 하며, LB Pair 전체 장애는 세 신규 진입 경로에 동시에 영향을 준다.

<a id="section-7"></a>

## 7. TLS·Firewall·NetworkPolicy

<a id="section-7-1"></a>

### 7.1 TLS와 이름 해석

Common VIP의 HAProxy는 :80/:443에서 L4 TCP 전달을 수행하고, HTTPS/WSS TLS termination은 NGINX Gateway Fabric에서 담당한다. 내부 CA는 Gateway와 Harbor 등 서버 인증서를 서명하는 공통 신뢰 기준으로 사용한다. service.stone.test, k8s-api.stone.test, db.stone.test, harbor.stone.test를 DNS 또는 Ansible 관리 /etc/hosts와 인증서 SAN에서 일치시킨다. 인증서 수명·갱신과 실제 DNS 전환은 06에서 구체화한다.

<a id="section-7-2"></a>

### 7.2 주요 L3/L4 허용 경계

| **Source** | **Destination** | **Port/Protocol** | **목적** |
| --- | --- | --- | --- |
| 관리자·Ansible | SSH 준비된 CentOS Guest VM·Node | 22/TCP | SSH, Guest OS Baseline, 인프라 자동화 |
| Client | Common VIP | 80, 443/TCP | HTTP Redirect, HTTPS/WSS |
| 관리자·CP·Worker·Automation | Common VIP | 6443/TCP | Kubernetes API |
| Backend Pod·제한 관리 경로 | Common VIP | 3306/TCP | MaxScale DB Access; 사용자 Client에는 미공개 |
| LB-01/02 | CP-01~03 | 6443/TCP | Kubernetes API Backend |
| LB-01/02 | Worker-01/02 | 30080, 30443/TCP | Gateway NodePort Data Plane |
| LB-01/02 | MaxScale-01/02 | MaxScale Listener/TCP | DB Backend Pool |
| CP-01~03 | CP-01~03 | 2379-2380/TCP | stacked etcd |
| Control Plane·Metrics Server | CP/Worker kubelet | 10250/TCP | Node 상태와 Resource Metric |
| K8s Node | K8s Node | 4789/UDP | Calico VXLAN 사용 시 |
| MaxScale-01/02 | MariaDB-01/02 | 3306/TCP | Query Routing·Monitor |
| MariaDB Replica | MariaDB Primary | 3306/TCP | Replication |
| Worker | NFS | 2049/TCP | NFSv4 PVC |
| Jenkins Agent | Harbor·Source Repo·Deploy Repo | 443/TCP | Image Push와 Repository 접근 |
| Worker | Harbor | 443/TCP | Image Pull |
| Prometheus | Exporter·Application | Scrape 대상 Port/TCP | Metric Scrape; 상세 Port는 구현 시 최소 허용 |
| Prometheus | Alertmanager | Alertmanager Service Port/TCP | 평가된 Alert 전달 |
| Grafana | Prometheus·Loki | 내부 Service Port/TCP | Metric·Log 조회 |
| Grafana Alloy | Loki | Loki 수집 Endpoint/TCP | Log 전송 |
| Jenkins Controller | Kubernetes API | 내부 Kubernetes API/TCP | 동적 Agent Pod 생성·관리 |
| Alertmanager | E-mail SMTP / 선택적 Discord 연계 | SMTP Provider Port 또는 443/TCP | E-mail 우선 통보; Discord는 Webhook 연계 채택 시만 |
| 전체 Linux VM·Node | DNS·NTP | 53/TCP·UDP, 123/UDP | 이름 해석과 시간 동기화 |
| 외부 관리·부하 Server | Common VIP | 443/TCP | k6 HTTPS/WSS 부하 발생 |
| CentOS Guest VM·외부 Linux VM Alloy | Worker-01/02 Loki 수집 Endpoint | 31000/TCP | 허용된 관리 대역의 외부 Linux Log 전송 |
| Ansible·Linux VM·K8s Node | 승인된 Package·Image Repository | 443/TCP | 설치 자산·Container Image 다운로드; 06에서 Repository·Version 고정 |
| MariaDB·Control Plane·Ansible 검증 경로 | 외부 관리·부하 Server | 22/TCP | MariaDB Backup·etcd Snapshot·Raw Evidence를 SSH/SFTP 계열로 전송 |

<a id="section-7-3"></a>

### 7.3 Pod 통신과 Secret

Calico NetworkPolicy는 Frontend→Backend, Backend→Redis, Backend→Common VIP :3306, GAME→Redis Streams, ANALYSIS→Redis, Prometheus→Exporter·Application, Prometheus→Alertmanager, Grafana→Prometheus·Loki, in-cluster Alloy→Loki, 허용된 외부 Linux 대역→Loki 수집 Endpoint, Jenkins Controller→Kubernetes API, Jenkins Agent→Harbor·Git, Argo CD→Deployment Repository·Kubernetes API, Alertmanager→E-mail SMTP 또는 선택적 Discord 연계 등 필요한 경로만 허용한다. Common VIP의 :3306은 사용자 서비스 포트와 분리하여 Backend Namespace와 제한된 관리 주체만 접근하도록 한다.

MariaDB 자격정보, Harbor Registry 인증정보, Git 접근정보, Jenkins Credential과 내부 CA·TLS Private Key는 일반 ConfigMap이나 문서에 평문으로 포함하지 않는다. Kubernetes Secret과 Ansible Vault 등 구현 수단의 책임을 분리하고 최소 RBAC를 적용한다. Kubernetes Secret의 실제 암호화 저장 여부는 별도 설정이므로 etcd Backup·접근 권한과 함께 검증한다.

<a id="section-8"></a>

## 8. Kubernetes Node·Pod·Service Network

| **영역** | **물리 설계** | **구현·검증 항목** |
| --- | --- | --- |
| Node Network | CP·Worker가 192.168.51~53.0/24에 분산 | Node 간 양방향 L3, Return Path, Kubernetes 필수 Port |
| Pod Network | 10.244.0.0/16 · Calico VXLAN CrossSubnet | IPPool, VXLAN 4789/UDP, MTU, natOutgoing, Pod-to-Pod |
| Service Network | 10.96.0.0/12 | CoreDNS, ClusterIP, Gateway·Application Service |
| Resource Metrics | Metrics Server in-cluster | metrics.k8s.io와 CPU·Memory 기반 HPA; Prometheus와 역할 분리 |
| Custom Metrics | 현재 기본 경로 아님 | 필요 시 Prometheus Adapter 등 별도 API 경로를 비교 후 추가 |
| Workload 분산 | 2 Worker에 Replica·상태·운영 Workload 분산 | anti-affinity/topology spread, PDB, Priority, requests/limits |

Prometheus는 단기 Metric 저장과 Alert Rule 평가에 사용하고 Metrics Server는 HPA와 kubectl top에 필요한 Resource Metrics API를 제공한다. 두 구성요소를 같은 역할로 표현하지 않는다. Pod CIDR은 10.244.0.0/16, Service CIDR은 10.96.0.0/12로 고정하며 외부망 10.1.93.0/24와 내부망 192.168.51~54.0/24의 중복 여부를 설치 전 다시 검사한다.

<a id="section-9"></a>

## 9. Data·Runtime State 물리 경계

| **계층** | **배치** | **책임·복구 경계** |
| --- | --- | --- |
| MariaDB-01 Primary | Server-02 / 192.168.52.40 | Member·Move·GameResult·Rating 등 영속 원본과 Transaction/Constraint |
| MariaDB-02 Replica | Server-01 / 192.168.51.40 | Primary와 다른 Failure Domain의 Replication·승격 후보 |
| MaxScale-01 | Server-03 / 192.168.53.40 | DB Routing·Monitor·Health; HAProxy Backend |
| MaxScale-02 | Server-04 / 192.168.54.40 | MaxScale 자체 단일 장애 수용 |
| Redis | Kubernetes Stateful Workload + AOF/PVC | Session, Room/Game/Turn/Vote, Pub/Sub, Analysis Stream과 최신 분석 상태 |
| ANALYSIS | Kubernetes 독립 Consumer Workload | 공식 Move 이후 Redis Streams Job 처리; 게임 권위 경로와 분리 |

MariaDB Replication과 MaxScale은 Backup을 대체하지 않는다. Primary 장애 시에는 구 Primary의 정지·격리를 확인한 뒤 운영자 승인으로 Replica를 승격하며, 2노드 구성에서 완전 자동 승격을 기본값으로 사용하지 않는다. MaxScale Monitor는 topology를 관측하고 승격 후 Routing을 갱신한다. MaxScale 2대가 서로 다른 변경을 수행하지 않도록 cooperative monitoring 또는 동등한 단일 권한 원칙을 적용하고, split-brain·Transaction 무결성·구 Primary 재편입을 함께 검증한다.

Redis는 영속 비즈니스 원본이 아니지만 인증 세션, 활성 게임 상태와 Analysis Job을 공유하므로 장애 영향이 넓다. Redis Pod 재기동, NFS 접근 실패, AOF 손상과 Streams Pending 복구는 서로 다른 장애로 구분한다. Sentinel을 사용하지 않으므로 Redis 자체를 완전한 HA로 표현하지 않는다.

<a id="section-10"></a>

## 10. External NFS와 Persistent Storage

<a id="section-10-1"></a>

### 10.1 선택 구조와 대안

| **후보** | **용도** | **판정** | **이유** |
| --- | --- | --- | --- |
| Linux VM 기반 External NFS | Shared File Storage·PVC | 선택 | 구축·자동화·장애 복구를 직접 검증할 수 있고 현재 규모에 적합 |
| NAS 기반 NFS | Storage System이 NFS 제공 | 대안 | NFS 제공 계층의 운영 편의와 안정성은 높일 수 있으나 현재 실제 구성요소가 아님 |
| Longhorn 등 분산 Storage | Node 기반 분산 Block Storage | 미채택 | Worker 2대와 1차 범위에서 운영 복잡도·자원 비용이 큼 |
| Worker Local Storage | Prometheus·Loki 단기 저장 | 선택 | NFS 제약을 피하고 4주 PoC를 단순화한다. Node·Pod 장애 시 과거 관측 데이터 손실 가능성을 수용하며 구체 Volume 유형은 06에서 확정 |
| MinIO | Object·Backup Storage | 확장 후보 | NFS 대체가 아니라 Object/Backup 문제를 위한 별도 계층 |

<a id="section-10-2"></a>

### 10.2 Storage 배치와 Failure Domain

| **소비자** | **저장 방식·대상** | **장애 영향·판정** |
| --- | --- | --- |
| Redis | NFS PVC · AOF와 Runtime State 복구 데이터 | NFS 장애 시 세션·게임 최신 상태·Analysis Stream 복구 저하 |
| Jenkins Controller | NFS PVC · Job·Plugin·설정 | NFS 장애 시 Controller 상태 재연결 실패와 신규 CI 중단 |
| MariaDB Backup Staging | NFS Share · Restore 실험용 임시 사본 | NFS와 함께 손실될 수 있어 독립 DR·off-site Backup으로 주장하지 않음 |
| Prometheus | Worker Local Filesystem · 7일 또는 20GiB 시작값 | NFS 무관; Pod·Node 장애 시 일부 과거 Metric 손실 가능 |
| Loki | Single Binary + Worker Local Filesystem · 72시간 또는 10GiB 시작값 | NFS 무관; Pod·Node 장애 시 일부 과거 Log 손실 가능 |
| Grafana | Dashboard·Datasource GitOps · 기본 PVC 없음 | Git Desired State에서 재생성; UI 전용 변경은 공식 상태로 인정하지 않음 |
| Alertmanager | Route·Receiver GitOps · 기본 PVC 없음 | 재기동 시 임시 Silence 상태 손실 가능; 사용자 서비스는 직접 영향 없음 |
| Protected Backup·Evidence | 외부 관리·부하 Server 별도 Disk · MariaDB Backup·etcd Snapshot·Raw Evidence | Physical Server 4대 및 NFS와 다른 복구 단위. 단일 외부 Server이므로 off-site·Site DR 또는 HA Storage로 주장하지 않음 |

Prometheus·Loki의 Retention과 용량은 실험 시작값이며 실제 유입량을 측정해 06에서 동결한다. 각 장애·부하 실험이 끝날 때 핵심 Metric·Log·Timestamp를 CSV·JSON·실행 로그 등 Raw Evidence로 즉시 내보내 관측 Pod나 Worker 장애가 결과 증거를 함께 지우지 않도록 한다.

NFS Subdir External Provisioner는 기존 NFS Share를 Redis·Jenkins 등 NFS 소비자의 StorageClass와 PVC에 연결하는 Kubernetes 내부 Deployment다. Provisioner 장애는 주로 신규 PV/PVC 생성에 영향을 주며, NFS Server 장애는 기존 Volume의 실제 I/O를 중단시킨다. Prometheus·Loki Local Storage 장애와는 별개로 구분하여 검증한다.

> **Backup 경계** Runtime PVC와 Backup을 동일 NFS에만 보관하면 NFS 장애에 함께 노출된다. NFS Share의 MariaDB Backup Staging은 Restore 실험용 임시 사본으로 유지하고, MariaDB Backup·etcd Snapshot·Raw Evidence는 외부 관리·부하 Server의 별도 Disk에도 전송한다. 이 Disk는 Physical Server 4대와 NFS 손실에서 분리된 복구 단위이지만 단일 외부 Server이므로 off-site·Site DR 또는 HA Backup으로 표현하지 않는다. Backup 전송과 부하 시험은 동시에 실행하지 않는다.

<a id="section-11"></a>

## 11. CI/CD·GitOps·Image Delivery

| **구성요소** | **배치** | **책임** | **장애 영향** |
| --- | --- | --- | --- |
| Jenkins Controller | Kubernetes 내부 + PVC | Pipeline 제어, Build/Test 요청 | 재스케줄·PVC 재연결 전 신규 CI 중단; Runtime Game은 직접 영향 없음 |
| Jenkins Agent Pod | Kubernetes 내부 일회성 Pod | Build/Test·Image 생성 | Worker 자원과 Build 동시성에 영향; 장애 시 재실행 |
| Harbor | Server-03 / 192.168.53.61 | Private Image Registry | 실행 Pod는 즉시 유지되나 캐시되지 않은 재Pull·재스케줄·배포·확장에 영향 |
| Argo CD | Kubernetes 내부 | Deployment Repository의 Desired State 동기화 | 실행 서비스는 유지되나 신규 Sync와 Drift 감지 저하 |
| Source Repository | 외부 Repository | Application Source와 Jenkinsfile | Jenkins 신규 Build 입력 중단 |
| Deployment Repository | 외부 Repository | Manifest·Desired State | Argo CD Sync 및 변경 배포 중단 |

Jenkins는 Kubernetes에 직접 최종 배포하지 않는다. Build/Test 후 Image를 Harbor에 Push하고 Deployment Repository의 Desired State를 변경하며, Argo CD가 Cluster에 동기화한다. Jenkins Controller 상태는 NFS PVC로 보존하지만 Build Workspace는 일시 Volume을 기본으로 하여 NFS I/O를 줄인다. Host의 Docker Socket 직접 Mount는 Worker 권한 경계가 과도하므로 기본안에서 제외하고, Rootless BuildKit 등 Socket을 노출하지 않는 Build 방식과 호환성은 06에서 최종 선택한다.

<a id="section-12"></a>

## 12. Observability·Alerting·Automation

<a id="section-12-1"></a>

### 12.1 관측성과 알림 역할

| **구성요소** | **배치** | **책임** |
| --- | --- | --- |
| Prometheus | Kubernetes 내부 + Worker Local | Metric 단기 저장 및 Alert Rule 평가 |
| Alertmanager | Kubernetes 내부 · 기본 PVC 없음 | Alert 그룹화·중복 억제·상호 억제·Silence·라우팅 |
| Grafana | Kubernetes 내부 · GitOps 복원 | Metric·Log 조회와 Dashboard·Datasource 시각화 |
| Grafana Alloy | Kubernetes DaemonSet + 외부 Linux Agent | Pod·Node 및 CentOS Guest·외부 Linux VM의 Journal·File Log 수집 |
| Loki | Kubernetes Single Binary + Worker Local | Log 단기 저장·조회 |
| Metrics Server | Kubernetes 내부 | Resource Metrics API와 CPU·Memory 기반 HPA |
| kube-state-metrics | Kubernetes 내부 | Kubernetes Object 상태를 Prometheus Metric으로 노출 |
| node_exporter | 각 CentOS Guest VM·Kubernetes Node | Host CPU·Memory·Disk·Network Metric |
| Application·DB Exporter | 각 책임 경계 | 서비스·MariaDB·MaxScale 등 검증 Metric 노출 |
| Evidence Export | 외부 관리·부하 Server 별도 Disk | 실험별 Metric·Log·Timestamp·명령 결과를 CSV·JSON·텍스트로 보존 |

Kubernetes 내부 Alloy는 Pod와 Node Log를 수집하고, MariaDB·MaxScale·LB·NFS·VRouter·Harbor·Ansible Controller 등 CentOS Guest VM과 외부 지원 Linux VM에는 systemd 서비스 형태의 Alloy Agent를 설치한다. 외부 Agent는 관리 대역에서만 허용되는 Worker Loki 수집 Endpoint로 전송하며, Journal 읽기 권한은 필요한 그룹·ACL로 최소화한다. Windows Host의 CPU·Memory·Disk와 VMware 자원 상태는 Task Manager·Performance Monitor·VMware 화면으로 확인하고 Linux Alloy·node_exporter 대상과 구분한다. 수집 중단은 서비스 장애가 아닌 관측성 열화로 판정한다.

| **상태** | **기록** | **기본 알림** | **운영 의미** |
| --- | --- | --- | --- |
| 자동 복구된 일시 장애 | Metrics·Logs·Events | 없음 또는 필요 시 기록성 경고 | 사람의 즉시 개입 없이 추세만 확인 |
| 반복 장애·가용성 저하 | Metrics·Logs | Warning | 이중화 열화·재시작 반복·실패율 증가 확인 |
| 자동 복구 실패·서비스 상실 | Metrics·Logs | Critical | 운영자 개입 필요 |
| 데이터 손실 위험·SPOF 장애 | Metrics·Logs | Critical | 복구 절차 수행과 무결성 확인 |

Alertmanager는 자동 복구 엔진이 아니며 사용자 서비스의 권위 경로에도 포함되지 않는다. Alertmanager 장애 시 게임 서비스는 계속될 수 있지만 능동 통보, Silence와 Alert 상태 관리가 저하된다. 1차 기본 통보 채널은 E-mail로 하고, Discord는 Generic Webhook을 직접 호환할 수 있는 연계 구성이 확인될 때만 선택적으로 추가한다. SMTP·Webhook Secret과 egress는 06에서 구체화한다.

<a id="section-12-2"></a>

### 12.2 Ansible Controller

Ansible Controller는 Server-04의 192.168.54.70에 배치하고 SSH 준비가 끝난 CentOS Guest VM과 Kubernetes Node를 관리 대상으로 둔다. Windows Host·Virtual Network Editor·VM Hardware 생성·CentOS 설치·초기 SSH 준비는 수동 공통 기반이며, 이후 Guest OS Baseline, VM 내부 Network, VRouter·Route·Firewall, kubeadm Bootstrap, External Infra, 내부 CA·인증서 배포와 장애 재구축을 자동화한다. Kubernetes Workload의 지속적 Desired State 배포는 Argo CD와 구분한다. Playbook·Inventory는 외부 Git Repository에도 보존하고 Secret 원문을 Inventory나 일반 로그에 남기지 않는다. MariaDB Backup·etcd Snapshot·Raw Evidence는 외부 지원 Server의 별도 Disk에 전송한다. Controller 또는 Server-04 장애 시 외부 지원 Server의 긴급 SSH 경로와 해당 복구 자료를 복구 출발점으로 사용한다.

<a id="section-13"></a>

## 13. HA·SPOF·Failure Domain

| **구성요소** | **가용성 구조** | **단일 장애 시 기대** | **판정** |
| --- | --- | --- | --- |
| Control Plane | CP 3 + stacked etcd | CP 1대 장애 시 남은 2대 quorum | HA |
| Worker | Worker 2 + Replica·재스케줄 | 남은 Worker의 Request 예산과 배치 제약 충족 시 핵심 Runtime 유지 | 조건부 HA 검증 |
| Common VIP/LB | HAProxy + Keepalived 2대(네트워크 지원 시) | VRRP·Gratuitous ARP 검증 통과 시 LB 1대 장애 후 VIP 이동 | 조건부 HA |
| Gateway Data Plane | Replica + Kubernetes Service | 다른 Worker로 진입 경로 재수용 | HA 검증 |
| MariaDB | Primary + Replica | 승격·재연결·무결성 검증 필요 | Replication 기반 복구 |
| MaxScale | 2대 분산 | HAProxy가 장애 인스턴스를 제외 | HA |
| Redis | Pod 재기동 + AOF/PVC | Pod 장애는 복구 가능, NFS·AOF 장애는 별도 복구 | 부분 HA |
| Jenkins | in-cluster Controller + 동적 Agent | Worker·Scheduler·NFS/PVC에 의존 | 재스케줄 기반 복구 |
| Alertmanager | in-cluster · 기본 PVC 없음 | 사용자 서비스는 유지, 능동 통보·임시 Silence 공백 가능 | 운영 기능 열화 |
| NFS | 단일 VM | Redis·Jenkins Shared PVC와 Backup Staging 영향 | 의도적 SPOF |
| Harbor | 단일 VM | 신규 Push/Pull 및 캐시 없는 재스케줄·배포 영향 | 허용 SPOF |
| Ansible Controller | 단일 VM | 자동화 실행·재구축 지연, Runtime 직접 영향 없음 | 허용 SPOF |
| VRouter | Server당 1대 | 해당 Private Subnet 격리 | Server Network Failure Domain |
| Prometheus·Loki Store | Worker Local · 단기 보존 | Pod·Worker 장애 시 일부 과거 관측 데이터 손실 가능; Raw Evidence 별도 보존 | 허용 비HA |

Pod anti-affinity 또는 topology spread를 사용해 Backend, Gateway와 가능한 운영 Replica를 두 Worker에 분산한다. PodDisruptionBudget은 계획된 중단에서 동시 손실을 제한하지만 Physical Server 장애를 막는 장치는 아니므로, 실제 장애 수용성은 남은 Worker의 자원과 Image·PVC 접근 가능 여부로 판정한다.

<a id="section-14"></a>

## 14. Physical Server 장애 영향

| **장애 Server** | **직접 손실 VM** | **동적 Workload 영향** | **검증해야 할 결과** |
| --- | --- | --- | --- |
| Server-01 | VRouter-01, CP-01, Worker-01, MariaDB Replica | Worker-01에 실제 배치된 Runtime·CI/GitOps·Observability·Cluster Add-on Pod | CP quorum, 서비스 진입·재연결, Pod 재스케줄, DB Primary 단독 처리, Harbor/NFS 접근과 Worker-01 Local 관측 데이터 손실 범위 |
| Server-02 | VRouter-02, CP-02, Worker-02, MariaDB Primary | Worker-02에 실제 배치된 Runtime·CI/GitOps·Observability·Cluster Add-on Pod와 DB Write 동시 영향 | CP quorum, 남은 Worker 수용, 운영자 승인 Replica 승격, MaxScale 경로 복구, Transaction 무결성과 Worker-02 Local 관측 데이터 손실 범위 |
| Server-03 | VRouter-03, CP-03, LB-01, MaxScale-01, Harbor | Worker는 유지되지만 Image Registry 접근 저하 | VIP·API·DB Endpoint 우회와 함께 캐시 없는 Image Pull·재배포·확장 영향 분리 |
| Server-04 | VRouter-04, LB-02, MaxScale-02, NFS, Ansible Controller | NFS PVC 기반 Redis·Jenkins와 Backup Staging 영향 | VIP·DB Endpoint 생존과 별개로 NFS I/O, 자동화 복구·MTTR, 외부 관리 Server의 긴급 SSH 경로와 Backup·Raw Evidence 보존 상태를 검증 |

Physical Server 장애표의 Kubernetes Workload는 고정 배치 VM 목록이 아니라 장애 시 해당 Worker에 실제 스케줄되어 있던 Pod를 의미한다. 따라서 장애 실험마다 사전 Pod 배치와 Prometheus·Loki Local Storage 위치 Snapshot, 사후 재스케줄 결과와 Raw Evidence Export 상태를 함께 기록한다.

<a id="section-15"></a>

## 15. 검증 목표·확장 조건

<a id="section-15-1"></a>

### 15.1 물리 설계 검증

| **검증 대상** | **장애·부하 조건** | **정상화 기준** | **측정 항목** |
| --- | --- | --- | --- |
| Worker | Worker 1대 중단 | 핵심 Pod가 남은 Worker에 수용 | 실패 요청, 재연결 성공률, Ready·복구시간, Eviction |
| Control Plane | CP 1대 중단 | API와 etcd quorum 유지 | API 성공률, 가용 공백 |
| Common VIP/LB | 사전 L2 검증 / 지원 시 Active LB 중단 | VRRP·ARP 지원 확인 후 VIP가 Standby로 이동; 미지원 시 07 단일 LB 경계 기록 | 사전 검증 결과, VIP 전환시간, 포트별 실패 구간 |
| Gateway | Gateway Pod·Worker 장애 | HTTPS/WSS 신규 진입 복구 | Gateway Ready, WebSocket 재연결 |
| MariaDB Primary | Primary 중단 | Replica 승격과 Write 재개 | RTO, 실패 Transaction, 중복·유실, 무결성 |
| MaxScale | MaxScale 1대 중단 | 생존 MaxScale로 DB 연결 | Health 반영과 재연결 시간 |
| Redis | Pod·AOF·NFS 장애 분리 | Current State와 Stream 복구 | MTTR, Session·Game State·Job 중복·유실 |
| ANALYSIS | 지연·중단·중복 Job | 게임 권위 경로 무영향 | Move·Turn 무영향, stale 오반영 0건 |
| Jenkins | Controller·Agent·Worker 장애 | Controller 재스케줄·PVC 재연결 | CI 복구시간, Build 재실행, Runtime 무영향 |
| Harbor | Registry 중단 후 Pod 재생성 | 캐시·복구 후 Image Pull | 재배포·재스케줄 영향 |
| NFS·Provisioner | Provisioner 중단 / NFS 중단 | PVC 생성과 기존 I/O 영향 구분 | MTTR, 신규 PVC, Redis·Jenkins·Backup Staging 영향 |
| Alerting | Warning·Critical·Resolved 조건 주입 | E-mail Grouping·Inhibition·Routing·Resolved 처리 | 전달 지연, 중복 통보, 알림 공백 |
| Prometheus·Loki Store | 관측 Pod 재기동·Worker 장애 | 서비스 권위 경로 유지, 단기 Local 데이터 손실 범위와 Evidence 보존 확인 | Retention, 유실 구간, Evidence Export 성공 |
| 외부 로그 수집 | 외부 Alloy·Loki 수집 Endpoint 중단 | 서비스는 유지되고 관측성 열화로 분리 | 수집 지연·누락 구간·복구시간 |
| 외부 부하 발생 | k6 단계 부하와 AI 활성 조건 | 부하 발생기 병목 없이 Vote·HTTPS/WSS 지표 수집 | Generator CPU·Network, RPS, p95·p99, Error rate |
| Backup·Evidence | MariaDB 원본 또는 NFS·Server-04 손실 | 외부 관리·부하 Server의 Backup으로 Restore하고 Raw Evidence를 유지 | Backup 성공, 표본 데이터 무결성, RTO·RPO, Evidence Checksum |

<a id="section-15-2"></a>

### 15.2 서비스 실행 Server 추가 확장 조건

외부 지원 Server는 서비스 실행 Failure Domain에 포함하지 않는다. 현재 서비스 실행 환경은 Server-01~04로 고정하며, 추가 서비스 실행 Server는 아래 조건에 해당하는 실측 병목이 확인될 때만 검토한다.

- Server-01/02에서 Worker와 MariaDB의 CPU·Memory·Disk I/O 경합이 Application·DB 성능 측정을 지속적으로 왜곡할 때

- Worker 1대 장애 시 신규 Build Agent와 비필수 배치 작업을 중지해도 남은 Worker가 Gateway·Frontend·Backend·Redis·CoreDNS·Jenkins Controller·Argo CD 및 필수 관측 최소선을 수용하지 못할 때

- Jenkins·Redis와 Backup Staging의 NFS I/O가 병목이 되어 장애·성능 검증을 방해할 때

- Prometheus·Loki의 Worker Local Disk 사용량이 Retention 시작값을 초과하거나 장기 보존 요구가 실제로 생길 때는 먼저 Retention을 조정하고, 이후에만 Object Storage 확장을 검토할 것

- Harbor Image·Scanner I/O가 Server-03의 Control Plane·LB·MaxScale 검증을 방해할 때

- 하나의 Physical Server 장애가 너무 많은 계층을 동시에 잃게 해 개별 장애의 원인과 결과를 분리하기 어려울 때

- Server-05 적용 후에도 동일 병목 또는 Failure Domain 문제가 남는 경우에만 Server-06을 검토할 것

<a id="section-16"></a>

## 16. 자동화·테스트 설계 이관

| **물리 대상** | **자동화·테스트에서 구체화할 내용** |
| --- | --- |
| VRouter·Static Route | IP forwarding, Route, Firewall을 Role·Template로 구성하고 재실행 멱등성 검증 |
| CP 3·Worker 2 | kubeadm bootstrap/join, release 버전 고정, Health Check, Node 장애·복구 |
| Calico | IPPool, VXLAN CrossSubnet, MTU, natOutgoing, NetworkPolicy |
| Common VIP·LB | 교육장 L2·VRRP·Gratuitous ARP 사전 점검, 지원 시 VRRP Interface·Priority와 HAProxy 포트별 Listener·Backend·Health Check, 미지원 시 07 단일 LB 이관 |
| Gateway | Gateway API·NGINX Gateway Fabric, Replica, NodePort 30080/30443, externalTrafficPolicy Cluster, TLS Secret, HTTPS/WSS 장애 |
| MariaDB·MaxScale | Replication, cooperative monitoring, 구 Primary Fencing 확인 후 운영자 승인 승격·재연결·재편입, Listener/Topology/Transaction Health, Backup/Restore와 무결성 |
| Redis·ANALYSIS | AOF/PVC, Pub/Sub·Streams Consumer Group, Pending/ACK·Retry, stale·중복 격리 |
| NFS·Provisioner | Export, NFSv4 Mount, StorageClass/PV/PVC, Provisioner와 NFS 장애 분리 |
| Jenkins·Harbor·Argo CD | Controller PVC, 동적 Agent Pod, Build 동시성, Host Docker Socket을 노출하지 않는 Image Build 방식, Image Push/Pull, GitOps Sync |
| Metrics·Logs·Alert | Metrics Server, Exporter, Prometheus·Loki Local Storage·Retention·Evidence Export, 외부 Linux Alloy Agent와 31000/TCP 수집 경계, Alertmanager E-mail 우선 Severity·Grouping·Inhibition·Routing·firing/resolved 검증 |
| Secret·TLS | Ansible Vault·Kubernetes Secret·RBAC, 내부 CA, SAN, 인증서 배포·갱신, etcd 보호 |
| DR | MariaDB Backup, Redis AOF, etcd Snapshot, PVC·NFS 복구 단위, 외부 관리·부하 Server 별도 Disk 전송, RTO/RPO·Retention |
| Before/After | 수동 대비 구축시간, 직접 개입 단계, 실패 Host/Task, 재실행 Changed/OK와 복구시간 |
| Windows Host·VMware / CentOS Guest OS | Windows Host IP·C 드라이브 500GB 확보·VMnet0 Bridged·Server별 Host-only VMnet·VM Hardware·CentOS 설치·초기 SSH는 수동 공통 기반으로 확인하고, 이후 CentOS Stream 9 Repository·SSH Key·become·Guest OS Baseline 자동화 범위와 멱등성을 검증 |
| Name·Load Generator | stone.test /etc/hosts·CoreDNS hosts·DNS·SAN, loadgen 10.1.93.91 중복 점검, 외부 k6 Profile과 Generator 병목 판정, Backup 작업과 부하 시험의 실행 시간 분리 |
| Controller Recovery | Playbook·Inventory 외부 Git 보존, Vault·CA 복구 경계, 외부 관리·부하 Server의 Backup·Raw Evidence 보존, Server-04 장애 시 긴급 SSH 절차 |

<a id="section-17"></a>

## 17. 기술적 사실 확인 근거

프로젝트의 결정은 팀 환경과 선행 설계를 우선하며, 아래 공식 문서는 2026-08-24 기준의 기술적 사실과 자원 시작값을 확인하는 근거로 사용한다. 공식 최소값은 설치 가능성 확인에 사용하고 공식 권장값이 제공되면 권장값을 우선하되, 최종 Size에는 실제 Workload와 장애 수용 예산을 함께 반영한다. Release Version과 호환성은 06에서 다시 확인해 고정한다.

| **주제** | **공식 근거** | **적용 사실** |
| --- | --- | --- |
| kubeadm HA | Kubernetes Docs · Creating Highly Available Clusters with kubeadm | stacked Control Plane은 etcd와 Control Plane을 함께 두며 안정된 Load Balancer Endpoint가 필요 |
| Kubernetes Port | Kubernetes Docs · Ports and Protocols | API, etcd, kubelet 등 기본 통신 경계 확인 |
| HPA·Metrics | Kubernetes Docs · Horizontal Pod Autoscaling | metrics.k8s.io는 일반적으로 Metrics Server가 제공하며 Prometheus와 역할이 다름 |
| Calico | Calico Docs · Overlay networking / IPPool | VXLAN CrossSubnet은 Subnet 경계를 지나는 Traffic만 선택적으로 캡슐화 가능 |
| Gateway | Kubernetes Docs · Gateway API / NGINX Gateway Fabric Docs | Gateway·Route 책임과 NGINX Gateway Fabric Data Plane의 배포·Health 경계 확인 |
| Common VIP·L4 | Keepalived Docs · VRRP / HAProxy Docs · Configuration Manual | VRRP 기반 VIP 소유권 전환과 포트별 TCP Listener·Backend Health Check 책임 분리 |
| NFS Provisioner | Kubernetes SIG Storage · NFS Subdir External Provisioner | 기존 NFS Share에서 StorageClass·PVC 기반 Dynamic PV 제공 |
| Alertmanager | Prometheus Docs · Alertmanager | 그룹화·억제·Silence·라우팅 책임 |
| Jenkins Agent | Jenkins Kubernetes Plugin | Kubernetes Pod 기반 동적 Agent와 emptyDir·PVC 등 Workspace 선택 |
| MariaDB·MaxScale | MariaDB Docs · MaxScale / MariaDB Monitor | Application과 DB topology 사이의 Routing·Monitor·Failover 경계 |
| Harbor | Harbor Docs · Installation Prerequisites | 권장 시작값 4 CPU, 8GB Memory, 160GB Disk |
| Kubernetes Secret | Kubernetes Docs · Secrets / Good practices | Secret은 기본적으로 etcd에 암호화되지 않을 수 있어 RBAC·암호화·Backup 보호 필요 |
| CentOS Stream 9 | CentOS Project · CentOS Stream 9 / EOL | EOL 2027-05-31; 4주 프로젝트에 사용하되 장기 확장 전 재평가 |
| VMware Network | Broadcom Knowledge Base · Using the Virtual Network Editor in VMware Workstation (Article 339371) | Bridged·Host-only·NAT·Custom Network와 VM vNIC 추가·변경의 동작 경계 확인 |
| Gateway NodePort | NGINX Gateway Fabric Docs · Deploy Data Plane / API Reference | NodePort와 externalTrafficPolicy Cluster·Local 동작 경계 |
| Prometheus Storage | Prometheus Docs · Storage | NFS 미지원, Local Filesystem 권장 |
| Loki Storage | Grafana Loki Docs · Filesystem Object Store | Filesystem은 PoC에 단순하지만 Production 지원 대상이 아님 |
| Grafana Provisioning | Grafana Docs · Provision Grafana | Dashboard·Datasource를 Version Control 가능한 파일로 관리 |
| External Alloy | Grafana Alloy Docs · Monitor Linux / Journal | 외부 Linux에서 Agent를 실행하고 Journal·File 권한을 별도 부여 |
