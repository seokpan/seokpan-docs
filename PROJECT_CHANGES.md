# 石나가는 판단 프로젝트 변경·결정 이력

이 문서는 `seokpan-docs`의 01-08 공식 기획·설계 문서를 기준점으로 하여,
실제 구현 과정에서 새로 확정되거나 변경·추가·삭제된 사항을 기록한다.

현재 구현 상태 전체를 다시 설명하는 문서가 아니며,
기존 공식 문서와 달라진 부분 또는 구현 단계에서 새롭게 구체화된 사항만 기록한다.

## 기록 기준

다음에 해당하는 경우 기록한다.

- 기존 문서의 값이나 구조가 변경된 경우
- 문서에서 미확정이던 값이 구현 단계에서 확정된 경우
- 새로운 구성요소나 요구사항이 추가된 경우
- 기존 계획에서 제거된 항목이 있는 경우
- 구현 결과에 따라 아키텍처나 담당 범위가 변경된 경우
- 실제 Runtime 검증으로 기존 가정이 수정된 경우

단순 작업 진행 상황, 임시 테스트 결과, 아직 확정되지 않은 아이디어는 기록하지 않는다.

각 변경사항은 관련 Issue, PR, Commit 또는 Repository 경로를 함께 남긴다.

---

## 2026-08-28

### 서비스 외부 Hostname 확정

- 구분: 추가 확정
- 기존 기준:
  - 서비스 외부 Hostname은 구체적으로 확정되지 않은 상태였다.
- 확정 내용:
  - 서비스 외부 Hostname을 `game.seokpan.soldesk.store`로 사용한다.
  - 서비스 HTTPS 진입 주소는 `https://game.seokpan.soldesk.store`로 한다.
  - TLS 종료는 NGINX Gateway Fabric에서 수행한다.
  - Gateway 인증서 SAN에는 최소 `game.seokpan.soldesk.store`를 포함한다.
- 관련:
  - `seokpan/seokpan-gitops#5`

### Gateway 구현 버전 및 소유권 경계 확정

- 구분: 구현 단계 확정
- 기존 기준:
  - 01-08 공식 문서에서는 Gateway API와 NGINX Gateway Fabric 사용 방향을 정의했지만, 실제 구현 버전과 Ansible/GitOps 간 세부 소유권 경계는 구현 단계에서 확정할 필요가 있었다.
- 확정 내용:
  - Gateway API는 `v1.5.1`을 사용한다.
  - NGINX Gateway Fabric은 `v2.6.7`을 사용한다.
  - Ansible `k8s_addons` 영역은 Gateway API 표준 CRD, NGINX Gateway Fabric CRD, Controller Bootstrap 및 플랫폼 수준 `GatewayClass` 설치·검증을 담당한다.
  - GitOps는 프로젝트 서비스용 `NginxProxy`, `Gateway`, 이후 `HTTPRoute` 등 서비스 Desired State를 담당한다.
  - NGINX Gateway Fabric 공식 NodePort 전체 Manifest에는 기본 `NginxProxy`와 `externalTrafficPolicy: Local` 설정이 함께 포함되므로 그대로 적용하지 않는다. 프로젝트의 `NginxProxy`는 GitOps에서 `externalTrafficPolicy: Cluster`, 고정 NodePort `30080/30443` 기준으로 관리한다.
- 영향:
  - Ansible과 GitOps가 동일한 Gateway Data Plane Desired State를 동시에 소유하지 않도록 경계를 분리한다.
  - Gateway Add-on 자동화는 버전 고정된 upstream Manifest를 기준으로 재현 가능하게 구성한다.
- 관련:
  - `seokpan/seokpan-infra#12`
  - `seokpan/seokpan-gitops#5`

### Ansible Project 실행환경 Version Matrix 확정

- 구분: 구현 단계 확정
- 기존 기준:
  - Python 및 Ansible 관련 버전은 후보 계열로만 정의되어 있었다.
- 확정 내용:
  - Project Python: `3.12.13`
  - ansible-core: `2.20.8`
  - Kubernetes Cluster: `1.36.2`
  - Python kubernetes client: `36.0.3`
  - kubernetes.core: `6.5.0`
  - System Python 3.9 및 기존 System Ansible은 Rollback 용도로 유지한다.
- 관련:
  - `seokpan/seokpan-infra#38`
  - `seokpan/seokpan-infra#41`

### Observability 저장소 사용 기준 재확인

- 구분: 기존 설계 재확인
- 기존 기준:
  - 06 문서에서 Prometheus와 Loki는 Local PV를 사용하도록 정의되어 있다.
- 구현 단계 반영:
  - Prometheus와 Loki는 NFS Subdir Provisioner를 사용하지 않는다.
  - Prometheus/Loki의 배치 Worker와 실제 hostPath는 구현 단계에서 별도로 확정한다.
  - NFS Subdir Provisioner는 Redis, Jenkins Controller, MariaDB Backup Staging 용도로 사용한다.
- 영향:
  - NFS Subdir Provisioner는 Observability 구성의 선행조건으로 취급하지 않는다.

### Prometheus·Loki Local PV 배치 확정

- 구분: 구현 단계 확정
- 기존 기준:
  - Prometheus와 Loki는 Local PV를 사용하되 실제 Worker와 hostPath는 구현 단계에서 확정하기로 했다.
- 확정 내용:
  - Prometheus: `worker-01:/mnt/observability/prometheus`
  - Loki: `worker-02:/mnt/observability/loki`
  - Prometheus와 Loki를 서로 다른 Worker에 배치한다.
- 이유:
  - Worker 1대 장애 시 Metric과 Log 저장소가 동시에 영향을 받지 않도록 장애 영향 범위를 분리한다.
  - Prometheus/Loki의 Node 종속 Local PV 장애 실험을 각각 독립적으로 확인할 수 있도록 한다.
- 영향:
  - Observability Manifest에서 Prometheus는 `worker-01`, Loki는 `worker-02`에 고정 배치하도록 구성한다.
  - 각 Local PV는 위 hostPath를 기준으로 생성한다.
- 관련:
  - Observability 구현 Issue/PR에서 실제 Manifest와 검증 결과를 연결한다.

### Storage Infrastructure 전용 Namespace 추가

- 구분: 구현 단계 추가 확정
- 기존 기준:
  - 01-08 설계에서는 External NFS, NFS Subdir External Provisioner, StorageClass/PV/PVC 역할을 정의했으나 Provisioner 전용 Kubernetes Namespace는 별도로 확정하지 않았다.
  - 구현 초기 Namespace는 `application`, `platform`, `observability`, `cicd` 중심으로 구성했다.
- 확정 내용:
  - NFS Subdir External Provisioner와 Storage 검증용 PVC/Pod를 분리 관리하기 위해 `storage-infra` Namespace를 추가한다.
  - 김상희의 기존 `platform/ksh` ServiceAccount를 `storage-infra` Namespace의 작업 권한에 연결한다.
  - Redis Runtime StatefulSet/PVC는 Storage Infrastructure와 분리하여 기존대로 `platform` Namespace를 유지한다.
  - StorageClass, PV, Provisioner용 ClusterRole/ClusterRoleBinding 등 Cluster-scoped 리소스 권한은 개별 작업자의 상시 권한으로 넓게 부여하지 않고 변경이 통제되는 Bootstrap/GitOps 범위에서 적용한다.
- 영향:
  - Kubernetes 프로젝트 Namespace가 구현 단계 기준으로 `application`, `platform`, `observability`, `cicd`, `storage-infra`로 구체화된다.
  - `seokpan-gitops/platform/namespaces-rbac/`의 Namespace/RBAC Desired State를 수정한다.
  - NFS Provisioner 구현 및 검증은 `storage-infra`, Redis Runtime은 `platform`이라는 경계를 사용한다.
- 관련:
  - `seokpan/seokpan-gitops#3`
  - `seokpan/seokpan-gitops#8`
  - `seokpan/seokpan-infra#44`

---

## 2026-08-30

### Harbor 내부 Hostname 및 TLS 기준 변경

- 구분: 기존 값 변경 및 역할 확정
- 기존 기준:
  - Harbor 내부 접근 이름은 `harbor.stone.test`를 사용했다.
  - 내부 CA 생성·보관과 Kubernetes 노드 신뢰 배포의 담당 경계는 구현 단계에서 구체화할 필요가 있었다.
- 변경/확정 내용:
  - Harbor 공식 서비스 FQDN은 `harbor.seokpan.soldesk.store`를 사용한다.
  - Harbor VM 주소는 기존과 동일한 `192.168.53.61`을 유지한다.
  - 구현 문서와 Ansible Role에서 VM은 `harbor-01`, Ansible Inventory Host는 `harbor`로 식별한다. 두 이름은 TLS 서비스 FQDN과 구분한다.
  - Harbor 인증서 SAN에는 최소 `DNS:harbor.seokpan.soldesk.store`와 `IP:192.168.53.61`을 포함한다.
  - 별도 내부 DNS가 준비되기 전에는 필요한 Host와 Kubernetes Node에 `/etc/hosts` 매핑을 배포한다.
  - Harbor 담당은 내부 CA와 Harbor 인증서를 준비하고 CA Private Key를 GitHub에 저장하지 않는다.
  - Kubernetes Node에는 CA 공개 인증서만 배포하며, Worker/containerd의 신뢰와 실제 Image Pull은 별도 검증한다.
- 영향:
  - 현재 설정과 신규 문서에서 `harbor.stone.test`를 현행 Endpoint로 사용하지 않는다.
  - Host 이름 해석, 인증서 SAN, Harbor Client와 Worker/containerd의 CA 신뢰를 같은 FQDN 기준으로 맞춘다.
  - `argocd`, `grafana`, `jenkins` 등 후속 내부 서비스는 `<service>.seokpan.soldesk.store` 명명 방향을 사용하되, 실제 적용·검증 완료는 각 구현 작업에서 별도로 확인한다.
- 관련:
  - `MVP_IMPLEMENTATION_BASELINE.md`
  - `seokpan/seokpan-infra`의 Harbor 및 CA 신뢰 자동화

### Application MVP 구현 기준 확정

- 구분: 구현 단계 추가 확정
- 기존 기준:
  - 01-08 문서는 Python/FastAPI Modular Monolith, Nginx Frontend, MariaDB·Redis 책임, WebSocket 실시간 처리와 Jenkins·Harbor·GitOps 흐름을 정의했다.
  - Application의 정확한 Runtime·Framework Version, HTTP/WebSocket 연동 규격, Migration 소유권, Frontend Stack과 단계별 검증 Gate는 구현 착수 전에 구체화할 필요가 있었다.
- 변경/확정 내용:
  - 기존 MariaDB Schema를 재사용하고 Domain → Adapter → Headless HTTP/WebSocket → Frontend → Provider·배포 통합 순서로 점진 구현한다.
  - Backend는 CPython 3.13.15 기반 FastAPI Modular Monolith와 실용적 Ports/Adapters 구조를 사용한다.
  - 상태 조회·변경 명령은 `/api/v1` HTTP JSON API, 실시간 권위 Snapshot·Event 전달은 `/ws/v1` WebSocket을 기본으로 한다.
  - MariaDB 영속 데이터와 Redis 공유 Runtime State의 기존 책임을 유지하며, Application이 SQLAlchemy Model·Alembic Migration·Redis Key/Lua 처리 규격을 소유한다.
  - Frontend는 React·TypeScript·Vite 기반 정적 Application으로 구성하고 Nginx에서 제공한다.
  - Backend와 Frontend는 별도 Image·Workload로 배포하며 Jenkins Test → Harbor Image → GitOps PR → Argo CD 흐름을 유지한다.
  - Windows Host에서 개발하되 Application 실행 자산은 CentOS Stream 9·Linux Container 기준 UTF-8·LF로 관리하고 Linux 재현 Gate를 둔다.
  - Frontend UX/UI Mockup이 제공되는 경우 공식 명세가 아닌 방향성 참고자료로 적용한다.
  - 세부 Version, 실행·환경·검증 상태와 미확정 항목은 `MVP_IMPLEMENTATION_BASELINE.md`를 따른다.
- 영향:
  - `seokpan-app`의 Scaffold·Lock·Test·Container·Pipeline은 공용 구현 기준을 소비한다.
  - `seokpan-infra`는 Provider·Registry·실행환경을 제공하고, `seokpan-gitops`는 Kubernetes Desired State와 Secret 참조를 소유한다.
  - 결정 완료와 실제 Linux·Provider·Cluster 검증 완료를 구분한다.
- 관련:
  - `MVP_IMPLEMENTATION_BASELINE.md`
  - `seokpan/seokpan-app`
  - `seokpan/seokpan-infra`
  - `seokpan/seokpan-gitops`

---

## 2026-08-31

### MaxScale MVP 구현 버전 조정

- 구분: 기존 값 변경
- 기존 기준:
  - D06의 Version Lock 기준은 MariaDB `11.8.9 LTS`, MaxScale `24.02.10`이었다.
- 변경/확정 내용:
  - MariaDB `11.8.9 LTS` 기준은 유지한다.
  - MaxScale의 MVP 구현 버전은 `24.02.9`로 조정한다.
  - MaxScale `24.02.10`은 2026-06-15 발표된 GA Release이지만, 2026-08-31 확인 시점의 Community 공개 Repository에서는 설치 가능한 Package가 제공되지 않았다.
  - Community Repository에서 설치 가능한 동일 `24.02` 계열의 최신 Patch인 `24.02.9`를 사용한다.
  - 추후 `24.02.10` Package가 Community Repository에 제공되더라도 자동 Upgrade하지 않는다. 별도 변경 작업에서 Package 가용성, Monitor·Router, Replication, Read/Write와 재실행 멱등성을 검증한 뒤 변경 여부를 결정한다.
- 영향:
  - Infra 자동화와 Version Lock은 MaxScale `24.02.9`를 정확한 설치 버전으로 사용한다.
  - 실제 설치·복제·Routing 검증 완료 여부는 구현 Repository의 작업 결과로 관리하며, 이 결정 기록만으로 Runtime 검증 완료를 의미하지 않는다.
  - 원본 D06 PDF는 변경 전 기준으로 보존한다.
- 관련:
  - [MaxScale 24.02.10 Release Notes](https://github.com/mariadb-corporation/mariadb-docs/blob/main/release-notes/maxscale/24.02/24.02.10.md)
  - [MariaDB MaxScale 24.02 Community 공개 목록](https://dlm.mariadb.com/browse/mariadbmaxscale/24.02/)
  - `seokpan/seokpan-infra#63`
  - `seokpan/seokpan-infra#65`

### Git Branch 운영 방식 유지 결정

- 구분: 협업 운영 기준 확정
- 기존 기준:
  - 각 Repository는 최신 `main`에서 Feature/Task Branch를 생성하고 `Issue → Branch → Commit → Pull Request → Review → Squash Merge` 흐름으로 운영하고 있었다.
  - 프로젝트 진행 중 개발·안정화·완료 상태를 Branch 수준에서 분리하기 위해 장기 `dev → staging → main` 구조 도입을 대안으로 검토했다.
- 확정 내용:
  - 1차 프로젝트에서는 현재의 `main` 중심 Feature/Task Branch + Pull Request 운영 방식을 유지한다.
  - 작업 Branch는 최신 `main`을 기준으로 생성하고, 완료 후 `main`을 대상으로 Pull Request를 생성한다.
  - Review 및 필요한 검증을 통과한 변경만 Squash Merge한다.
  - 장기 `dev`, `staging` Branch는 1차 프로젝트에서는 도입하지 않는다.
- 판단 근거:
  - `dev → staging → main` 방식 자체가 부적절해서 제외한 것이 아니다. 실제 Development / Staging / Production 환경이 분리되어 있거나 Release Gate가 필요한 팀에서는 충분히 적절한 전략이다.
  - 현재 프로젝트는 4인 팀, 짧은 1차 일정, 기존 `main` 중심 Workflow 운영 중, 별도 Dev/Staging Runtime 미구성이라는 조건이다.
  - 현 시점에서 장기 Branch를 추가하면 여러 Repository의 기준 Branch 변경, Branch 간 동기화, Merge 순서 및 Conflict 관리 비용이 늘어난다.
  - 실제 Dev/Staging 실행환경이 없는 상태에서 Branch만 추가할 경우 환경 분리 효과는 제한적이며, 1차 프로젝트 핵심 목표인 On-premise 구축·Ansible 재현 자동화·서비스 통합·정량 검증에 대한 직접 기여가 낮다.
  - 따라서 복잡한 Branch 전략을 몰라서 사용하지 않는 것이 아니라, 현재 프로젝트 조건에서 추가 복잡도를 도입하지 않기로 선택한 것이다.
- 향후 재검토:
  - 실제 Development / Staging / Production Runtime 환경을 별도로 운영하는 경우
  - `main` Merge와 Production 배포 시점을 분리해야 하는 경우
  - 여러 기능의 장기 통합 검증 또는 Release Candidate 승인 Gate가 필요한 경우
  - 2차 프로젝트에서 환경별 GitOps 구조가 필요한 경우
  - 재검토 시에는 Branch 이름만 추가하기보다 `dev → Development`, `staging → Staging`, `main → Production`처럼 실제 배포환경과 연결되는 구조를 우선 검토한다.
- 관련:
  - `seokpan/seokpan-docs#6`

---

## 작성 형식

### 변경 또는 결정 제목

- 구분:
- 기존 기준:
- 변경/확정 내용:
- 영향:
- 관련:
