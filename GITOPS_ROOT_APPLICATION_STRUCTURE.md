# GitOps Root Application 공식 구조 결정

## 1. 목적

이 문서는 `seokpan-gitops`의 Argo CD Root Application 경로와 Repository 역할 경계를 구현 단계 공식 기준으로 기록한다.

기존 `apps/root/`는 최초 생성 당시 Argo CD 검증용으로 만들어졌으나, 이후 `seokpan-infra`의 Argo CD Bootstrap과 Storage GitOps 작업에서 실제 Root Application 경로로 사용되었다.

현재 Runtime 기능 자체는 정상 동작하고 있으나, `apps/`의 Application Runtime 의미와 `apps/root/`의 Argo CD 제어 계층 의미가 혼재되어 있어 Redis, Jenkins, Observability, Frontend, Backend의 후속 GitOps 편입 전에 경계를 정리한다.

이 결정은 01-08 공식 기획·설계의 서비스·인프라 아키텍처를 변경하는 것이 아니라, 구현 단계의 GitOps Repository 구조와 Argo CD 관리 경계를 명확히 하는 결정이다.

## 2. 확정 구조

```text
seokpan-gitops/
├── argocd/
│   └── applications/
│       ├── storage-nfs.yaml
│       ├── redis.yaml
│       ├── jenkins.yaml
│       ├── observability.yaml
│       ├── frontend.yaml
│       └── backend.yaml
│
├── apps/
│   ├── frontend/
│   └── backend/
│
├── platform/
│   ├── redis/
│   ├── storage-nfs/
│   ├── gateway/
│   └── namespaces-rbac/
│
├── cicd/
└── observability/
```

## 3. 역할 경계

### `argocd/applications/`

Argo CD Child `Application` CR 선언 전용 경로로 사용한다.

각 파일은 실제 Workload Manifest 자체가 아니라 Argo CD가 어느 Repository 경로를 어떤 Namespace에 동기화할지 정의한다.

후속 Redis, Jenkins, Observability, Frontend, Backend Application 선언도 동일 경로를 사용한다.

### `apps/`

Frontend와 Backend 등 실제 서비스 Application Runtime의 Kubernetes Desired State를 관리한다.

Application Source Code는 계속 `seokpan-app`이 소유하며, 이 경로에는 Kubernetes 배포 상태만 둔다.

### `platform/`

Redis, Storage Client, Gateway, Namespace/RBAC 등 공통 Runtime Platform의 Kubernetes Desired State를 관리한다.

Redis Runtime Manifest는 기존 결정대로 `platform/redis/`를 유지한다.

### `cicd/`

Jenkins 등 CI/CD Runtime Desired State를 관리한다.

### `observability/`

Prometheus, Grafana, Loki, Alloy, Alertmanager 등 관측 Runtime Desired State를 관리한다.

## 4. 기존 구조 처리

기존 `apps/root/`는 신규 공식 경로 전환이 완료될 때까지 즉시 삭제하지 않는다.

현재 `apps/root/storage-nfs.yaml`을 통해 실제 `storage-nfs` Child Application이 운영 중이므로 다음 순서를 따른다.

```text
신규 argocd/applications 경로 준비
→ storage-nfs Application 정의를 동일 이름·동일 의미로 신규 경로에 준비
→ seokpan-infra Argo CD Bootstrap의 Root source path 변경
→ Root Application 전환
→ Argo CD 및 Storage Runtime 회귀검증
→ 전부 PASS 후 기존 apps/root 제거
```

Root Application에 `prune` 및 `selfHeal`이 적용되어 있으므로 신규 경로 전환 전에 기존 경로를 먼저 제거하지 않는다.

## 5. Infra 정합화

`seokpan-infra`의 `argocd_bootstrap`은 현재 다음 값을 사용한다.

```yaml
gitops_root_path: "apps/root"
```

공식 구조 전환 시 다음 값으로 변경한다.

```yaml
gitops_root_path: "argocd/applications"
```

Root Application의 App-of-Apps 방식 자체는 유지한다.

다른 Task, Defaults, Template 또는 검증 코드에 `apps/root`가 직접 하드코딩되어 있는지 함께 확인하고 정합화한다.

## 6. Runtime 보호 원칙

- 기존 `storage-nfs` Runtime을 재구축하지 않는다.
- 기존 PVC, PV, NFS 데이터를 삭제하지 않는다.
- Root Application 전환 전후 `storage-nfs` Application 이름과 실제 Workload 의미를 유지한다.
- 수동 `kubectl edit/apply/delete`보다 Git Desired State 변경을 우선한다.
- 구조 변경 완료와 Runtime 회귀검증 완료를 구분해서 기록한다.
- 신규 경로 정상화 전에 기존 `apps/root`를 제거하지 않는다.

## 7. 전환 후 검증 기준

### Argo CD

- Root Application `Synced`
- Root Application `Healthy`
- Root Application source path가 `argocd/applications`
- `storage-nfs` Child Application 존재
- `storage-nfs` `Synced / Healthy`
- 의도하지 않은 Child Application 삭제 없음

### Storage

- NFS Provisioner Pod `Running`
- StorageClass `nfs-k8s` 유지
- 기존 PV/PVC 상태 정상
- 필요 시 Validation PVC/Pod를 통한 동적 Provisioning 확인
- NFS Mount 및 Write/Read 확인
- 기존 Runtime 데이터 영향 없음

### Repository

- 기존 `apps/root` 테스트 경로 제거
- README와 실제 Repository Tree의 역할 일치
- `seokpan-infra` Bootstrap과 `seokpan-gitops` Root 경로 일치
- 신규 Child Application 선언 경로가 `argocd/applications/`로 단일화

## 8. 후속 작업

구조 전환 완료 이후 다음 Child Application 선언은 모두 `argocd/applications/` 아래에서 관리한다.

- Redis
- Jenkins
- Observability
- Frontend
- Backend

Redis Runtime Manifest 자체는 기존 `platform/redis/` 구조를 유지하며 Root 구조 정합화 완료 후 `seokpan-gitops#7` 작업을 재개한다.

## 9. 관련 작업

- `seokpan/seokpan-gitops#15` — Root Application 경로 정합화 및 공식 구조 전환
- `seokpan/seokpan-gitops#7` — Redis Runtime StatefulSet 및 Backend 연결 구성
- `seokpan/seokpan-gitops#10` — NFS Provisioner·StorageClass GitOps Desired State 구성
- `seokpan/seokpan-gitops#13` — NFS Subdir External Provisioner/RBAC/StorageClass GitOps 편입
- `seokpan/seokpan-docs#15` — 본 결정 기록 작업

## 10. 상태

- 구조 결정: 확정
- 문서 기록: 진행 중
- GitOps Repository 구조 전환: 미적용
- Infra Bootstrap 경로 변경: 미적용
- Runtime 회귀검증: 미실시

구조 결정이 기록되었다는 사실만으로 실제 GitOps 전환과 Runtime 검증이 완료된 것으로 보지 않는다.
