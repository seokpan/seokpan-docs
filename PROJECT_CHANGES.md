# 石나가는 판단 프로젝트 변경·결정 이력

이 문서는 `seokpan-docs`의 01~08 공식 기획·설계 문서를 기준점으로 하여,
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
  - 사용자 서비스 HTTPS 진입 주소는 `https://game.seokpan.soldesk.store`로 한다.
  - TLS 종료는 NGINX Gateway Fabric에서 수행한다.
  - Gateway 인증서 SAN에는 최소 `game.seokpan.soldesk.store`를 포함한다.
- 관련:
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

---

## 작성 형식

### 변경 또는 결정 제목

- 구분:
- 기존 기준:
- 변경/확정 내용:
- 영향:
- 관련:
