# Redis Runtime MVP Resource Baseline

이 문서는 `seokpan-gitops#7`의 Redis Runtime 구현에 사용할 **최초 Resource Baseline**을 기록한다.

이 값은 성능 충분성이 검증된 최종 용량이 아니다. 01-08 공식 설계, 현재 MVP 물리 자원, Storage Runtime 검증 결과와 Redis의 실제 책임 범위를 기준으로 구현 착수 전에 정한 시작값이며, First Success 및 부하 시험 결과에 따라 유지 또는 조정한다.

## 1. 기준 문서 및 현재 구현 상태

### 01-08 공식 기준

- D05의 Worker 시작값은 `8 vCPU / 28GB RAM / 140GB Disk`이며, ANALYSIS Runtime이 MVP에서 제외되더라도 축소하지 않는다.
- D05의 Worker 1대 장애 수용 예산에서 `Redis·ANALYSIS`의 상시 Request 예산 상한은 `CPU 1 vCPU / Memory 3GB`이다.
- D05는 위 값을 장애 시 살아남은 Worker 한 대의 수용 예산으로 정의하며, 실제 Kubernetes `requests/limits`는 부하 시험으로 확정하도록 한다.
- D06은 Redis를 단일 Stateful Workload로 두고 NFS PVC와 AOF `appendfsync everysec`을 사용한다. Sentinel/Cluster는 MVP 기본 범위가 아니다.
- D07은 ANALYSIS Runtime을 MVP에서 제외하지만 Redis AOF/PVC와 Runtime State 복구는 유지한다.

### 현재 구현·Runtime 기준

- Redis Version: `8.10.1`
- Namespace: `platform`
- StatefulSet: 1 Replica
- Service: Backend용 `redis` ClusterIP + StatefulSet용 `redis-headless`
- StorageClass: `nfs-k8s`
- `seokpan-infra#44`, `seokpan-gitops#10`, `seokpan-gitops#13`에서 동적 PV/PVC, NFS 하위 디렉터리 생성, Pod Mount/Write/Read와 `archiveOnDelete=true` 동작까지 검증 완료
- Redis는 Session, Room, Participant, Ready, Runtime Game/Turn, Current Vote와 reconnect Snapshot 등 공유 Runtime State를 담당한다.
- MariaDB의 공식 Move, GameResult, Rating/RatingHistory 등 영속 비즈니스 원본을 Redis가 대체하지 않는다.

## 2. MVP 최초 Resource Baseline

| 항목 | 시작값 |
| --- | --- |
| CPU request | `500m` |
| CPU limit | `1` |
| Memory request | `1Gi` |
| Memory limit | `3Gi` |
| Redis `maxmemory` | `1Gi` |
| Redis `maxmemory-policy` | `noeviction` |
| PVC request | `5Gi` |
| StorageClass | `nfs-k8s` |
| AccessMode | `ReadWriteOnce` |
| AOF | `appendonly yes` |
| AOF fsync | `appendfsync everysec` |
| RDB Snapshot | `save ""` |

## 3. Resource Baseline 근거

### CPU

- `500m` request는 Redis가 서비스 Runtime의 핵심 공유 상태 계층이라는 점을 반영해 Scheduler가 지나치게 작은 Pod로 취급하지 않도록 하기 위한 최초 예약값이다.
- `1` CPU limit은 D05의 `Redis·ANALYSIS` CPU Request 예산 상한 1 vCPU와 현재 MVP에서 ANALYSIS가 제외된 점을 함께 고려한 시작값이다.
- Redis의 실제 CPU 포화, throttling, latency가 확인되기 전에는 더 큰 값을 성능 요구사항으로 확정하지 않는다.

### Memory

- `1Gi` request는 Redis가 단순 캐시가 아니라 활성 Room/Game/Vote/Session 및 reconnect Snapshot을 공유하는 Runtime State Store라는 점을 반영한다.
- `3Gi` limit은 D05의 `Redis·ANALYSIS` Memory 예산 3GB 안에서 Redis 단독 MVP Runtime에 충분한 상한 여유를 주기 위한 시작값이다.
- Redis AOF rewrite와 allocator/RSS overhead 때문에 Redis dataset 크기와 Pod 전체 RSS는 동일하지 않다. 따라서 Pod limit과 Redis `maxmemory`를 같은 값으로 두지 않는다.

### Redis maxmemory / eviction

- Redis `maxmemory`는 `1Gi`로 시작한다.
- Pod Memory limit `3Gi`와 분리하여 AOF rewrite, allocator overhead, client/output buffer 등 dataset 외 메모리 사용 여유를 남긴다.
- `maxmemory-policy noeviction`을 사용한다. Redis가 메모리 한계에 도달했을 때 기존 Runtime State를 임의로 제거하기보다 새 write를 실패시켜 장애를 명시적으로 드러내도록 한다.
- Redis 공식 문서도 persisted instance에서는 `maxmemory` 외의 buffer/overhead를 위한 RAM 여유를 남기도록 안내하며, write-heavy 상황의 AOF rewrite에서는 평상시보다 큰 추가 메모리가 필요할 수 있음을 명시한다.

## 4. PVC Baseline 근거

- Redis PVC 요청량은 `5Gi`로 시작한다.
- Redis의 MVP 데이터는 서비스 Runtime State와 AOF이며, 현재 External NFS는 Redis·Jenkins Shared PVC 및 MariaDB Backup Staging을 함께 수용한다.
- NFS Subdir External Provisioner의 PVC `requests.storage` 값은 NFS filesystem quota와 동일한 물리적 강제 상한으로 간주하지 않는다.
- 따라서 `5Gi`는 Kubernetes Desired State의 최초 요청값이며, 실제 `/srv/nfs/k8s` 사용량과 Redis AOF 증가량을 별도로 측정한다.

## 5. 실측 후 동결 기준

최초 배포 후 다음 항목을 Evidence로 수집한다.

- Kubernetes Pod CPU 사용량 및 CPU throttling
- Pod Memory Working Set / RSS
- Redis `used_memory`, `used_memory_peak`, `used_memory_rss`
- Redis `mem_not_counted_for_evict`
- Redis OOM/write error 및 `evicted_keys`
- Redis command latency
- AOF current/base size, rewrite 실행과 실패 여부
- NFS 실제 Redis 하위 디렉터리 사용량
- NFS I/O latency와 Redis Pod 재기동 시 복구 시간
- Worker 1대 장애 시 남은 Worker의 전체 Request 수용 상태

실측 전에는 현재 값을 “최적값”, “성능 충분성이 검증된 값”, “최종 Capacity”라고 표현하지 않는다.

부하 시험과 장애 시험 결과로 조정이 필요하면 변경 전/후 값, 측정 근거와 영향 범위를 `PROJECT_CHANGES.md` 및 관련 Issue/PR에 기록한다.

## 6. 적용 경계

- 이 Baseline은 `seokpan-gitops/platform/redis/`의 Redis StatefulSet/ConfigMap/PVC 정의가 소비한다.
- Application은 Kubernetes Service DNS `redis.platform.svc.cluster.local:6379`를 사용하며 IP를 하드코딩하지 않는다.
- Secret/Password/Token은 이 문서나 Git에 평문으로 저장하지 않는다.
- NetworkPolicy는 Backend 및 운영 검증 경로가 확정된 후 최소 허용 범위로 적용한다.
- Redis AOF/PVC 복구 검증은 Storage/Recovery 담당과 연계하되 Redis Runtime Manifest 및 Backend 연결은 `seokpan-gitops#7`에서 진행한다.

## 7. 관련 작업

- `seokpan/seokpan-gitops#7`
- `seokpan/seokpan-infra#44`
- `seokpan/seokpan-gitops#10`
- `seokpan/seokpan-gitops#13`
- `seokpan/seokpan-docs#13`

## 8. 외부 기술 근거

- Redis Key Eviction / `maxmemory`: https://redis.io/docs/latest/develop/reference/eviction/
- Redis Administration / Memory: https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
