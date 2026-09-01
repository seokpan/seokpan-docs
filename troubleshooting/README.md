# 트러블슈팅 보고서 목차

> 이 디렉토리는 「石나가는 판단」 프로젝트 진행 기간 동안 실제로 발생하거나 검증 과정에서 발견된 트러블슈팅 사례를 누적 관리합니다.  
> 각 사례는 하나의 독립적인 Markdown 보고서로 작성하며, 이 목차에서 해당 보고서로 바로 이동할 수 있습니다.

## 기록 원칙

- 단순 오타나 표현 수정처럼 기술적 판단 가치가 없는 변경은 기록하지 않습니다.
- 실제 장애와 사전 검토에서 발견한 위험을 구분합니다.
- 과거 판단이 후속 검증으로 바뀐 경우 이전 판단을 지우지 않고 변경 과정을 함께 기록합니다.
- 각 보고서는 다른 문서를 읽지 않아도 사건의 배경·영향·원인·조치·검증을 이해할 수 있도록 작성합니다.
- 기술 용어가 정확한 식별에 유리할 때는 쉬운 설명과 기술 용어를 함께 표기하고, 불필요한 내부식 표현은 사용하지 않습니다.
- GitHub Issue, Pull Request, 검토, Commit, 실행 결과는 사실관계를 확인할 수 있는 근거로 연결합니다.
- 새 사례가 생기면 기존 TS 번호를 바꾸지 않고 다음 번호를 추가합니다.

## 목차

1. **[TS-001 — Calico Pod CIDR 후보 설정과 프로젝트 사설망 중첩](TS-001_Calico_Pod_CIDR_사설망_중첩.md)**
2. **[TS-002 — Harbor TLS 인증서 선행조건과 내부 CA 연동](TS-002_Harbor_TLS_인증서_선행조건.md)**
3. **[TS-003 — Snapshot Rollback 이후 SSH 공개키 복구 문제](TS-003_Snapshot_Rollback_SSH_공개키_복구.md)**
4. **[TS-004 — MariaDB Master 역전과 DCL 복제 충돌](TS-004_MariaDB_Master_역전_DCL_복제_충돌.md)**
5. **[TS-005 — Ansible 공용 실행환경 버전 고정과 재현성 문제](TS-005_Ansible_공용_실행환경_Version_Lock.md)**
6. **[TS-006 — NGINX Gateway Fabric 대형 CRD 적용 실패](TS-006_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)**
7. **[TS-007 — Argo CD ApplicationSet CRD 누락과 Controller 장애](TS-007_ArgoCD_ApplicationSet_CRD_누락.md)**
8. **[TS-008 — Harbor Robot API 404와 멱등성 분기 오류](TS-008_Harbor_Robot_API_404_멱등성_오류.md)**
9. **[TS-009 — MariaDB/MaxScale 버전 불일치와 업그레이드 후 복제 문제](TS-009_MariaDB_MaxScale_Version_Drift_Major_Upgrade.md)**
10. **[TS-010 — MariaDB 정적 Master/Replica Inventory 모델 오류](TS-010_MariaDB_정적_Master_Replica_Inventory_오류.md)**
11. **[TS-011 — GitOps Namespace·API·Network·JCasC 기준 불일치](TS-011_GitOps_Namespace_API_Network_JCasC_불일치.md)**
12. **[TS-012 — Harbor VM 재부팅 후 컨테이너 자동기동 실패](TS-012_Harbor_VM_재부팅_컨테이너_자동기동_실패.md)**
13. **[TS-013 — Kubernetes Check Mode 검증 Skip 문제](TS-013_Kubernetes_Check_Mode_검증_Skip.md)**
14. **[TS-014 — NFS Provisioner RBAC 불필요 Node 권한 제거](TS-014_NFS_Provisioner_RBAC_불필요_Node_권한.md)**
15. **[TS-015 — NFS Export Handler·Check Mode 검증 결함](TS-015_NFS_Export_Handler_Check_Mode_검증_결함.md)**
16. **[TS-016 — GitOps Root 전환 중 Argo CD Bootstrap 연쇄 오류](TS-016_ArgoCD_Root_전환_kubeconfig_tmp_충돌.md)**
17. **[TS-017 — MariaDB read_only·Auto Failover·GTID 충돌](TS-017_MariaDB_read_only_Auto_Failover_GTID_충돌.md)**
18. **[TS-018 — BuildKit Harbor CA 연동 중 kubeconfig 경로·권한 문제](TS-018_BuildKit_Harbor_CA_kubeconfig_경로_권한.md)**
19. **[TS-019 — 빈 환경 Smoke에서 Network/LB 자동화 결함 발견](TS-019_빈_Project_Smoke_Network_LB_자동화_결함.md)**
20. **[TS-020 — Frontend Node.js 버전 실행환경 검증 결함](TS-020_Frontend_Nodejs_Version_실행환경_검증_결함.md)**
21. **[TS-021 — Harbor 비밀정보 평문 노출과 자격증명 교체 필요](TS-021_Harbor_비밀정보_평문_노출_교체.md)**

---

현재 등록된 트러블슈팅 보고서: **21건**  
기준 시점: **2026-09-01**
