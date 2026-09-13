<a id="project-proposal"></a>

# 코오롱베니트 선도기업 아카데미 프로젝트 기획안

|  |  |  |  |
| --- | --- | --- | --- |
| 교육과정명 | [코오롱베니트]하이브리드 클라우드 컨테이너 플랫폼 설계 및 구현 과정 9회차 | 교육일자 | 2026-04-13 ~ 2026-10-26 |
| 팀원 | 정태훈, 김상희, 이유빈, 최유준 | 팀명 | 石판 (석나가는 판단) |
| 프로젝트 교과 | 인프라 및 애플리케이션 자동화 프로젝트 |  |  |
| 프로젝트 부제 | 실시간 투표형 오목 서비스 컨테이너 플랫폼 자동화 구축 |  |  |

<a id="architecture"></a>

## 아키텍처

![01. 전체 논리 아키텍처](logical-architecture/01_SeokPan_전체 논리 아키텍처.png)

<a id="logical-architecture-key-design-points"></a>

### 핵심 설계 포인트

하나의 구조 안에서도 역할과 데이터 책임을 분리해, 장애가 발생했을 때 영향 범위와 복구 지점을 추적할 수 있도록 설계한다

- **Common Endpoint**  
  10.1.93.90을 유지하되 80/443은 서비스, 6443은 Kubernetes API, 3306은 DB Access로 구분한다.
- **Kubernetes 내부 역할 분리**  
  Gateway·Frontend·Backend·Redis와 Jenkins·Argo CD·Observability를 역할별로 분리한다.
- **Runtime State와 확정 데이터 분리**  
  Redis는 Room·Game·Turn·Vote 등 진행 중 공유 상태와 실시간 전달을 담당하고, MariaDB는Move·GameResult·Rating 등 재시작 이후에도 남아야 하는 기록을 저장한다.
- **자동화·배포 책임 분리**  
  Ansible은 인프라 구성, Jenkins는 Build/Test와 Image 생성, Harbor는 Image Registry, Argo CD는 Application Desired State 동기화를 담당한다.

<a id="physical-architecture"></a>

## 물리 아키텍처

![01. 전체 물리 아키텍처](physical-architecture/01_SeokPan_전체 물리 아키텍처.png)

<a id="physical-architecture-key-design-points"></a>

### 핵심 설계 포인트

실제 구현은16개 가상환경MVP를 기준으로 하고, 확장 모델18개 가상환경 구조는 동일한 역할·Endpoint·데이터 경계를 유지한 확장 설계로 둔다.

- **4대 Physical Server 분산**  
  Control Plane·Worker·DB·LB·Storage 등을 서로 다른 Server에 배치한다.
- **Network 경계**  
  각 Server의 Private Subnet은 VRouter Static Routing으로 연결하고, Pod 간 통신은 Calico VXLAN을 사용한다.
- **MVP 우선 구현**  
  16VM에서 Control Plane 3대·Worker 2대, MariaDB Primary/Replica, LB-01, MaxScale-01, VRouter 4대, Harbor, NFS, Ansible Controller를 유지한다.
- **확장 모델**  
  MVP 검증 후Go/No-Go 조건을 충족하면 기존Endpoint와 서비스 구조를 유지한 채lb-02·maxscale-02를 추가해LB·DB Proxy HA를 확장하고, 여력이 있을 경우 보존된 이벤트 계약을 기반으로ANALYSIS 런타임을 추가한다. 확장 후에는Failover와 관련Failure Domain을 추가 검증한다

<a id="project-purpose"></a>

## 프로젝트 목적

「石나가는 판단」은 실시간 집단 투표 오목 서비스를 실제 검증Workload로 사용하여, On-premise Kubernetes 환경에서 인프라 자동화, 서비스 연속성, 데이터 보호·Recovery, 배포 자동화, Observability를 구현하고 실제 측정값으로 검증하는 프로젝트다. 실제 수행은16VM MVP를 기준으로 하며, Must 기능과 핵심 검증을 완료한 뒤Go/No-Go 조건을 충족할 경우에만18VM 확장 모델 구조로 확장한다.

<a id="project-purpose-1"></a>

### 1. 표준화·재사용 가능한On-premise 인프라 자동화

Windows Host·VMware·가상환경생성은 수동 공통 기반으로 두고, CentOS Guest 이후의 공통 설정·Network·Kubernetes·External Infrastructure를 Ansible Role/Playbook과 Inventory·Variable 구조로 표준화한다.

<a id="project-purpose-2"></a>

### 2. 실시간 서비스 상태의 일관성과 장애 연속성

Room·Game·Turn·Board·Vote는 Backend의 비즈니스 로직이 최종 판단하고, Redis는 여러 Backend가 같은 Runtime State를 공유·복원할 수 있도록 지원한다.

<a id="project-purpose-3"></a>

### 3. Data Availability와 Recovery의 분리 검증

MariaDB Primary/Replica는 DB Availability를 위한 복제 기반 구조로 두고, MVP의 MaxScale-01은 Backend가 직접 DB topology에 의존하지 않도록 DB Access/Routing Endpoint를 제공한다. MaxScale 자체는 MVP에서 단일 구성으로 SPOF이며, Proxy HA는 확장 모델 확장 범위에서 검증한다. mariadb-backup·etcd Snapshot·Redis AOF와 Restore 절차는 Recovery 경로로 구분한다.

<a id="project-purpose-4"></a>

### 4. CI/CD·GitOps와 Observability 기반 운영

Jenkins는 Build/Test와 Image 생성, Harbor는 Image Registry, Argo CD는 Git에 선언된 배포 상태 동기화를 담당한다. Prometheus는 Metrics 수집·저장과 Alert Rule 평가, Grafana Alloy는 Log 수집, Loki는 Log 저장·조회, Alertmanager는 Alert 라우팅, Grafana는 Metric·Log·Alert 상태의 조회·시각화를 담당한다.

<a id="project-purpose-5"></a>

### 5. MVP 우선 구현 후 확장 모델 구조로 확장

4인·4주 범위에서는lb-02·maxscale-02와ANALYSIS 런타임을 제외한16개의 가상환경MVP로First Success와Must 기능, 자동화·장애·Recovery·부하 검증을 우선 완료한다.

<a id="project-technologies"></a>

## 프로젝트 진행 활용 기술

<a id="project-technologies-1"></a>

### 1. Infrastructure Automation

Ansible Role/Playbook, Inventory·group_vars·host_vars, Ansible Vault를 사용해 CentOS Guest 공통 설정, VRouter·Static Routing·Firewall, Kubernetes bootstrap, External Infrastructure를 코드로 관리한다.

<a id="project-technologies-2"></a>

### 2. Kubernetes Platform & Network

kubeadm 기반 Control Plane 3대·Worker 2대, Calico VXLAN, CoreDNS, Metrics Server, Gateway API + NGINX Gateway Fabric, MVP의 HAProxy LB-01을 사용한다.

<a id="project-technologies-3"></a>

### 3. Application & Realtime Processing

Nginx Frontend, FastAPI Backend, WebSocket, Redis 공유 상태·Pub/Sub·AOF/PVC를 사용한다.

<a id="project-technologies-4"></a>

### 4. Data, Storage & Recovery

MariaDB Primary/Replica, MaxScale 1대, External NFS + NFS Subdir External Provisioner를 사용한다.

<a id="project-technologies-5"></a>

### 5. CI/CD & GitOps

Jenkins(in-cluster) Build/Test → Harbor(external) Image Registry → Deployment Git Repository → Argo CD(in-cluster) → Kubernetes 흐름으로 구성한다.

<a id="project-technologies-6"></a>

### 6. Observability & Validation

Prometheus·kube-state-metrics는 Metrics 수집·상태 관측, Grafana Alloy는 Log 수집, Loki는 Log 저장·조회, Alertmanager는 Alert 라우팅, Grafana는 Metric·Log·Alert 상태의 조회·시각화를 담당한다.

<a id="project-learning-content"></a>

## 프로젝트 학습 내용

아래 세부 수행 내용을 기반으로 최종 부합여부 및 의견(코멘트) 작성 부탁드립니다.

| 세부 수행 내용(과업) | 편성 근거 |
| --- | --- |
| **On-premise 인프라 표준화·Ansible 자동화**<br>• Windows Host 4대·VMware 기반 가상환경 16개의MVP는 수동으로 생성하고 자동화 범위를 명확히 분리<br>• CentOS Stream 9 Guest baseline, 계정·NTP·방화벽·Static Route를 Ansible로 표준화<br>• Role/Playbook 기반으로 Network·Kubernetes·DB·Storage·CI/CD·관측 구성의 재사용 가능한 자동화 구조 설계<br>• 최초 실행·동일 조건 재실행·부분 실패·Drift 복원과 수동/자동Before·After 측정 | 여러 가상환경을 같은 기준으로 구축·재구축할 수 있어야On-premise 자동화의 효과를 설명할 수 있다. 가상환경 생성은 수동 기반으로 명확히 제외하고SSH 이후의 반복 설정을 코드화하여, 구성 편차·직접 개입·재실행 결과가 실제로 개선되는지 측정한다. |
| **Kubernetes 플랫폼·네트워크**<br>• kubeadm Control Plane 3대·Worker 2대와 stacked etcd quorum 구성<br>• Calico VXLAN, CoreDNS, NetworkPolicy, Metrics Server 구성<br>• VRouter 정적 라우팅과 Common Endpoint 10.1.93.90·LB-01 HAProxy L4 Listener 구성<br>• Gateway API + NGINX Gateway Fabric으로 HTTPS/WSS 진입 경로 구성 | Control Plane quorum, Worker 재스케줄, Pod Network와 공통Endpoint는 서비스가Kubernetes 위에서 계속 동작하기 위한 핵심 기반이다. MVP에서는 단일LB로Endpoint 계약을 먼저 검증하고, 확장 모델 확장에서Keepalived·두 번째LB를 추가해Failover 범위를 넓힌다. |
| **실시간 애플리케이션·상태 일관성**<br>• FastAPI 기반 Member/Guest·Room·Game·Vote·Realtime 핵심 기능 구현<br>• 15×15 렌주, 서버 측 deadline, game_id+turn_no 기준 단일 Move, Pass·재접속·결과 멱등 처리<br>• WebSocket + Redis 공유 상태·Pub/Sub로 다중Backend 간 현재 상태 동기화<br>• Backend 장애를 사용자 이탈·몰수·공동 패배로 오판하지 않는 복구 검증 | 실시간 투표에서는 여러 사용자의 요청이 동시에 도착하므로 중복 처리·stale 요청·연결 단절이Game State를 오염시키지 않아야 한다. 따라서 실제 서비스 기능을 인프라 검증Workload로 사용해 상태 일관성, 재접속, Backend 장애 후 연속성을 함께 검증한다. |
| **데이터베이스·스토리지·Backup/Recovery**<br>• MariaDB Primary/Replica와 GTID 기반 복제, MaxScale 1대를 통한 DB 접근 경계 구성<br>• External NFS, StorageClass/PV/PVC와 Redis AOF·Jenkins PVC 사용 경로 검증<br>• mariadb-backup·etcd Snapshot·Redis AOF 기반 Restore 절차 수행<br>• 복구 후 데이터 무결성, ACK 데이터 유실 여부, RTO/RPO 측정 | Replication과Backup/Restore는 해결하는 문제가 다르다. DB·NFS·PVC 장애 시나리오를 구분해 검증하고, DB 승격과 데이터Restore 후 무결성·Recovery Time을 측정해Availability와Recovery를 구분해서 설명할 수 있도록 편성한다. |
| **CI/CD·GitOps**<br>• Jenkins in-cluster Pipeline과 동적 Agent를 통한 Build/Test 및 Image 생성<br>• Harbor 외부 Registry에서 Image Tag/Digest 관리<br>• Deployment Git Repository와 Argo CD를 통한 선언 상태 동기화·Self-Heal<br>• Git Revert 기반 Rollback과 배포·복구 시간 측정 | Build/Test, Image 보관, Desired State 관리를 서로 다른 도구가 담당하게 해 변경 책임을 명확히 한다. 이를 통해 어느 단계에서 문제가 발생했는지 추적하고, Git과Image Digest를 기준으로 배포 상태와Rollback을 재현할 수 있게 한다. |
| **관측성·장애·부하 검증**<br>• Prometheus·Grafana·Loki·Grafana Alloy·Alertmanager와 E-mail 알림 구성<br>• 프로젝트용 Grafana 운영 Dashboard에서 핵심 메트릭·로그·알림 상태 확인<br>• Worker·DB·NFS·CI/CD·Physical Server 단위 장애 주입과 Recovery Timeline 기록<br>• HTTP/WSS/Vote 부하, HPA, 자동화·복구 Before·After의 Evidence 수집 | 운영 도구를 설치하는 것 자체가 목적이 아니라 장애·부하·Recovery의 결과를 설명할 근거를 확보하는 것이 목적이다. Metrics·Log·Alert·Timeline을 같은 Run ID로 연결하고 p95/p99응답시간·오류율·구축 시간·Recovery Time·RTO/RPO를 실제 측정값으로 남긴다. |

<a id="expected-competencies"></a>

- **Infrastructure Automation & Configuration Management**  
  Inventory·Variable·Role·Playbook을 구조화하고 Idempotency·부분 재실행·Drift 복원을 통해 여러 VM을 같은 기준으로 구축·재구축할 수 있는 역량
- **Kubernetes Platform Engineering**  
  kubeadm Control Plane quorum, Worker Scheduling/Rescheduling, Calico VXLAN, Gateway API·Service·HPA를 구성하고 남은 Worker의 자원·Image/PVC 접근 조건을 확인하면서 Node 장애가 서비스에 미치는 영향을 분석·Recovery할 수 있는 역량
- **Network & Traffic Engineering**  
  VRouter Static Routing, Source IP 보존, HAProxy L4 Listener, Common Endpoint·Port, Firewall·NetworkPolicy를 기준으로 서비스·API·DB의 Traffic Path와 Failure Domain을 설명하고 검증할 수 있는 역량
- **Realtime Service Reliability**  
  FastAPI·WebSocket·Redis 기반에서 동시 투표, deadline, 단일 Move, 재접속, stale 요청, 멱등 처리를 설계하고 여러 Backend의 Runtime State가 일관되게 유지되는지 검증할 수 있는 역량
- **Data Availability & Recovery**  
  MariaDB Replication, MaxScale 기반 DB Access, NFS/PVC, mariadb-backup·etcd Snapshot·Redis AOF Restore를 구성하고 Availability와 Recovery의 차이를 데이터 무결성·Recovery Time·RTO/RPO로 설명할 수 있는 역량
- **CI/CD & GitOps Engineering**  
  Jenkins·Harbor·Argo CD의 책임을 분리하고 Image Tag/Digest와 Git 변경 이력으로 Build·배포·Self-Heal·Rollback 흐름을 추적할 수 있는 역량
- **Observability & Incident Analysis**  
  Prometheus·Grafana·Loki·Grafana Alloy·Alertmanager의 Metrics·Log·Alert를 장애 Timeline과 연결해 원인·영향 범위·Recovery 시점을 분석할 수 있는 역량
- **Reliability & Performance Validation**  
  수동/자동 구축, HTTP/WSS/Vote 부하, 장애·Recovery 시험을 같은 조건에서 수행하고p95/p99응답시간·오류율·구축 시간·Recovery Time·RTO/RPO와Evidence로 설계 효과를 설명할 수 있는 역량

<a id="terminology"></a>

## 용어 설명

| 용어 | 이 문서에서의 의미 |
| --- | --- |
| Common Endpoint | 서비스·Kubernetes API·DB가 외부에서 접근할 때 사용하는 공통 주소. 같은IP를 유지하고Port별로 실제 목적지를 구분한다. |
| Runtime State | Room·Game·Turn·Vote처럼 게임 진행 중 빠르게 변하며 여러Backend가 함께 봐야 하는 현재 상태. |
| Source of Truth | 특정 영역에서 최종 기준으로 삼는 상태·데이터·선언. 게임 규칙과Move/Result 판단은Backend 도메인 로직과 서버 상태를 기준으로 하며, GitOps 배포 상태는Deployment Repository, 인프라 자동화 상태는Ansible 코드와 변수 정의를 기준으로 한다. |
| Idempotency | 같은 요청이나 자동화 작업이 다시 실행되어도 중복 결과를 만들지 않고 의도한 상태를 유지하는 성질. |
| Drift | 코드·설정으로 정의한Desired State와 실제 서버 상태 사이에 생긴 차이. Ansible 재실행으로 탐지·복원 가능한지 검증한다. |
| Availability<br>Recovery | Availability는 장애 중 서비스를 계속 제공할 수 있는 능력, Recovery는 장애·손실 이후 정상 상태나 데이터를 되찾는 능력을 의미한다. |
| Evidence | 시험 결과를 뒷받침하는 원본 근거. Run ID, Timeline, Log, Metrics, CSV/JSON/TXT 결과 등을 포함하며 Screenshot은 보조 자료로 사용한다. |
| First Success<br>Go·No-Go | First Success는 핵심E2E 경로가 처음으로 연결된 시점이며, Go/No-Go는MVP 검증 후 확장 모델 확장을 진행할지 판단하는Gate다. |

<a id="project-fit-review"></a>

| 평가분류 | 평가문항 | 평가점수 |
| --- | --- | --- |
| 프로젝트 부합 여부 | 해당 훈련생(팀)들 제시한 프로젝트 부제 및 프로젝트 목표가 선도기업 코오롱베니트에서 제시한 프로젝트 주제와 부합하는가?<br><br>\*선도기업에서 제시한 프로젝트 주제 및 수행 내용, 수요가 반영되었는지 확인 및 이에 대한 코멘트 작성 요망. | □ 매우적합<br>□ 적합<br>□ 부적합 |

20  .   .   .

<a id="reviewer-information"></a>

## 검토자 정보

|  |  |  |  |
| --- | --- | --- | --- |
| 기업명 |  |  |  |
| 부서명(직급) |  | 검토자 | (인) |
