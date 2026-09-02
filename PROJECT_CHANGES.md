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

### Application 서비스 세부 구현 기준 1차 확정 및 방장 승계 규칙 변경

- 구분: 구현 단계 추가 확정 및 기존 동작 변경
- 기존 기준:
  - D01은 방장이 나가거나 30초 재접속 유예가 만료된 뒤 연결 중인 Member에게 방장을 승계하는 흐름을 설명했다.
  - D07은 방장이 변경되면 모든 Ready를 해제하도록 정의했다.
  - 인증 수명, HTTP 오류·멱등 처리, WebSocket 연결 단위, Redis Lifecycle·원자 처리와 기존 MariaDB Schema 보완 상세는 구현 전에 확정할 필요가 있었다.
- 변경/확정 내용:
  - 방장이 명시적으로 퇴장하거나 연결 단절이 감지되면 30초를 기다리지 않고 접속 중인 Member 가운데 입장 순서가 가장 빠른 참가자에게 즉시 승계한다.
  - 실제 방장 변경과 모든 Ready 해제, Room 상태 Version 증가는 하나의 Redis 원자 처리로 수행한다.
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
    `read_only OFF` 유지, Slave는 재시작 후 `read_only ON`으로 수렴, GTID 완전
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
  - Infra(Ansible)는 Database·계정·권한 및 7개 Table 생성까지만 담당하고, 이후
    컬럼/제약조건 등 스키마 구조 변경(Alembic Migration)은 Backend 책임 범위로
    분리.
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

---

## 작성 형식

### 변경 또는 결정 제목

- 구분:
- 기존 기준:
- 변경/확정 내용:
- 영향:
- 관련:
