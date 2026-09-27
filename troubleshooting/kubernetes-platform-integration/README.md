# Kubernetes·플랫폼 통합

[← 전체 트러블슈팅](../README.md) · [이전 번호 대응표](../LEGACY_ID_MAP.md)

Kubernetes 클러스터와 애플리케이션 런타임의 플랫폼 통합 관련 사례를 기록합니다.

등록 문서: **16건**

## 문서 목록

- [K8S-001 — Calico Pod CIDR 후보 설정이 프로젝트 사설망과 중첩](K8S-001_Calico_Pod_CIDR_사설망_중첩.md)
- [K8S-002 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림](K8S-002_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)
- [K8S-003 — Argo CD ApplicationSet CRD 누락으로 Controller가 장시간 CrashLoopBackOff](K8S-003_ArgoCD_ApplicationSet_CRD_누락.md)
- [K8S-004 — Kubernetes 통합 Playbook의 Ansible Check Mode에서 검증 명령이 Skip되어 Assert가 실패](K8S-004_Kubernetes_Check_Mode_검증_Skip.md)
- [K8S-005 — NFS Provisioner RBAC 검토에서 초기 판단을 재검증해 불필요한 Node 조회 권한을 제거](K8S-005_NFS_Provisioner_RBAC_불필요_Node_권한.md)
- [K8S-006 — GitOps Root Application 경로 전환 중 Argo CD 초기 구성 자동화(Bootstrap) 오류가 연쇄적으로 발생](K8S-006_ArgoCD_Root_전환_kubeconfig_tmp_충돌.md)
- [K8S-007 — Kubernetes Pod에서 프로젝트 도메인이 CoreDNS SERVFAIL로 조회 실패](K8S-007_CoreDNS_Project_Endpoint_SERVFAIL.md)
- [K8S-008 — CoreDNS 설정 줄바꿈 오류로 신규 Pod가 CrashLoopBackOff 발생](K8S-008_CoreDNS_관리_Block_newline_escaping_CrashLoop.md)
- [K8S-009 — Kubernetes containerd가 Harbor 내부 CA를 신뢰하지 못해 Image Pull 실패](K8S-009_Kubernetes_containerd_Harbor_CA_Trust_미적용.md)
- [K8S-010 — cp-03 etcd bbolt DB 손상으로 etcd와 kube-apiserver가 기동하지 못한 문제](K8S-010_cp03_etcd_bbolt_DB_손상_Control_Plane_복구.md)
- [K8S-011 — kube-apiserver AnonymousAuth·AlwaysAllow 조합 시 전체 401 Unauthorized](K8S-011_kube-apiserver%20AnonymousAuth%C2%B7AlwaysAllow%20%EC%A1%B0%ED%95%A9%20%EC%8B%9C%20%EC%A0%84%EC%B2%B4%20401%20Unauthorized.md)
- [K8S-012 — Ansible copy 모듈 내장 checksum(SHA-1 고정)과 SHA-256 무결성 검증 요구 충돌](K8S-012_Ansible%20copy%20%EB%AA%A8%EB%93%88%20%EB%82%B4%EC%9E%A5%20checksum%28SHA-1%20%EA%B3%A0%EC%A0%95%29%EA%B3%BC%20SHA-256%20%EB%AC%B4%EA%B2%B0%EC%84%B1%20%EA%B2%80%EC%A6%9D%20%EC%9A%94%EA%B5%AC%20%EC%B6%A9%EB%8F%8C.md)
- [K8S-013 — Jinja2 JSON 파싱 시 `.items`가 dict.items() 메서드로 오인됨](K8S-013_Jinja2%EC%97%90%EC%84%9C%20.items%EA%B0%80%20dict%EC%9D%98%20.items%28%29%20%EB%A9%94%EC%84%9C%EB%93%9C%EB%A1%9C%20%EC%98%A4%EC%9D%B8%EB%90%98%EB%8A%94%20%EB%B2%84%EA%B7%B8.md)
- [K8S-014 — etcd-tools 배포 태스크의 delegate_to 미적용으로 3-node 중 일부 노드 배포 누락](K8S-014_etcd-tools%203-node%20%EB%B0%B0%ED%8F%AC%20%EC%8B%9C%20delegate_to%20%EB%88%84%EB%9D%BD%EC%9C%BC%EB%A1%9C%20%EC%9D%BC%EB%B6%80%20%EB%85%B8%EB%93%9C%20%EB%AF%B8%EB%B0%B0%ED%8F%AC.md)
- [K8S-015 — Kubernetes Secret 확인 중 DB 비밀번호를 복원할 수 있는 값이 출력된 문제](K8S-015_Kubernetes_Secret_data_출력_DB_Credential_노출_대응.md)
- [K8S-016 — Backend 2 Replica 강제 분산과 RollingUpdate가 충돌해 Rollout이 교착된 문제](K8S-016_Backend_강제분산_RollingUpdate_Surge_교착.md)

## 다른 영역의 관련 사례

- [OPS-005 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패](../cicd-observability/OPS-005_buildkit-harbor-ca-trust.md)
- [OPS-006 — BuildKit이 Harbor Robot 인증 파일을 찾지 못해 401 Unauthorized 발생](../cicd-observability/OPS-006_BuildKit_Harbor_Credential_파일_계약_불일치.md)
- [OPS-007 — Rootless BuildKit이 Dockerfile 첫 RUN에서 /proc 권한 문제로 실패](../cicd-observability/OPS-007_BuildKit_Rootless_nested_RUN_seccomp_실행_실패.md)
- [NET-002 — DR-02 격리 etcd 3-member 간 Peer 통신 실패(firewalld 신규 포트 차단)](../network-ansible-automation/NET-002_DR-02%20%EA%B2%A9%EB%A6%AC%20etcd%203-member%20%EA%B0%84%20Peer%20%ED%86%B5%EC%8B%A0%20%EC%8B%A4%ED%8C%A8%20%28firewalld%20%EC%8B%A0%EA%B7%9C%20%ED%8F%AC%ED%8A%B8%20%EC%B0%A8%EB%8B%A8%29.md)
- [NET-003 — DR-02 격리 VM 간 9시간 Clock Drift(타임존 UTC 잔존)](../network-ansible-automation/NET-003_DR-02%20%EA%B2%A9%EB%A6%AC%20VM%20%EA%B0%84%209%EC%8B%9C%EA%B0%84%20Clock%20Drift%28%ED%83%80%EC%9E%84%EC%A1%B4%20UTC%20%EC%9E%94%EC%A1%B4%29.md)
- [DB-013 — mysqld_exporter slave_status collector의 SLAVE MONITOR 권한 누락으로 Access denied 발생](../database-storage-recovery/DB-013_mysqld_exporter_SLAVE_MONITOR_권한_누락.md)
- [DB-015 — Ansible Vault의 DB 비밀번호가 실제 MariaDB·Kubernetes Secret과 달라진 문제](../database-storage-recovery/DB-015_Ansible_Vault_DB_Credential_불일치.md)
