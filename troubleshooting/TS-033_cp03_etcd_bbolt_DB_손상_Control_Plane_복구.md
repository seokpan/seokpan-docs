[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-033 — cp-03 etcd bbolt DB 손상으로 etcd와 kube-apiserver가 기동하지 못한 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하고 복구까지 완료한 장애를 기록합니다. 당시 실행 결과로 확인한 사실과 확인하지 못한 원인을 구분해 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결 / DB 손상을 일으킨 원인은 미확정** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | `cp-03` etcd, kube-apiserver, Kubernetes Control Plane 1대 |

## 문제 개요

`cp-03`이 `NotReady` 상태로 전환된 뒤 Kubernetes에서 해당 노드의 상태를 더 이상 정상적으로 갱신하지 못했다.

확인된 주요 시점은 다음과 같다.

```text
07:37경   cp-03 마지막 Node Heartbeat
07:40경   cp-03 Lease 마지막 갱신
07:41경   NodeStatusUnknown 전환
09:27경   cp-03 재부팅
```

재부팅 후 `kubelet`과 `containerd` 서비스 자체는 실행 중이었지만, etcd와 kube-apiserver는 반복적으로 종료됐다. `2379`, `2380`, `6443` 포트도 열리지 않았다.

장애 조사 시점의 `crictl` 상태에서 etcd는 attempt 111, kube-apiserver는 attempt 115까지 재시작이 반복된 상태였다.

## 직접 원인

etcd가 로컬 데이터를 저장하는 bbolt DB(`/var/lib/etcd/member/snap/db`)를 여는 과정에서 오류가 발생하며 프로세스가 panic하는 것을 확인했다.

```text
panic: freepages: failed to get all reachable pages
(page 1794: multiple references ...)
```

즉 cp-03의 etcd가 로컬 DB를 정상적으로 열지 못했고 시작 직후 종료됐다.

그 결과 cp-03의 kube-apiserver도 다음 로컬 etcd endpoint에 연결할 수 없었다.

```text
https://127.0.0.1:2379
```

kube-apiserver 로그에는 반복적인 `connection refused` 이후 다음 오류가 기록됐다.

```text
error creating storage factory: context deadline exceeded
```

따라서 이번 장애에서 확인된 직접 원인은 다음과 같다.

```text
cp-03 bbolt DB 손상
→ etcd 시작 실패
→ 127.0.0.1:2379 미기동
→ kube-apiserver가 로컬 etcd에 연결하지 못함
→ cp-03 Control Plane 복귀 실패
```

## DB 손상을 일으킨 원인은 미확정

bbolt DB가 손상돼 etcd가 기동하지 못한 사실은 확인했지만, **왜 DB가 손상됐는지는 확인하지 못했다.**

부팅 이력에는 비정상 종료로 보이는 `crash` 기록이 있었지만, 이를 DB 손상의 직접 원인으로 증명할 Host·VMware·Storage I/O 자료는 확보하지 못했다.

따라서 비정상 종료, Host 문제, VMware 문제, Storage I/O 문제 등은 가능한 원인 후보일 뿐 이번 문서에서 원인으로 확정하지 않는다.

## 장애 조사 시 확인한 Control Plane 가용성

cp-03가 `NotReady`인 상태에서 조사했을 때 cp-01과 cp-02의 etcd endpoint는 정상 응답했고 cp-03 endpoint만 실패했다.

```text
https://192.168.51.20:2379   healthy=true
https://192.168.52.20:2379   healthy=true
https://192.168.53.20:2379   healthy=false
```

이 시점의 etcd membership은 3개였으며 cp-01과 cp-02가 정상이라 quorum 2/3을 유지하고 있었다.

같은 조사 시점에 Kubernetes API VIP도 정상 응답했다.

```text
https://10.1.93.90:6443/livez
→ HTTP 200
```

따라서 **Control Plane 1대가 이탈한 조사 시점에도 cp-01·cp-02의 etcd quorum과 Kubernetes API 가용성이 유지되고 있음을 확인했다.**

단, 07:37경 최초 장애부터 복구 완료까지 API가 한 번도 중단되지 않았다는 연속 가용성까지 측정한 것은 아니다.

## 복구 전 안전조치

손상된 member를 수정하기 전에 정상 cp-01 etcd에서 snapshot을 확보했다.

```text
Revision    4,394,087
Total Keys  1,017
Total Size  31 MB
SHA256      a480316bb1e5d839c4a45d7521c9bec9c98b96c43d1b3b9f8cbb5d3ce62f3fca
```

cp-03에서는 다음 자료를 별도로 보존했다.

- etcd static Pod manifest
- 손상된 `/var/lib/etcd/member/snap/db`의 SHA256
- etcd panic 로그
- 기존 `/var/lib/etcd` 전체 데이터

복구가 실패해도 원본을 다시 확인할 수 있도록 기존 데이터를 삭제하지 않고 별도 경로에 보관한 뒤 작업했다.

## 1차 복구 시도 — snapshot 디렉터리만 분리

처음에는 손상된 `member/snap` 디렉터리만 분리하고 기존 WAL(Write-Ahead Log, 쓰기 내용을 먼저 기록하는 로그 파일)은 유지한 채 etcd를 다시 시작했다.

새 `snap/db` 자체는 정상적으로 생성됐지만 snapshot index가 `0`인 상태에서 기존 WAL과 이어지지 못했다.

```text
No snapshot found. Recovering WAL from scratch!
...
failed to open WAL
wal: file not found which matches the snapshot index '0'
```

따라서 손상된 snapshot DB만 제거하고 기존 WAL을 재사용하는 방식으로는 cp-03 member를 복구할 수 없었다.

## 2차 복구 — 손상된 etcd member 교체

1차 시도가 실패한 뒤 cp-03의 기존 etcd member를 제거하고 새 member로 다시 등록했다.

기존 member:

```text
8c78341873425b06
```

신규 member:

```text
69454523511ca6dc
```

복구 순서는 다음과 같다.

```text
cp-03 etcd static Pod 중지
→ 기존 /var/lib/etcd 보존
→ 빈 /var/lib/etcd 생성
→ 기존 cp-03 etcd member 제거
→ 같은 peer URL(192.168.53.20:2380)로 cp-03 새 member 등록
→ etcd static Pod 다시 시작
```

기존 cp-03 member를 제거한 직후 `kubectl exec`로 member 목록을 다시 확인하는 과정에서 `TLS handshake timeout`이 한 차례 발생했다. 이때는 즉시 다음 membership 변경으로 진행하지 않았다.

cp-01과 cp-02를 각각 직접 확인한 결과:

```text
cp-01 local API /livez   HTTP 200
API VIP /livez           HTTP 200
cp-01 etcd endpoint      healthy=true
cp-02 etcd endpoint      healthy=true
cp-02 local API /livez   HTTP 200
```

두 정상 member와 API가 동작하고 있음을 확인한 뒤 cp-03 새 member를 추가했다. 해당 `TLS handshake timeout`의 원인은 별도로 확정하지 않았으며, 이후 동일 확인 과정에서는 재발하지 않았다.

새 cp-03 member가 시작된 뒤 당시 leader였던 cp-02에서 snapshot을 전달받았다.

```text
receiving database snapshot
incoming-snapshot-index: 6208598
...
received and saved database snapshot
...
restored snapshot [index: 6208598, term: 29]
```

초기 member 정보 게시 과정에서 7초 timeout이 한 차례 기록됐지만 바로 다음 시도에서 정상적으로 게시됐고, 이어서 client request를 제공할 수 있는 상태로 전환됐다.

```text
failed to publish local member to cluster through raft
...
published local member to cluster through raft
ready to serve client requests
grpc service status changed ... status="SERVING"
```

## 복구 검증

### etcd 3개 member 상태

복구 후 세 endpoint가 모두 healthy 상태인지 확인했다.

```text
192.168.51.20:2379   true
192.168.52.20:2379   true
192.168.53.20:2379   true
```

새 cp-03 member의 Raft 합의 상태도 정상 member와 같은 term으로 수렴했고, 확인 시점의 index 차이는 2였다.

```text
cp-01  term 29 / index 6209122
cp-03  term 29 / index 6209122
cp-02  term 29 / index 6209124 / leader
```

### kube-apiserver 복구

etcd가 정상화된 뒤 cp-03 kube-apiserver가 자동으로 다시 시작됐다.

```text
Serving securely on [::]:6443
Storage is ready for all registered resources
```

로컬 `/livez`도 정상 응답했다.

```text
https://127.0.0.1:6443/livez
→ ok
```

### Node와 네트워크 복구

최종 확인에서 cp-03는 다시 `Ready` 상태가 됐다.

```text
NetworkUnavailable=False  reason=CalicoIsUp
MemoryPressure=False
DiskPressure=False
PIDPressure=False
Ready=True                reason=KubeletReady
```

Node Lease도 현재 시각으로 다시 갱신됐고, cp-03의 다음 구성요소가 모두 `Running` 상태로 확인됐다.

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kube-proxy
- calico-node
- Calico CSI
- node-exporter

Kubernetes Service Network도 다시 정상 응답했다.

```text
https://10.96.0.1:443/livez
→ ok
```

마지막 확인에서 최근 2분의 kubelet journal을 조회했을 때 추가 로그 항목이 없었다(`-- No entries --`).

## Before → Change → After

```text
Before
cp-03 etcd가 손상된 bbolt DB를 열지 못하고 panic
→ 로컬 etcd 127.0.0.1:2379 미기동
→ kube-apiserver 시작 실패
→ cp-03 NotReady

Change
정상 etcd snapshot 확보 + 손상 자료 보존
→ snapshot 디렉터리만 분리한 1차 복구는 WAL 불연속으로 실패
→ 기존 cp-03 etcd member 제거
→ 빈 data-dir로 새 member 등록
→ 정상 member에서 snapshot 재동기화

After
etcd 3개 endpoint 모두 healthy
→ cp-03 kube-apiserver 자동 복구
→ kubelet·kube-proxy·Calico 재수렴
→ cp-03 Ready
→ Kubernetes Service Network /livez 정상
```

## 시간 기록

이번 사건은 장애가 발생한 즉시 복구 작업을 시작한 것이 아니므로, 최초 Heartbeat 중단부터 Ready 복귀까지의 전체 시간을 그대로 MTTR로 사용하지 않는다.

확인 가능한 주요 시점은 다음과 같다.

```text
07:37경   마지막 Node Heartbeat
07:41경   NodeStatusUnknown
14:20대   실제 조사 시작
14:44:52  cp-03 etcd가 client request 제공 시작
14:49경   kube-apiserver 정상 서비스 시작
14:53경   cp-03 Ready=True 확인
```

향후 MTTR을 비교 지표로 사용할 경우 장애 탐지 시각, 대응 시작 시각, etcd 복구 시각, API 복구 시각, Node Ready 복귀 시각을 따로 기록한다.

## 관련 근거

- Docs Issue #95 — 사후 장애 기록 및 TS 작성: https://github.com/seokpan/seokpan-docs/issues/95
- 정상 etcd snapshot SHA256: `a480316bb1e5d839c4a45d7521c9bec9c98b96c43d1b3b9f8cbb5d3ce62f3fca`
- 손상된 cp-03 bbolt DB SHA256: `314ed8a2612f2ba19f34836f60ecefdc558d01805b06eec5e31520116aaede00`

## 남은 확인 사항

이번 복구로 cp-03의 etcd·kube-apiserver·Node 상태는 정상화됐지만, bbolt DB 손상을 일으킨 원인은 확인하지 못했다.

동일 문제가 다시 발생하면 Host/VMware 전원 이벤트, Storage I/O 오류, filesystem 상태와 비정상 종료 이력을 함께 확보해 DB 손상 이전 단계의 원인을 추적한다.
