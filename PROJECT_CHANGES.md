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
  - 상태 조회·변경 명령은 `/api/v1` HTTP JSON API, 실시간 서버 기준 Snapshot·Event 전달은 `/ws/v1` WebSocket을 기본으로 한다.
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

### Application 서비스 세부 구현 기준 1차 확정 및 방장 승계 규칙 변경

- 구분: 구현 단계 추가 확정 및 기존 동작 변경
- 기존 기준:
  - D01은 방장이 나가거나 30초 재접속 유예가 만료된 뒤 연결 중인 Member에게 방장을 승계하는 흐름을 설명했다.
  - D07은 방장이 변경되면 모든 Ready를 해제하도록 정의했다.
  - 인증 수명, HTTP 오류·멱등 처리, WebSocket 연결 단위, Redis Lifecycle·Lua 처리와 기존 MariaDB Schema 보완 상세는 구현 전에 확정할 필요가 있었다.
- 변경/확정 내용:
  - 방장이 명시적으로 퇴장하거나 연결 단절이 감지되면 30초를 기다리지 않고 접속 중인 Member 가운데 입장 순서가 가장 빠른 참가자에게 즉시 승계한다.
  - 실제 방장 변경과 모든 Ready 해제, Room 상태 Version 증가는 하나의 Redis Lua 실행으로 함께 처리한다.
  - 이전 방장이 30초 이내 재접속하면 참가자·팀·진행 중 Game 상태는 복원할 수 있지만 방장 권한과 단절 시점의 Vote는 자동 복원하지 않는다.
  - 승계 가능한 접속 중 Member가 없으면 Room을 종료하고 Guest에게 Room 종료를 알린 뒤 Lobby로 이동시킨다.
  - `WAITING` Room 종료에는 Game·Result·Rating 처리를 만들지 않는다. `PLAYING` Room 종료로 이미 생성된 Game을 계속할 수 없을 때만 개인 패배·전적·Rating을 반영하지 않는 `SYSTEM_INVALID`로 종결한다.
  - 인증은 Redis 서버측 Session Cookie, Origin·CSRF 검사와 Argon2id 비밀번호 저장을 사용한다.
  - 상태 변경은 `/api/v1` HTTP 명령, 실시간 전달과 복구는 `/ws/v1` Snapshot·Event를 사용하며, 요청 멱등성과 상태 Version을 검사한다.
  - Redis는 `stone:v1:` Key Prefix, Room 단위 Hash Tag, Version 관리 Lua와 서버 시각을 사용한다. MariaDB는 기존 7개 Table을 재사용하고 Application Migration으로 참가자 식별·중복 제약을 최소 보완한다.
- 영향:
  - 30초 Disconnect Lease는 참가자·팀·진행 중 Game 상태 복원에 적용하며 이전 방장의 권한을 예약하지 않는다.
  - 팀 변경 시 해당 참가자의 Ready 해제, 투표 시간 변경과 Game 종료 시 전체 Ready 해제 규칙은 유지한다.
  - 정확한 Endpoint·Event·오류 Code·Redis Key·Migration은 `seokpan-app`의 단일 구현 기준과 코드·Test로 관리한다.
  - 이 결정은 구현 방향을 확정한 것이며 Application, Provider, Container, Cluster 검증 완료를 의미하지 않는다.
- 관련:
  - `MVP_IMPLEMENTATION_BASELINE.md`
  - `seokpan/seokpan-app`

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

## 2026-09-01

### GitOps Root Application 선언 경로 및 Repository 역할 경계 확정

- 구분: 구현 구조 변경 및 공식 경로 확정
- 기존 기준:
  - `seokpan-gitops/apps/`는 Frontend/Backend 등 실제 Application Runtime Desired State를 관리하는 영역으로 정의되어 있다.
  - `apps/root/`는 최초 Argo CD 검증용으로 생성되었으나, 이후 `seokpan-infra`의 Argo CD Bootstrap과 Storage GitOps 작업에서 실제 Root Application source path로 사용되었다.
  - 현재 `apps/root/storage-nfs.yaml`을 통해 `storage-nfs` Child Application이 운영되고 있으므로 기존 경로를 즉시 제거할 수 없다.
- 변경/확정 내용:
  - Argo CD Child `Application` CR 선언의 공식 경로를 `argocd/applications/`로 사용한다.
  - `apps/`는 Frontend/Backend 등 실제 서비스 Application Runtime Desired State를 관리한다.
  - `platform/`은 Redis, Storage, Gateway, Namespace/RBAC 등 공통 Runtime Platform Desired State를 관리한다.
  - `cicd/`는 Jenkins 등 CI/CD Runtime Desired State를 관리한다.
  - `observability/`는 Prometheus, Grafana, Loki, Alloy, Alertmanager 등 관측 Runtime Desired State를 관리한다.
  - Redis Runtime Manifest는 기존 결정대로 `platform/redis/`를 유지하며, Redis Child Application 선언은 Root 구조 전환 완료 후 `argocd/applications/redis.yaml`에서 관리한다.
  - `seokpan-infra`의 Argo CD Bootstrap `gitops_root_path`는 `apps/root`에서 `argocd/applications`로 정합화한다.
  - 기존 `apps/root/`는 신규 경로 준비 → Root source path 전환 → Argo CD/Storage 회귀검증이 모두 완료된 후 제거한다.
- 영향:
  - Argo CD 제어 계층과 실제 Workload Desired State 경로를 분리하여 Repository 디렉터리의 역할을 단일 기준으로 해석할 수 있게 한다.
  - 현재 정상 동작 중인 `storage-nfs` Runtime을 재구축하지 않고 동일 Application 이름과 Workload 의미를 유지한 상태에서 경로만 안전하게 전환한다.
  - Root Application의 `prune/selfHeal` 영향을 고려하여 기존 `apps/root`를 먼저 제거하지 않는다.
  - 이 기록은 구조 결정의 확정을 의미하며 실제 GitOps 경로 전환, Infra Bootstrap 변경, Runtime 회귀검증 완료를 의미하지 않는다.
  - 원본 01-08 PDF와 Architecture 이미지는 변경하지 않는다.
- 관련:
  - `seokpan/seokpan-gitops#15`
  - `seokpan/seokpan-docs#15`
  - `seokpan/seokpan-gitops#7`
  - `seokpan/seokpan-gitops#10`
  - `seokpan/seokpan-gitops#13`

---

## 2026-09-02

### MariaDB read_only / auto_failover 정책 충돌 해결 및 GTID Domain 통합

- 구분: 기존 설정 변경
- 기존 기준:
  - `custom.cnf`에 `read_only` 값이 정적으로 고정되어 있어, MaxScale `auto_failover`가
    Master/Slave 역할을 전환해도 실제 `read_only` 상태가 새 역할과 어긋나는 충돌이
    있었다(이슈 #50).
  - 양쪽 서버의 `gtid_domain_id`가 통일되어 있지 않았다.
- 변경/확정 내용:
  - `custom.cnf`에서 `read_only` 항목을 제거하고, `enforce_read_only_slaves=true`
    설정으로 MaxScale이 Runtime에서 단독으로 `read_only` 상태를 관리하도록 변경했다.
  - `gtid_domain_id`를 양쪽 서버 모두 `1`로 통일했다.
  - 실서버(mariadb-01/02) 적용 및 재시작 검증 완료: Master는 재시작 후
    `read_only OFF` 유지, Slave는 재시작 후 `read_only ON`으로 정상 반영, GTID 완전
    일치, 복제 정상 확인.
- 영향:
  - Ansible이 `read_only`를 정적으로 관리하지 않고, MaxScale이 역할 변경에 따라
    동적으로 관리하는 구조로 바뀌었다.
  - 향후 failover/switchover 발생 시에도 `read_only` 상태가 실제 역할과
    불일치할 위험이 제거된다.
- 관련:
  - `seokpan/seokpan-infra#50`
  - `seokpan/seokpan-infra#88`

### MariaDB Master/Replica 역할을 설계 문서 고정값이 아닌 동적 조회 기준으로 운영

- 구분: 기존 가정 수정
- 기존 기준:
  - 05 물리 아키텍처 문서 등에서는 `mariadb-01`을 Master, `mariadb-02`를 Slave로
    전제하고 있었다.
- 변경/확정 내용:
  - MaxScale `auto_failover` 도입 이후 Master/Slave 역할은 장애·재부팅 상황에
    따라 자동으로 바뀔 수 있음을 확인했다(2026-09-01 기준 실제 Master는
    `mariadb-02`, Slave는 `mariadb-01`).
  - 문서상 고정 역할 표기는 참고값으로만 취급하고, 실제 작업 전에는 항상
    `maxctrl list servers`로 현재 역할을 재확인하는 것을 원칙으로 확정했다.
- 영향:
  - 계정/스키마 등 DCL·DDL 작업은 문서 기준이 아니라 실행 시점의 실제 Master를
    기준으로 수행해야 한다.
  - 향후 05 문서 갱신 시 "고정 역할"이 아닌 "동적 역할, MaxScale 관리" 방식으로
    표현 수정이 필요하다.
- 관련:
  - `seokpan/seokpan-infra#50`

### auto_rejoin GTID 갈라짐 시 자동 재편입 실패 확인

- 구분: Runtime 검증으로 기존 가정 수정
- 기존 기준:
  - MaxScale `auto_rejoin` 기능으로 장애 복구 후 Slave가 자동으로 복제에
    재편입될 것으로 가정하고 있었다.
- 변경/확정 내용:
  - PR #88 재부팅 검증 중, GTID가 갈라진 상태에서는 `auto_rejoin`이 안전을 위해
    재편입을 거부(실패)하고, `mariadb-backup` 기반 수동 재구축이 필요함을
    확인했다. GTID가 일치하는 상태에서는 `auto_rejoin`이 정상 동작함을 별도로
    확인했다.
- 영향:
  - F-06(Primary 장애) 시나리오에 "GTID 갈라짐 시 수동 재구축 필요" 케이스를
    반드시 포함해야 한다.
  - 이슈 #55(Backup/Restore 자동화)에 이 수동 재구축 절차를 반영해야 한다.
- 관련:
  - `seokpan/seokpan-infra#88`
  - `seokpan/seokpan-infra#55`

### 기존 app_user 계정 폐기, identity_svc/game_svc로 완전 전환

- 구분: 기존 계획 변경(항목 제거)
- 기존 기준:
  - MVP 초기에는 `app_user` 단일 계정(SELECT/INSERT/UPDATE/DELETE/EXECUTE ON
    stone_game.*)이 Backend Runtime 접근 계정으로 존재했다.
- 변경/확정 내용:
  - 개인정보(member)와 게임 데이터 간 신뢰 경계를 분리하기 위해 `app_user`를
    폐기하고 `identity_svc`/`game_svc` 2개 계정 체계로 완전히 전환하기로
    확정했다(2026-09-01).
  - 실서버 확인 결과 `app_user` 계정은 이미 존재하지 않는 상태였다(삭제
    시점·경위 기록 없음).
- 영향:
  - `stone_game_flow_test.sql` 등 `app_user` 기준으로 작성된 문서/스크립트는
    identity_svc/game_svc 조합 기준으로 갱신이 필요하다.
  - Backend 커넥션 설계는 단일 풀이 아닌 identity_svc/game_svc 2-커넥션 풀
    구조를 전제로 한다.
- 관련:
  - `seokpan/seokpan-infra#89`

### identity_svc/game_svc Runtime 계정 분리 코드화 및 game_svc Rating UPDATE 권한 추가

- 구분: 구현 단계 확정
- 기존 기준:
  - identity_svc/game_svc 계정과 권한은 수동으로 생성·검증된 상태였고, game_svc는
    `member(member_id, nickname, rating)` 컬럼 SELECT만 가능했다(UPDATE 권한 없음).
  - db_admin 계정의 사용 범위(Migration 전용 여부)는 별도로 명시되어 있지 않았다.
- 변경/확정 내용:
  - identity_svc/game_svc 계정 생성 및 GRANT를 Ansible로 코드화하기로
    확정(이슈 #89).
  - 경기 결과 확정 시 Rating을 반영할 수 있도록 `game_svc`에 `member(rating)`
    UPDATE 권한을 추가하기로 확정(이슈 #91).
  - `db_admin` 계정은 Alembic Migration 실행 전용으로 범위를 고정하고, 정상
    Backend Runtime 커넥션에는 제공하지 않기로 확정.
  - Infra(Ansible)는 `stone_game` Database와 계정·권한을 준비하고, Table 생성·변경을 포함한 Schema DDL은 `seokpan-app`의 Alembic Revision을 유일한 기준으로 관리한다.
- 영향:
  - Backend는 게임 결과 확정 트랜잭션(`game_result` INSERT + `game`
    UPDATE + `member` UPDATE(rating) + `member_stats` UPDATE +
    `rating_history` INSERT)에서 `game_svc` 계정만으로 Rating 갱신까지 처리할
    수 있게 된다.
  - Schema 구조 변경 권한과 계정 관리 권한의 소유 주체가 Infra/Backend로
    명확히 분리된다.
- 관련:
  - `seokpan/seokpan-infra#89`
  - `seokpan/seokpan-infra#91`

### Alembic Migration 실행 Gate 및 기존/신규 DB 적용 경계 확정

- 구분: 구현 단계 추가 확정
- 기존 기준:
  - `seokpan-app`이 SQLAlchemy Model·Table DDL·Alembic Revision을 소유하고, `db_admin`을 Migration 전용 계정으로 사용하기로 확정되어 있었다.
  - 기존 Runtime DB는 사전 DDL·행 Audit 후 Baseline Revision으로 Stamp하고, 신규 빈 DB는 동일 Revision Chain으로 생성하도록 경계가 정해져 있었다.
  - Migration은 Backend Replica 시작마다 실행하지 않고 단일 선행 Job 또는 운영 절차로 실행하는 원칙까지는 있었으나, 실제 Provider Integration에서의 실행 Gate는 구체화가 필요했다.
- 변경/확정 내용:
  - CI는 Alembic Revision Chain, Migration Test, Offline SQL 등 실제 DB를 변경하지 않는 정적 검증까지만 자동 수행한다.
  - 실제 DDL 또는 Stamp는 Backend Startup이나 각 Replica에서 자동 실행하지 않고, Provider Integration 단계에서 `db_admin` Credential을 사용하는 **승인된 단일 One-shot Migration 실행**으로 수행한다.
  - One-shot 실행 자산의 구체 형태와 Secret 참조 위치는 구현 단계에서 `seokpan-app#22`를 기준으로 확정한다.
  - 기존 Runtime DB는 Master·Replication, DDL·행 Audit, Backup/Restore·Rollback 가능 상태를 확인한 뒤 Baseline Revision으로 Stamp하고, 이후 필요한 Revision을 별도 승인 후 적용한다.
  - 신규 빈 DB는 Infra가 `stone_game` Database와 Migration 계정을 준비한 뒤 동일 Alembic Revision Chain에 대해 `alembic upgrade head`를 실행해 Schema를 생성한다.
  - Migration 성공 후 Revision, Replication, MaxScale Read/Write 및 기존 데이터 보존을 확인한 뒤 Backend Rollout을 진행한다.
  - `db_admin` Credential은 정상 Backend Pod에 주입하지 않는다.
- 영향:
  - Schema 변경 권한을 Runtime Backend에서 분리하면서도 신규 DB 재현성과 기존 DB 안전한 Baseline 편입을 같은 Alembic Revision Chain으로 관리할 수 있다.
  - Jenkins 일반 배포나 Argo CD Auto-Sync가 자동으로 DB DDL을 수행하는 구조로 해석하지 않는다.
  - 실제 Migration 실행·검증 완료 여부는 구현 Repository의 Issue/Evidence로 관리하며, 이 결정 기록만으로 Runtime 적용 완료를 의미하지 않는다.
- 관련:
  - `seokpan/seokpan-app#17`
  - `seokpan/seokpan-app#22`
  - `seokpan/seokpan-app#19`
  - `seokpan/seokpan-infra#55`
  - `seokpan/seokpan-infra#89`
  - `seokpan/seokpan-infra#91`
  - `seokpan/seokpan-docs#30`

---

## 2026-09-03

### Controller 관리자 kubeconfig 및 privileged Ansible 접근 기준 확정

- 구분: 구현 단계 추가 확정
- 기존 기준:
  - Kubernetes 관리자 kubeconfig는 Controller의 `/etc/seokpan/kubeconfig/admin.conf`를 공용 경로로 사용하도록 구성되어 있었다.
  - Controller에서 Kubernetes API를 사용하는 Ansible 자동화는 `cluster_kubeconfig` 변수를 통해 공용 kubeconfig를 참조하는 방향으로 통일되어 있었다.
  - 일반 Kubernetes 작업은 담당자별 ServiceAccount와 역할별 RBAC를 사용하는 구조로 구성되어 있었다.
  - 다만 개인 사용자 홈에 별도의 `kubernetes-admin` kubeconfig가 존재할 수 있었고, `ansible-kube` 그룹 멤버십에 대한 공통 운영 기준은 명확히 정의되어 있지 않았다.
- 변경/확정 내용:
  - Controller의 관리자 kubeconfig 기준 경로는 `/etc/seokpan/kubeconfig/admin.conf` 하나로 유지한다.
  - 개인 사용자 홈에는 별도의 `kubernetes-admin` kubeconfig를 운영 기준으로 유지하지 않는다.
  - 일반 Kubernetes 작업은 각 담당자의 기존 ServiceAccount와 역할별 RBAC에 연결된 개인 kubeconfig를 사용한다.
  - 역할별 개인 kubeconfig Credential은 TokenRequest 기반으로 발급하며, 유효기간 기준은 90일로 한다.
  - 역할별 Token이 만료되면 기존 ServiceAccount와 RBAC 범위를 변경하지 않고 동일 역할 범위로 재발급한다.
  - ServiceAccount Token과 kubeconfig 실제 Credential은 Git에 저장하지 않는다.
  - 비만료 ServiceAccount Token을 신규 운영 기준으로 사용하지 않으며, 별도 X.509 User 인증체계로 전환하지 않는다.
  - Controller에서 `cluster_kubeconfig`를 소비하는 승인된 privileged Ansible 작업은 공용 관리자 kubeconfig를 사용한다.
  - `ansible-kube` 그룹은 공용 관리자 kubeconfig에 대한 접근 제어 그룹으로 사용하며, `cluster_kubeconfig` 기반 privileged Ansible 작업을 실제 수행할 필요가 있는 Linux 사용자에게만 멤버십을 부여한다.
  - 팀원 전체 또는 일반 Kubernetes 작업 편의를 이유로 `ansible-kube` 멤버십을 일괄 부여하지 않는다.
  - privileged Ansible 작업 필요성이 없는 계정에는 관리자 kubeconfig 접근권한을 부여하지 않는다.
- 영향:
  - 관리자 kubeconfig 접근 지점을 Controller의 공용 경로로 단일화하여 cluster-admin Credential의 불필요한 개인별 복제를 방지한다.
  - 일반 Kubernetes 작업과 privileged Ansible 자동화의 인증 경로를 분리하여 역할별 RBAC와 관리자 권한의 사용 목적을 구분한다.
  - 역할별 kubeconfig의 만료·재발급 기준을 명시하여 TokenRequest Credential이 만료된 채 방치되는 운영 공백을 방지한다.
  - 담당자의 역할이 변경되거나 새로운 privileged Ansible 작업이 추가되는 경우에도 실제 실행 책임을 기준으로 관리자 kubeconfig 접근 여부를 판단한다.
- 관련:
  - `seokpan/seokpan-infra#90`
  - `seokpan/seokpan-infra#118`
  - `seokpan/seokpan-gitops#3`

### Gateway Argo CD 소유권 및 HTTPS Platform 완료

- 구분: 구현 상태 변화 및 책임 경계 확정
- 기존 기준:
  - Gateway/NginxProxy Manifest와 Runtime은 존재했지만, Argo CD 공식 Child Application 편입 전에는 `platform/gateway/base`가 Root App-of-Apps의 직접 관리 대상이 아니었다.
  - Gateway Base의 HTTP Listener와 Worker `30080` 경로는 검증됐으나, HTTPS는 `application/game-seokpan-tls` Secret이 없어 `InvalidCertificateRef` 상태로 Deferred돼 있었다.
  - 2026-09-02 Mentoring Baseline은 `Gateway HTTP 완료 / Gateway HTTPS·WSS 미완료`로 기록돼 있다.
- 변경/확정 내용:
  - `argocd/applications/gateway.yaml`을 통해 Gateway를 공식 Argo CD Child Application으로 편입하고, `platform/gateway/base`를 Gateway Runtime Desired State의 단일 Git 경로로 유지한다.
  - Gateway/NginxProxy/향후 HTTPRoute는 `seokpan-gitops`가 소유하고, Gateway 전용 Leaf Certificate 발급과 `application/game-seokpan-tls` Secret Provider는 `seokpan-infra`의 Ansible이 소유한다.
  - 기존 Seokpan Internal Root CA는 재사용하되 Harbor와 Gateway의 Leaf Certificate/Private Key는 분리한다.
  - Gateway TLS Secret 실제 값과 Private Key는 GitOps Repository에 저장하지 않는다.
  - `application/game-seokpan-tls`는 `kubernetes.io/tls` 타입으로 자동 생성·갱신하며, 재실행 시 유효한 Certificate/Secret 상태에서는 변경하지 않는다.
  - Runtime에서 HTTPS Listener `Accepted=True`, `ResolvedRefs=True`, `Programmed=True`를 확인했고 기존 `InvalidCertificateRef`가 제거됐다.
  - Worker-01/02의 `30080/30443` TCP, Worker `30443` TLS, Common VIP `10.1.93.90:443` TLS를 모두 검증했다.
  - 정상 Hostname `game.seokpan.soldesk.store`에서는 Worker/VIP HTTPS 요청이 TLS 검증 후 NGF의 `404` 응답까지 도달했고, 잘못된 HTTPS SNI Hostname은 `tlsv1 unrecognized name`으로 거부되는 것을 확인했다.
  - Argo CD `gateway` Application은 `Synced / Healthy` 상태를 확인했다.
- 영향:
  - Gateway Platform 수준의 HTTPS/TLS는 서비스 구현과 독립적으로 완료 상태로 전환됐다.
  - `Gateway HTTPS Platform PASS`는 실제 Frontend/Backend Route, HTTP→HTTPS Redirect, WebSocket/WSS, Browser First Success 완료를 의미하지 않는다. 해당 Application 통합 범위는 계속 Deferred한다.
  - 2026-09-02 Mentoring Baseline은 당시 시점의 역사 기록으로 유지하며, 본 항목을 후속 변화 근거로 연결한다.
  - 임시 Echo Service/HTTPRoute는 사용하지 않았으므로 별도 정리 대상은 없다.
- 관련:
  - `seokpan/seokpan-gitops#22`
  - `seokpan/seokpan-gitops` PR #23
  - `seokpan/seokpan-infra#111`
  - `seokpan/seokpan-infra` PR #112
  - `seokpan/seokpan-infra#118`
  - `seokpan/seokpan-infra#122`
  - `seokpan/seokpan-infra` PR #123
  - `seokpan/seokpan-docs#34`

---

## 2026-09-04

### Project Endpoint Registry 및 Kubernetes Pod 이름 해석 기준 확정

- 구분: 구현 단계 추가 확정 및 기존 누락 보완
- 기존 기준:
  - Host와 Kubernetes Node에서 필요한 프로젝트 FQDN은 `/etc/hosts`로 제공하고, Pod에서 필요한 프로젝트 Endpoint는 CoreDNS에서 별도로 제공하도록 05·06 설계에 정의돼 있었다.
  - 실제 구현에서는 Host/Node의 `/etc/hosts` 배포만 먼저 적용돼 있었고, Pod에서 사용하는 CoreDNS에는 프로젝트 FQDN이 등록되지 않아 BuildKit Agent가 `harbor.seokpan.soldesk.store`를 조회할 때 `SERVFAIL`이 발생했다.
- 변경/확정 내용:
  - 공용 `project_endpoints` 정의를 두고 Host `/etc/hosts`와 CoreDNS가 같은 Endpoint 값을 사용하도록 구성한다.
  - Host에서 필요한 Endpoint는 `common_hosts`, Pod에서 필요한 Endpoint는 Ansible `coredns_records` Role을 통해 각각 배포한다.
  - CoreDNS 전체 설정을 덮어쓰지 않고 관리 대상 `hosts` Block만 갱신하며, 변경 전 Backup, 사전 구조 확인, 변경 시 Rollout, 실패 시 복구 절차를 포함한다.
  - 현재 Host/CoreDNS에 적용하는 Endpoint는 다음과 같다.
    - `harbor.seokpan.soldesk.store → 192.168.53.61:443`
    - `db.seokpan.soldesk.store → 10.1.93.90:3306`
    - `game.seokpan.soldesk.store → 10.1.93.90:80/443`
    - `grafana.seokpan.soldesk.store → 10.1.93.90:443`
  - `k8s-api.seokpan.soldesk.store`는 API Server 인증서 SAN 정리 전이므로 Host/CoreDNS에 배포하지 않는다.
  - `jenkins.seokpan.soldesk.store`는 외부 Route 미구성 상태이므로 배포하지 않는다.
  - `argocd.seokpan.soldesk.store`는 MVP에서 외부 UI가 필수가 아니므로 배포를 보류한다.
  - Redis, Prometheus, Loki, Alertmanager 등 Kubernetes 내부 서비스는 기존 Kubernetes Service DNS를 계속 사용한다.
  - NFS는 현재 Storage Backend IP를 그대로 사용한다.
- 검증 결과:
  - Kubernetes Service DNS와 일반 External DNS 회귀검증 PASS
  - Harbor/DB/Game/Grafana 프로젝트 Endpoint 이름 해석 PASS
  - Pod → Harbor DNS, TCP/443, TLS Handshake 및 HTTP 응답 확인
  - Pod → DB DNS 및 TCP/3306 확인
  - Host `/etc/hosts` 적용과 Ansible 재실행 멱등성 확인
  - `seokpan-gitops#21`의 후속 BuildKit 검증에서 기존 Harbor DNS `SERVFAIL`이 재발하지 않았고 실제 Image Build/Push까지 이어서 확인
- 영향:
  - Host와 Pod가 같은 프로젝트 FQDN을 사용하되 이름 해석 경로는 Host `/etc/hosts`와 Kubernetes CoreDNS로 분리해 관리한다.
  - Backend는 개별 MariaDB 서버 IP 대신 `db.seokpan.soldesk.store:3306`을 사용한다. 실제 Backend Query와 Migration 검증은 Application/DB 통합 단계에서 별도로 수행한다.
  - `game.seokpan.soldesk.store`의 이름 해석과 Gateway HTTPS 기반 완료는 실제 Frontend/Backend Route 완료를 의미하지 않는다.
  - `grafana.seokpan.soldesk.store`의 이름 해석 완료는 Observability 서비스 전체 정상화를 의미하지 않는다.
- 관련:
  - `seokpan/seokpan-infra#99`
  - `seokpan/seokpan-infra` PR #108
  - `seokpan/seokpan-gitops#21`
  - `seokpan/seokpan-docs#33`

### WebSocket 메시지 순서와 상태 변경 번호의 역할 구분

- 구분: 구현 연동 규격 명확화
- 기존 기준:
  - WebSocket Envelope는 `event_id`, `room_id`, `game_id`, `state_version`을 필요한 범위에서 전달하고, Version 누락·중복·역전을 발견하면 Snapshot을 다시 받도록 정해져 있었다.
  - App 구현에는 Room과 Game/Vote 상태 변경 번호가 각각 존재하지만, 한 Room WebSocket에서 두 종류의 Event가 함께 전달될 때 Envelope의 `state_version`에 어느 값을 사용할지는 명확히 구분되지 않았다.
- 변경/확정 내용:
  - WebSocket Envelope의 `state_version`은 Lobby 전체 메시지 흐름 또는 각 Room 메시지 흐름의 순서를 나타내며 메시지 흐름별로 독립적으로 증가한다.
  - 같은 Lobby 또는 Room을 구독하는 연결은 같은 메시지 순서 기준을 사용하며, Socket 연결마다 별도의 순서 번호를 새로 시작하지 않는다.
  - Snapshot 안의 Room·Game 객체는 HTTP 상태 변경과 오래된 요청 검사에 사용하는 각자의 `state_version`을 그대로 유지한다.
  - Event Payload에서 Room 또는 Game 상태 변경 번호가 필요하면 `room_state_version` 또는 `game_state_version`으로 구분해 전달한다.
  - 최초 Snapshot과 이후 Event는 같은 메시지 순서 흐름을 사용하며, Room 상태와 Game/Vote 상태가 번갈아 바뀌어도 Envelope Version은 뒤로 가지 않는다.
  - 같은 상태 변경 Event를 다시 전달할 때는 최초 발행 때 정한 `event_id`와 Envelope `state_version`을 그대로 사용한다.
  - 클라이언트는 이미 처리한 `event_id`를 다시 적용하지 않고, Envelope Version이 건너뛰거나 재연결되면 Snapshot을 다시 받아 상태를 맞춘다.
- 영향:
  - Room과 Game/Vote의 상태 변경 번호를 하나로 합치지 않으며 기존 WebSocket Envelope에 새 Version 필드를 추가하지 않는다.
  - HTTP `expected_state_version`의 의미와 오래된 상태 거부 규칙은 변경하지 않는다.
  - 실제 Redis Pub/Sub, 여러 Backend 사이의 메시지 순서 공유와 재전달 검증은 Application Provider 통합 단계에서 수행한다.
- 관련:
  - `seokpan/seokpan-docs#41`
  - `seokpan/seokpan-app#3`
  - `seokpan/seokpan-app#46`
  - `seokpan/seokpan-app` PR #49

### MaxScale TLS 연결 및 공개 CA 전달 기준 확정

- 구분: 구현 연동 규격 확정
- 기존 기준:
  - Backend는 변동 가능한 MariaDB Master IP 대신 MaxScale/Common Endpoint를 사용하고, Runtime 계정과 Migration 계정을 분리하도록 정해져 있었다.
  - Infra PR #137의 담당자 실행 자료에서 MaxScale `Read-Write-Listener:3306`의 TLS 전용 전환, `db.seokpan.soldesk.store` SAN과 CA·Hostname 검증 완료가 보고됐다.
  - Worker OS의 CA Trust는 Backend Container에 자동으로 전달되지 않으며, DB TLS 연결 설정과 Kubernetes CA 전달 방법은 구현 단계 확정 항목으로 남아 있었다.
- 변경/확정 내용:
  - Backend Runtime과 Alembic Online Migration은 실제 Master IP나 Common VIP를 연결 문자열에 직접 사용하지 않고 `db.seokpan.soldesk.store:3306`으로 접속한다.
  - Identity·Game Runtime은 기존 `SEOKPAN_IDENTITY_DATABASE_URL`, `SEOKPAN_GAME_DATABASE_URL`을 유지하고, 승인된 단일 Migration 실행만 `SEOKPAN_MIGRATION_DATABASE_URL`을 사용한다.
  - 공개 Root CA 파일 경로는 세 DB URL에 반복하지 않고 `SEOKPAN_DATABASE_CA_FILE=/etc/seokpan/pki/ca.crt` 하나로 제공한다.
  - Application은 공개 CA와 Hostname 검증이 켜진 공통 SSL Context를 Identity·Game Runtime Engine과 Alembic Online Engine에 적용한다.
  - DB URL에 별도 CA 경로나 Hostname 검증 해제 등 TLS 정책을 우회하는 Query Option을 두지 않는다. 실제 Runtime과 Online Migration은 CA 파일이 없거나 읽을 수 없으면 시작하지 않는다.
  - Alembic Offline SQL 생성은 DB와 CA 파일에 접근하지 않고 계속 실행할 수 있어야 한다.
  - Application Runtime GitOps는 `application` Namespace의 `seokpan-internal-ca` ConfigMap, Key `ca.crt`를 `/etc/seokpan/pki/ca.crt`에 읽기 전용으로 Mount한다.
  - Identity·Game DB URL은 일반 Backend 전용 Secret 참조로 제공하고, Migration URL과 `db_admin` Credential은 일반 Backend Deployment에 넣지 않는다.
  - 이번 DB TLS 연결 구성에는 공개 Root CA만 전달한다. CA Private Key와 MaxScale 서비스 Private Key는 App·GitOps Repository, Image 또는 Kubernetes에 복사하지 않는다. 공개 Root CA 인계 시 X.509 SHA-256 Certificate Fingerprint를 함께 대조한다.
  - 서비스 Leaf Certificate의 현재 발급 유효기간은 365일이며, `tls_deploy`의 30일 기준은 인증서 전체 유효기간이 아니라 만료 전 재발급 판단 구간이다.
- 영향:
  - App의 TLS 연결 구현은 `seokpan-app#50`에서 진행한다. `seokpan-gitops#29`가 만든 Application Runtime 기반 위에 공개 CA Mount·DB Secret 참조·단일 Migration Workload를 추가하는 작업은 별도 GitOps Issue와 PR로 진행한다.
  - 공용 `tls_deploy` Role의 SAN 변경 감지는 `seokpan-infra#138`에서 병렬로 보완한다. 이 작업은 App TLS 연결 구현을 막지 않지만 실제 배포 전 공개 CA와 Fingerprint를 인계받아야 한다.
  - 구체 DB Secret Resource 이름과 단일 Migration Workload 이름은 실제 Provider 활성화 작업에서 확정한다.
  - 문서 반영과 App 정적 테스트는 실제 Migration·Backend DB 연결 완료를 의미하지 않는다. 승인된 Migration 실행 후 Backend 1 Replica에서 Identity·Game DB TLS 연결을 확인하고 이후 2 Replica로 확장한다.
- 관련:
  - `seokpan/seokpan-app#22`
  - `seokpan/seokpan-app#50`
  - `seokpan/seokpan-infra#102`
  - `seokpan/seokpan-infra#138`
  - `seokpan/seokpan-infra` PR #137
  - `seokpan/seokpan-gitops#29`

### MariaDB Backup 전략 변경 — 단순 Full 1일 1회 → Full+Incremental GFS 체이닝

- 구분: 기존 값 변경
- 기존 기준:
  - 06 문서 17.1절은 MariaDB Backup을 "mariadb-backup Full 1일 1회 시작값"으로
    정의하고, Retention은 "최근 7개"로 정의했다.
- 변경/확정 내용:
  - Full 백업은 주 1회(일요일)만 수행하고, 나머지 요일은 Incremental 백업으로
    전환하는 GFS 방식 체이닝 자동화로 변경했다(이슈 #114, PR #125).
  - Full이 GTID 미변경으로 스킵되어 체인이 없는 주에는 그 시점 Incremental을
    Full로 자동 승격해 RPO 공백을 방지하는 로직을 추가했다(이슈 #129, PR #131).
  - Retention은 "최근 7개(횟수 기준)"에서 "7일(기간 기준)"로 변경했다
    (`backup_transfer_retention_days: 7`).
  - 기존 단순 Full 전용 백업 스크립트(`backup_full.sh`)와 관련 산출물은
    체이닝 방식으로 완전히 대체되어 제거했다(이슈 #128, PR #130).
  - 백업 대상은 auto_failover로 역할이 바뀔 수 있는 서버 중 실행 시점에
    Replica로 판별되는 서버로 고정한다(운영 Primary 부하 회피 목적).
- 영향:
  - RTO/RPO 계산 시 "1일 1회 Full" 단일 기준이 아니라 "체인 내 마지막
    Incremental 시점"을 기준으로 재계산해야 한다.
  - Restore 절차(DR-01)도 단일 Full Restore가 아니라 Full→Incremental
    순서로 Roll-forward하는 `mariadb_restore_chain.yml` 기준으로 변경됐다.
  - 이슈 #129의 자연 체인 만료(주차 롤오버) 최종 검증은 아직 진행 중이라,
    승격 로직의 실서버 완전 검증 완료 여부는 별도로 확인한다.
- 관련:
  - `seokpan/seokpan-infra#55`
  - `seokpan/seokpan-infra#114`
  - `seokpan/seokpan-infra#128`
  - `seokpan/seokpan-infra#129`
  - `seokpan/seokpan-infra` PR #117, #125, #130, #131

---

## 2026-09-07

### Application 구현·검증 단계 순서 정합화

- 구분: 구현·검증 순서 변경 및 기존 기록 간 불일치 해소
- 기존 기준:
  - 공용 구현 기준 3절은 실제 MariaDB·Redis 연동 테스트를 Headless·Frontend보다 먼저 두었다.
  - 2026-08-30의 Application MVP 구현 기준 기록과 App Roadmap은 Headless·Frontend 뒤에 Provider·배포 통합을 두었다. App README와 내부 구현 기준은 실제 Provider를 Frontend보다 먼저 두어 문서별 순서가 일치하지 않았다.
- 변경/확정 내용:
  - 단계 번호는 `seokpan-app` Roadmap #3을 따른다.
  - A-07은 Fake Provider 기반 Headless HTTP/WebSocket First Success E2E로 수행한다.
  - A-08에서 Frontend First Success를 구현하고, A-09에서 기존 Container·Jenkins 자산을 이어받아 P0~P2 검증을 마무리한다.
  - A-10에서 실제 MariaDB·Redis 연결, 승인된 Migration, Backend 1→2 Replica와 Gateway·GitOps P3 통합을 검증한다. 실제 환경의 D07 M5 First Success와 장애·복구·대표 부하 등을 포함한 MVP P4도 이 단계에서 완료한다.
  - D07 M5의 M3 Runtime·M4 Delivery/Observability 선행 조건은 그대로 적용한다.
  - A-04/A-05의 실제 Provider 검증 잔여 항목은 A-10의 실행 결과로 확인한다. Fake·Scripted Provider 테스트가 실제 DB·Redis·Kubernetes 검증을 대신하지 않는다.
- 변경 이유:
  - A-07의 자동 테스트로 Frontend가 사용할 HTTP·WebSocket·결과 조회 규격을 먼저 확인할 수 있다.
  - Kubernetes 통합에는 Linux Image와 Delivery 산출물이 필요하므로 Container·Jenkins 검증을 선행한다.
  - 실제 Provider 준비 상태가 Frontend 구현을 막지 않도록 하면서 실제 통합 검증은 MVP 완료 전 필수 조건으로 유지한다.
- 영향:
  - MVP 기능, MariaDB·Redis 저장 책임, 기존 Schema 재사용, 보안 및 Migration 승인 조건은 유지한다.
  - 이 순서는 Application의 완료 판정 순서이며, 준비된 Provider의 개별 검증이나 다른 담당자의 Platform·Delivery 병렬 작업을 막지 않는다.
  - A-07·A-08·A-09 완료만으로 실제 Provider 통합, D07 M5 또는 MVP P4 완료를 선언하지 않는다.
  - 공용 `MVP_IMPLEMENTATION_BASELINE.md` 3절을 갱신한다. App README·내부 구현 기준의 같은 순서 반영은 App #53에서 진행한다.
- 관련:
  - [Application Roadmap](https://github.com/seokpan/seokpan-app/issues/3)
  - [Prototype 검증 관점 추적](https://github.com/seokpan/seokpan-app/issues/21)
  - [Migration 실행 및 검증](https://github.com/seokpan/seokpan-app/issues/22)
  - [MaxScale TLS 연결](https://github.com/seokpan/seokpan-app/issues/50)
  - [A-07 Headless First Success](https://github.com/seokpan/seokpan-app/issues/53)

---

## 2026-09-08

### DR 백업 보호 위치 전략 변경

- 구분: 기존 값 변경 및 기존 계획 범위 축소
- 기존 기준:
  - 06 문서 17.1절은 MariaDB를 "NFS Staging + loadgen Disk" 이중 저장으로,
    etcd Snapshot을 "loadgen Disk" 단독 저장으로 정의했다.
  - loadgen(외부 관리·부하 Server)은 DR 백업 보호 위치이자 격리된 etcd Restore
    실행 환경으로 원 설계에 반영되어 있었다.
  - 그러나 MVP 축소 설계 및 실제 구현(`backup_transfer` role, 이슈 #55/#114)
    단계에서는 loadgen 저장 로직이 처음부터 구현되지 않았고, NFS 전송·검증까지만
    구현되어 있었다. 이 차이는 문서화되지 않은 채로 진행되어 왔다.
  - DR-02(#113) 1단계 수동 검증(Snapshot 생성·NFS 전송·무결성 검증) 진행 중
    이 격차가 명시적으로 드러났다.
- 변경/확정 내용:
  - loadgen(격리 서버) 구성 자체는 계획에서 제거하지 않는다. loadgen 구축에 필요한
    자원(VM/서버)은 확보되어 있음을 팀 회의에서 확인했다.
  - 다만 프로젝트 기간이 짧고, 현재 진행 중인 인프라 구성·부하 테스트·시연 준비·
    발표 자료 준비 등 기간 내 필수로 완료해야 할 작업량이 많아, loadgen 구성을
    후순위로 조정한다.
  - 시간이 허락하면 loadgen 구성을 진행하고, 그렇지 않으면 NFS 서버를 실질적인
    백업·복구 저장소로 채택하여 다음 순서로 진행한다.
    1. NFS 기반 백업·복구(Restore)가 실제로 정상 동작하는지 수동 검증
    2. 정상 동작 확인 후 Ansible 자동화 코드화
    3. 자동화 결과에 대한 회귀 테스트로 이상 여부 확인
  - 이 순서는 DR-01(MariaDB)에서 이미 적용한 "수동 검증 → 결과 기록 →
    자동화 코드화" 패턴과 동일하다.
  - etcd DR(#113/#156)의 격리된 3-member Restore, quorum, Kubernetes API/Object
    검증, RTO 측정은 loadgen 진행 여부와 무관하게 1차 핵심 검증으로 계속 진행한다.
  - Restore 검증에 필요한 환경은 정식 loadgen 서비스 전체가 아니라, 운영
    cp1~cp3과 Data Directory·Member Name·Peer/Client URL·Cluster Token·Network·
    kubeconfig/API Endpoint를 분리한 소형 VM 3개(`restore-etcd-01/02/03`)를
    별도로 신규 구성하는 최소 환경이다. 이는 loadgen 여부와 독립적으로 착수한다.
- 판단 근거:
  - loadgen 구현 자체가 부적절해서가 아니라, 4인 팀·1차 프로젝트 짧은 일정
    안에서 인프라 구성·부하테스트·시연/발표 준비 등 기간 내 필수 작업의
    우선순위가 loadgen 신규 구축보다 높다고 판단했다.
  - "loadgen 미구현"과 "DR Restore 핵심 검증 미수행"은 서로 다른 문제다.
    loadgen은 백업 저장 위치이자 부하 생성 환경이고, DR Restore 검증(#113/#156)이
    요구하는 것은 "운영과 물리적/논리적으로 격리된 환경"이며 이는 소형 VM
    3개로도 충분히 충족 가능하다. 따라서 loadgen 후순위화가 DR-02의 완료
    기준(Restore·quorum·API·Object·RTO) 자체를 낮추는 근거가 되지 않는다.
  - NFS는 13절 HA·SPOF 표에 이미 "의도적 SPOF"로 명시되어 있어, NFS 단독
    저장은 리스크가 있으나, 현재 가용 자원 내에서 백업·복구 검증을 우선
    진행하고 이상 없음을 확인한 뒤 자동화하는 것이 프로젝트 기간 내 가장
    현실적인 순서라고 판단했다.
  - MariaDB Backup(이슈 #55/#114)에서도 유사한 축소가 사실상 선행되어
    있었으나 명시적으로 기록되지 않았던 것을, 이번 etcd DR 검증을 계기로
    전체 DR 계획 차원에서 공식화한다.
- 영향:
  - DR-02(#113) 완료 기준 14개 항목 중 Snapshot 생성·무결성·백업 저장 관련
    항목만 이번 라운드에서 충족하며, 운영/복구 완전 분리·quorum·API 응답·RTO
    측정 관련 항목은 격리 서버 확보 후 후속 이슈에서 충족한다.
  - DR-01(MariaDB)·DR-03(Redis)도 동일 기준으로 "로컬+NFS 저장까지 1차 완성,
    격리 서버 이중화는 스트레치 목표"로 범위를 통일한다.
  - 상위 DR 추적 이슈(#143)의 완료 판정 시 이 범위 축소를 전제로 반영해야 한다.
  - 포트폴리오 서술 시 "원 설계(loadgen 이중화) 대비 실제 구현 범위와 그
    트레이드오프를 인지하고 선택한 근거"로 이 기록을 사용한다.
- 관련:
  - `seokpan/seokpan-infra#113`
  - `seokpan/seokpan-infra#143`
  - `seokpan/seokpan-infra#55`
  - `seokpan/seokpan-infra#114`
  - `seokpan/seokpan-infra` PR #152

---

## 2026-09-08 — Application A-08

### Application 인증 복구와 화면 상태 재조회 보완

- 구분: 기존 인증 방식의 구현 상세 추가
- 기존 기준:
  - Redis 서버측 Session Cookie와 일반 상태 변경 HTTP의 Origin·CSRF 검사를 사용한다.
  - 세션 발급 응답으로 받은 CSRF가 화면 메모리에서 유실되면 Cookie가 유효해도 기존 Session 조회로 복구할 수 없었다.
- 변경/확정 내용:
  - `POST /api/v1/session/csrf`에서 같은 세션의 난수 CSRF와 현재 신원을 반환한다. 유효 Cookie·정확한 허용 Origin·`X-CSRF-Bootstrap: 1`·JSON 요청을 요구하며 Referer만으로 허용하지 않는다. 이 조회에만 기존 CSRF를 요구하지 않고 일반 API 검사는 유지한다.
  - 반복 복구는 CSRF·Session ID·Idle/Absolute 만료·참가 상태를 바꾸지 않는다. CSRF 포함 응답은 캐시를 금지하고 브라우저 메모리에만 보관하며 URL·로그·Socket·일반 상태 응답으로 노출하지 않는다.
  - 로비·Room/Game 재조회는 상태와 메시지 순서 기준을 함께 제공한다. 기존 Socket을 교체하지 않고 메시지 순서 번호와 Resource 변경 검사 번호를 구분한다. 서버 시각·마감 시각은 남은 시간 표시에 사용하고 게임 판정은 서버에 둔다.
  - 단순 창 복귀 시 인증 확인 중 조작을 막은 채 기존 화면을 유지할 수 있다. 다른 신원·만료·확인 실패에는 이전 화면을 폐기하고, 결과 불명 명령을 자동 재실행하지 않는다.
- 영향:
  - App Session 저장 규격·Adapter·HTTP/OpenAPI·Frontend 복구 흐름에 적용한다. Cookie 속성·Session TTL·Member/Guest 권한과 실제 게임 단절 시 방장 승계·Ready·표 처리 기준은 유지한다.
  - 구형 자료를 요청 중 자동 변환하거나 삭제하지 않는다. 실제 배포 전 자료·실행 버전·되돌리기 방법을 확인하고 필요한 전환은 별도 승인 절차로 처리한다.
- 관련:
  - [MVP 구현 기준 5절](MVP_IMPLEMENTATION_BASELINE.md#5-httpwebsocket-연동-규격)
  - [App #56](https://github.com/seokpan/seokpan-app/issues/56), [App PR #59](https://github.com/seokpan/seokpan-app/pull/59)
  - [인증 복구 구현·시험 설명](https://github.com/seokpan/seokpan-app/blob/2fc8393241b98e842e9532b85c7a35c72ab115f3/frontend/docs/api-and-session.md)

### Application 화면 완료 범위와 조회 정보 보완

- 구분: 서비스 구현 범위 추가 확정·조회 정보 보완
- 기존 기준:
  - D07 3쪽은 핵심 게임 흐름과 Should 보조 기능을 구분했다. 공용 Frontend 기준도 채팅·고급 UI를 First Success의 선행조건으로 두지 않았다.
  - D01 11쪽은 보드·투표 현황·채팅·게임 방법·랭킹·접속자·사용자 메뉴를, 19-21쪽은 로비·관전·채팅·재경기를 설명한다. 목업의 배치와 기능 구성을 반영하면서 A-08 화면 완료 범위를 구체화할 필요가 있었다.
- 변경/확정 내용:
  - 기존 가입·입장·투표·결과·다음 판 흐름에 로비/방 채팅, WAITING 방장 강퇴, 진행 중 방 관전, 공개 랭킹·Member 내 전적, 접속자 표시, 게임 방법·사용자 메뉴를 포함한다. D07의 원 분류는 이력으로 보존하되 A-08에서는 이 보조 기능을 제외하지 않는다.
  - 보드·사이드에 서버 집계 기준 득표 수·비율을 표시한다. 분모는 해당 턴 유효 투표자 수이며 개인별 표는 공개하지 않는다. `last_move`는 마지막 공식 착수 번호·팀·좌표로 제공하고 착수 전 null, Pass 뒤 유지, 새 Game 초기화를 적용한다.
  - 공개 누적 전적과 본인의 경기별 Rating 변동을 구분한다. D01 31쪽의 유효 경기·정렬·Guest 영구 전적 제외 기준을 유지한다.
  - D07 4쪽의 ANALYSIS 제외는 유지한다. 화면에는 정적 미제공 안내만 두고 분석 API·모델·Workload, 공개 복기·채팅 영구 이력은 추가하지 않는다. 목업의 상이한 수치·권한은 서비스 규칙으로 채택하지 않는다.
- 영향:
  - App 서버·화면·회귀 시험의 완료 범위에 반영한다. 기존 게임 규칙·Member/Guest 권한·MariaDB/Redis 책임 및 A-08 → A-09 → A-10 순서는 바꾸지 않는다.
  - 자료 형식 변경의 구·신버전 호환성과 되돌리기는 실제 배포 전에 별도로 확인한다. 화면 시험만으로 실제 DB·Redis·배포 통합 완료를 선언하지 않는다.
- 관련:
  - [MVP 구현 기준 7절](MVP_IMPLEMENTATION_BASELINE.md#7-frontend-기준)
  - [App #56](https://github.com/seokpan/seokpan-app/issues/56), [App PR #59](https://github.com/seokpan/seokpan-app/pull/59)
  - [Application Roadmap #3](https://github.com/seokpan/seokpan-app/issues/3)
  - [App 구현 기준](https://github.com/seokpan/seokpan-app/blob/2fc8393241b98e842e9532b85c7a35c72ab115f3/docs/mvp-implementation-baseline.md)

### 비영속 채팅과 사용자 단위 접속자 집계 구분

- 구분: 포함 기능의 전달·집계 방식 상세화
- 기존 기준:
  - D01 21쪽은 로비/방 채팅 범위와 입장 이후 메시지 전달·영구 이력 미제공을 정의했다. 공용 WebSocket 설명은 Lobby/Room 상태 Snapshot·Event 복구를 중심으로 작성돼 있었다.
  - D01 11·19쪽은 접속자 표시를 정의했으나 여러 탭·기기의 중복 집계 단위는 지정하지 않았다.
- 변경/확정 내용:
  - 채팅은 세션·범위·입력 검사를 거친 HTTP 전송과 별도 수신 WebSocket을 사용한다. 메시지 규격 버전·고유 ID를 사용하고 Room/Game 상태 번호나 대화 이력 Snapshot은 추가하지 않는다. 신원·참가·연결 변경 시 권한을 다시 확인한다.
  - 접속자는 Member 계정 ID·Guest 임시 사용자 ID별로 중복을 제거한다. 한 사용자의 다른 유효 연결이 있으면 유지하며, 마지막 유효 연결이 없어졌을 때 제외한다. 서로 다른 Guest 신원은 각각 집계한다.
  - 인증된 `/ws/v1/presence`는 접속 확인용 ping/pong만 주고받고 전체 숫자만 공개한다. 실패를 정상 0명으로 표시하지 않으며, 네트워크 단절 감지 전까지 즉시 반영된다고 보장하지 않는다.
  - 채팅·접속자 연결은 기존 게임 상태 연결과 분리한다. 접속 확인은 인증 Idle TTL을 늘리지 않으며 채팅 장애·접속자 변동을 게임 단절·Ready 해제·표 삭제·방장 승계·승패 사유로 삼지 않는다.
- 영향:
  - App의 채팅·접속자 연결과 화면에 적용한다. 기존 Lobby/Room 상태 연결의 수신 전용·순서 검사·Snapshot 복구는 유지한다.
  - 채팅 Redis 전달·접속자 Redis 공유 집계는 A-10에서 구현·검증한다. 랭킹 MariaDB 연결·다중 Replica·Gateway/WSS·자료 전환도 별도 검증하며 Memory 시험을 실제 통합 성공으로 취급하지 않는다. 시험용 시간·수용량을 운영 기본값으로 확정하지 않는다.
  - Infra/GitOps의 Endpoint·Namespace·인증서·Secret·배포 설정과 담당 업무는 이번 문서 변경으로 수정하지 않는다.
- 관련:
  - [MVP 구현 기준 5.2절](MVP_IMPLEMENTATION_BASELINE.md#52-채팅접속자-연결의-구분)
  - [App PR #59](https://github.com/seokpan/seokpan-app/pull/59)
  - [채팅 규격·검증 범위](https://github.com/seokpan/seokpan-app/blob/2fc8393241b98e842e9532b85c7a35c72ab115f3/backend/docs/chat-delivery.md), [접속자 집계 규격·검증 범위](https://github.com/seokpan/seokpan-app/blob/2fc8393241b98e842e9532b85c7a35c72ab115f3/backend/docs/presence.md)

---

## 2026-09-09

### Redis Recovery 장애 주입·최소 권한 경계 확정

- 구분: 구현 단계 추가 확정
- 기존 기준:
  - 역할별 Kubernetes 작업 영역은 Namespace와 ServiceAccount/RBAC로 분리하고, 일상 작업에 불필요한 광범위 권한을 부여하지 않는다.
  - 김상희의 `platform` Redis/Recovery 쓰기 권한은 실제 Recovery 작업에서 필요한 범위가 확인될 때 최소 권한으로 추가하기로 유보했다.
  - Redis Runtime은 `platform`, NFS Provisioner 및 Storage 검증 Resource는 `storage-infra` Namespace를 사용한다.
- 확정 내용:
  - Redis Recovery 검증에서 실제 Runtime 장애 주입은 Data, Storage & Recovery 담당 김상희가 직접 수행한다.
  - 기존 `platform/ksh` ServiceAccount를 재사용하며 `platform` 전체 관리자 권한은 부여하지 않는다.
  - 실제 Redis Runtime `redis-0`에 대해서만 Pod 삭제와 Pod Log 조회에 필요한 최소 권한을 추가한다.
  - 기존 Pod/PVC/PV/Event/StatefulSet 등의 일반 상태 조회는 기존 Cluster read-only 권한을 계속 사용한다.
  - Container 내부에서 임의 명령 실행이 가능한 `pods/exec` 권한은 실제 Runtime 최소 권한에서 제외한다.
  - 실제 `platform` Redis Runtime에서는 데이터나 PVC 자체를 직접 훼손하지 않는 Pod 재기동 수준의 장애 주입만 수행한다.
  - AOF 파일 교체·변조, PVC 손상 모사 등 파괴적 Recovery 검증은 실제 Runtime PVC가 아니라 `storage-infra` Namespace의 격리된 테스트 Resource에서 수행한다.
  - Redis Runtime Manifest와 Kubernetes Runtime 운영 책임은 기존대로 Kubernetes & Application Integration 담당이 유지하며, Redis AOF/PVC Recovery와 MariaDB 권위 데이터 기준 정합성 검증 책임은 Data, Storage & Recovery 담당이 유지한다.
- 현재 상태:
  - 위 운영 결정과 권한 경계는 확정됐다.
  - 실제 RBAC Desired State 반영, Argo CD Sync, 허용·거부 권한 및 Runtime 검증은 `seokpan-gitops#47`에서 진행한다.
  - 실제 Redis Recovery 실행 및 결과 Evidence는 `seokpan-infra#115`에서 관리한다.
- 영향:
  - RBAC 반영 후 Data 담당이 Recovery 검증에 필요한 Redis Pod 장애 주입을 직접 수행할 수 있도록 한다.
  - `platform` 전체 권한을 확대하지 않고 실제 DR 검증에 필요한 작업만 허용하도록 최소 권한 경계를 유지한다.
  - 실제 Runtime과 파괴적 Recovery 검증 환경의 장애 영향 범위를 분리한다.
  - 이번 권한 추가는 Redis Runtime 소유 책임을 Data 담당에게 이전하는 것이 아니다.
- 관련:
  - `seokpan/seokpan-gitops#3`
  - `seokpan/seokpan-gitops#25`
  - `seokpan/seokpan-gitops#47`
  - `seokpan/seokpan-gitops#7`
  - `seokpan/seokpan-infra#115`

---

## 2026-09-10

### DB 서비스 계정 Operator Credential 소비 경로 확정

- 구분: 구현 단계 운영 기준 추가 확정
- 기존 기준:
  - `identity_svc`, `game_svc`, `db_admin` 계정과 권한은 Ansible로 관리하고, DB Password의 Source of Truth는 Ansible Vault로 유지한다.
  - Application Runtime은 `identity_svc`/`game_svc`, Migration은 `db_admin`을 사용하며 Kubernetes에서는 Runtime/Migration Secret을 분리해 소비한다.
  - MaxScale 공식 Endpoint `db.seokpan.soldesk.store:3306`은 Internal CA 기반 TLS와 Server Certificate 검증을 사용한다.
  - DB 담당자가 수동 검증·운영 작업에서 현재 회전된 Credential을 안전하게 소비하는 공식 절차는 별도로 확정되어 있지 않았다.
- 변경/확정 내용:
  - DB Password의 Source of Truth는 기존대로 Ansible Vault를 유지하며, 과거 공유 Password를 복원하거나 현재 DB Password를 추가 회전하지 않는다.
  - 승인된 DB Operator는 Ansible Controller에서 `identity_svc`, `game_svc`, `db_admin` 중 필요한 계정을 선택하고 Vault-backed Operator 경로로 현재 Credential을 소비한다.
  - 최종 Operator 실행 명령은 `./tools/db-operator-login <identity_svc|game_svc|db_admin>`으로 고정하며, 실제 작업 전 `~/work/seokpan-infra/ansible`에서 프로젝트 `.venv`를 활성화한 상태로 실행한다.
  - DB Password 원문은 사용자에게 전달하지 않고, 실행 중에만 Git 관리 경로 밖의 예측하기 어려운 임시 MariaDB Client 설정 파일에 기록한다.
  - 임시 Client 설정 파일은 Mode `0600`을 사용하고 정상 종료뿐 아니라 실패·Interrupt·Signal 경로에서도 삭제한다.
  - MariaDB Client는 공식 Endpoint 계약과 기존 Database 계약을 재사용하고, `--defaults-extra-file`, Internal Root CA, Server Certificate 검증을 사용해 TCP/TLS로 접속한다.
  - Ansible Controller에는 CA Private Key 접근 권한을 확대하지 않고, 기존 `ca_trust` Role을 재사용해 공개 Root CA 인증서만 System Trust Store에 배포한다.
  - DB Credential 변경 완료 판정 시 Ansible Vault·MariaDB·Kubernetes Runtime/Migration Secret뿐 아니라 DB Operator 수동 소비 경로도 함께 확인한다.
- 검증 결과:
  - `identity_svc`, `game_svc`, `db_admin` 모두 현재 Vault Credential로 공식 DB Endpoint 인증에 성공했다.
  - 잘못된 Synthetic CA를 사용한 Negative Control은 TLS 검증 오류로 거부되어 CA 검증 경로가 실제 적용됨을 확인했다.
  - `game_svc`의 허용된 Member 조회와 `member.rating` UPDATE는 성공했고, `login_id`/`password_hash` 조회는 거부됐다.
  - `identity_svc`의 Game Table 접근은 거부됐으며, 세 계정의 `SHOW GRANTS` 결과가 기존 권한 계약과 일치했다.
  - `backend_db_secrets.yml --check`는 `changed=0`, `failed=0`으로 Runtime/Migration Secret 소비 계약의 회귀가 없음을 확인했다.
  - Controller CA Trust 재실행은 `changed=0`이었고, 임시 Credential 파일은 검증 종료 후 잔존하지 않았다.
  - 실제 DB 담당자 `ksh` 사용자 컨텍스트에서도 Vault 인증 후 `game_svc@%`로 `stone_game` 접속에 성공했으며, 종료 후 해당 사용자 소유 임시 Credential 파일은 0개였다.
  - 실제 DB Password, 완성된 Credential URL, CA Private Key는 Git·Issue·PR·검증 출력에 기록하지 않는다.
- 영향:
  - 승인된 DB Operator는 회전 전 공유 Password를 별도로 전달받거나 기억해 사용할 필요 없이 현재 Vault 기준 Credential을 재현 가능하게 소비할 수 있다. 실제 사용자는 Ansible Controller 및 Vault 인증에 필요한 승인된 접근 권한을 별도로 보유해야 한다.
  - Application/Migration의 Kubernetes Secret 소비 구조, 기존 DB 계정·GRANT, `mariadb_account`의 Password 관리 정책은 변경하지 않는다.
  - 향후 Credential 회전 작업은 자동화 Consumer뿐 아니라 승인된 Operator 접근 경로까지 정상 전환됐는지 확인해야 완료 처리한다.
- 관련:
  - `seokpan/seokpan-infra#91`
  - `seokpan/seokpan-infra#159`
  - `seokpan/seokpan-infra#169`
  - `seokpan/seokpan-infra#172`
  - `seokpan/seokpan-infra#174`
  - `seokpan/seokpan-infra#175`

### MariaDB 백업 체인 상태(state) authority를 NFS 공유 스토리지로 이전

- 구분: 기존 구조 변경
- 기존 기준:
  - 2026-09-02 "MariaDB Backup 전략 변경" 기록 및 실제 구현(`backup_transfer`
    role)에서, 백업 체인 상태 파일(`.backup_chain_state.json`)과
    `--incremental-basedir` 참조 경로는 각 MariaDB 호스트의 로컬 디스크에
    있었다.
- 변경/확정 내용:
  - `auto_failover=true` 환경에서 실제로 Master/Slave 역할이 백업 스케줄과
    무관하게 바뀔 수 있음을 2026-09-09~10 장애 재현으로 재확인했다(수요일
    Full을 수행한 호스트가 failover로 바뀐 뒤, 다른 호스트가 다음 Incremental
    시도 시 직전 체인을 인식하지 못하고 불필요한 Full 승격·백업 실패 발생).
  - 상태 파일과 `--incremental-basedir` 참조를 호스트 로컬 디스크에서 NFS
    공유 스토리지로 이전해, 두 MariaDB 호스트가 어느 쪽이 백업을 수행하든
    동일한 체인 상태를 참조하도록 변경했다.
  - 상태 파일 갱신 시 임시파일을 최종 파일과 동일한 NFS 디렉터리에 생성하도록
    변경해, 파일시스템 간 이동(`mv`)으로 인한 원자성 손상을 방지했다.
  - NFS로 옮겨진 공유 상태에 대해 두 호스트 간 `flock` 기반 공유 lock을
    추가해 동시 read-modify-write(lost update)를 방지했다. 실측 중 크로스
    서브넷 NFSv4 환경에서 lock 해제 통지가 약 30초 폴링 주기로 지연되는
    특성을 확인하고, lock 대기 시간을 이에 맞춰 조정했다.
- 영향:
  - 백업 체인 상태의 단일 기준(single source of truth)이 호스트 로컬에서
    NFS 공유 스토리지로 이동했다 — 향후 DR-01 Restore 절차나 상태 조회
    자동화는 이 NFS 경로를 기준으로 참조해야 한다.
  - NFS 마운트 장애 시 상태 조회/갱신 자체가 실패할 수 있다는 새로운 의존성이
    생겼다(기존에도 마운트 여부를 사전 확인 후 중단하므로 큰 회귀는 아님).
- 관련:
  - `seokpan/seokpan-infra#166`
  - `seokpan/seokpan-infra#143`
  - `seokpan/seokpan-infra` PR #168

---

## 작성 형식

### 변경 또는 결정 제목

- 구분:
- 기존 기준:
- 변경/확정 내용:
- 영향:
- 관련:
