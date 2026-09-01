# 트러블슈팅 보고서 목차

> 이 디렉토리는 「石나가는 판단」 프로젝트 진행 기간 동안 실제로 발생하거나 검증 과정에서 발견된 트러블슈팅 사례를 누적 관리합니다.  
> 각 사례는 하나의 독립적인 Markdown 보고서로 작성하며, 이 목차에서 해당 보고서로 바로 이동할 수 있습니다.

## 기록 원칙

- 단순 오타나 표현 수정처럼 기술적 판단 가치가 없는 변경은 기록하지 않습니다.
- 실제 장애와 사전 검토에서 발견한 위험을 구분합니다.
- 과거 판단이 후속 검증으로 바뀐 경우, 이전 판단을 지우지 않고 변경 과정을 함께 기록합니다.
- 각 보고서는 다른 문서를 읽지 않아도 사건의 배경·원인·조치·검증을 이해할 수 있도록 작성합니다.
- GitHub Issue, Pull Request, Review, Commit, 실행 결과는 사실관계를 확인할 수 있는 근거로 연결합니다.
- 새 사례가 생기면 기존 TS 번호를 바꾸지 않고 다음 번호를 추가합니다.

## 목차

1. **[TS-001 — Calico Pod CIDR 후보 설정이 프로젝트 사설망과 중첩](TS-001_Calico_Pod_CIDR_사설망_중첩.md)**
2. **[TS-002 — Harbor 설치 자동화가 TLS 인증서 부재에서 중단된 뒤 내부 CA/TLS 체계로 연결](TS-002_Harbor_TLS_인증서_선행조건.md)**
3. **[TS-003 — Snapshot Rollback 이후 SSH 공개키가 과거 상태로 돌아갈 수 있는 문제](TS-003_Snapshot_Rollback_SSH_공개키_복구.md)**
4. **[TS-004 — MariaDB Backup 검증 중 실제 Master 역전과 DCL 복제 충돌로 SQL Thread 정지](TS-004_MariaDB_Master_역전_DCL_복제_충돌.md)**
5. **[TS-005 — Ansible 공용 실행환경이 버전 고정 없이 System 환경에 의존](TS-005_Ansible_공용_실행환경_Version_Lock.md)**
6. **[TS-006 — NGINX Gateway Fabric 대형 CRD가 client-side apply Annotation 제한에 걸림](TS-006_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)**
7. **[TS-007 — Argo CD ApplicationSet CRD 누락으로 Controller가 장시간 CrashLoopBackOff](TS-007_ArgoCD_ApplicationSet_CRD_누락.md)**
8. **[TS-008 — Harbor Robot Account 자동화에서 잘못된 API 경로 404와 멱등성 분기 오류가 연쇄적으로 발견됨](TS-008_Harbor_Robot_API_404_멱등성_오류.md)**
9. **[TS-009 — MariaDB/MaxScale Version Drift와 Major Upgrade 후 시스템 테이블·Replication Channel 문제](TS-009_MariaDB_MaxScale_Version_Drift_Major_Upgrade.md)**
10. **[TS-010 — Auto Failover 환경에서 정적 `mariadb_master` / `mariadb_replica` Inventory가 잘못된 모델이 됨](TS-010_MariaDB_정적_Master_Replica_Inventory_오류.md)**
11. **[TS-011 — GitOps 초안이 실제 Namespace·API·Network·JCasC 기준과 여러 지점에서 어긋남](TS-011_GitOps_Namespace_API_Network_JCasC_불일치.md)**
12. **[TS-012 — Harbor VM 재부팅 후 대부분의 컨테이너가 자동 기동하지 않음](TS-012_Harbor_VM_재부팅_컨테이너_자동기동_실패.md)**
13. **[TS-013 — Kubernetes 통합 Playbook의 Check Mode에서 검증 명령이 Skip되어 Assert가 실패](TS-013_Kubernetes_Check_Mode_검증_Skip.md)**
14. **[TS-014 — NFS Provisioner RBAC 검토에서 초기 판단을 재검증해 불필요한 Node 조회 권한을 제거](TS-014_NFS_Provisioner_RBAC_불필요_Node_권한.md)**
15. **[TS-015 — NFS Export 자동화에서 Handler 지연과 Check Mode Skip 때문에 검증이 실제 반영 상태를 보장하지 못할 수 있었음](TS-015_NFS_Export_Handler_Check_Mode_검증_결함.md)**
16. **[TS-016 — GitOps Root Application 경로 전환 과정에서 Argo CD Bootstrap 오류가 연쇄적으로 발생](TS-016_ArgoCD_Root_전환_kubeconfig_tmp_충돌.md)**
17. **[TS-017 — MariaDB `read_only` 정적 설정이 Auto Failover와 충돌하고 GTID Domain 설정도 잘못 분리됨](TS-017_MariaDB_read_only_Auto_Failover_GTID_충돌.md)**
18. **[TS-018 — BuildKit Harbor CA Trust 연결 중 Kubernetes 관리자 kubeconfig의 경로·소유 방식이 문제로 드러남](TS-018_BuildKit_Harbor_CA_kubeconfig_경로_권한.md)**
19. **[TS-019 — 빈 Project 재현성 Smoke에서 Network/LB 자동화의 NIC Fact 구조 가정과 Check Mode 검증 결함이 드러남](TS-019_빈_Project_Smoke_Network_LB_자동화_결함.md)**
20. **[TS-020 — Application Frontend의 Node.js Version 계약이 `npm ci`에서 경고만 내고 통과하던 실행 Gate 결함](TS-020_Frontend_Nodejs_Version_실행환경_검증_결함.md)**
21. **[TS-021 — Harbor 관리용 비밀정보가 평문으로 노출된 사실이 확인되어 자격증명 교체가 필요한 보안 후속 문제](TS-021_Harbor_비밀정보_평문_노출_교체.md)**

---

현재 등록된 트러블슈팅 보고서: **21건**  
기준 시점: **2026-09-01**
