# 트러블슈팅 보고서 목차

> 이 디렉토리는 「石나가는 판단」 프로젝트 진행 중 발생한 문제 가운데 **원인·조치·검증까지 완료된 사례**를 공식 트러블슈팅 보고서로 관리합니다.  
> 각 사례는 하나의 독립적인 Markdown 보고서로 작성하며, 이 목차에서 해당 보고서로 바로 이동할 수 있습니다.

## 기록 원칙

- 실제 장애 또는 검토 과정에서 확인된 기술적 결함 중 **해결 완료가 검증된 사례만 게시**합니다.
- 진행 중인 문제는 Issue·Pull Request에서 추적하고, 해결과 재검증이 완료된 뒤 보고서로 게시합니다.
- 단순 오타나 표현 수정처럼 기술적 판단 가치가 없는 변경은 기록하지 않습니다.
- 실제 장애와 사전 검토에서 발견해 수정한 예방형 문제를 구분합니다.
- 판단이 추가 검증으로 바뀐 경우 변경 과정을 지우지 않고, 날짜·실제 작업·근거를 명시하여 기록합니다.
- 각 보고서는 다른 문서나 대화를 읽지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있어야 합니다.
- 기술 용어가 정확한 식별에 유리할 때는 쉬운 설명과 기술 용어를 함께 표기하고, 불필요한 내부식 표현은 사용하지 않습니다.
- GitHub Issue, Pull Request, 검토, Commit, 실제 실행 결과를 사실관계 확인 근거로 연결합니다.
- TS-001~TS-016 재번호는 초기 공개 목록 자체를 교정하기 위한 예외 작업이었습니다. 이후에는 이미 게시된 TS 번호를 유지하고, 새로 해결 완료된 사례를 다음 번호부터 순차적으로 추가합니다.

## 목차

1. **[TS-001 — Calico Pod CIDR 후보 설정과 프로젝트 사설망 중첩](TS-001_Calico_Pod_CIDR_사설망_중첩.md)**
2. **[TS-002 — Harbor TLS 인증서 선행조건과 내부 CA 연동](TS-002_Harbor_TLS_인증서_선행조건.md)**
3. **[TS-003 — MariaDB Master 역전과 DCL 복제 충돌](TS-003_MariaDB_Master_역전_DCL_복제_충돌.md)**
4. **[TS-004 — Ansible 공용 실행환경 버전 고정](TS-004_Ansible_공용_실행환경_Version_Lock.md)**
5. **[TS-005 — NGINX Gateway Fabric 대형 CRD 적용 실패](TS-005_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)**
6. **[TS-006 — Argo CD ApplicationSet CRD 누락과 Controller 장애](TS-006_ArgoCD_ApplicationSet_CRD_누락.md)**
7. **[TS-007 — Harbor Robot API 404와 멱등성 분기 오류](TS-007_Harbor_Robot_API_404_멱등성_오류.md)**
8. **[TS-008 — MariaDB/MaxScale 버전 불일치와 업그레이드 후 복제 문제](TS-008_MariaDB_MaxScale_Version_Drift_Major_Upgrade.md)**
9. **[TS-009 — MariaDB 정적 Master/Replica Inventory 모델 오류](TS-009_MariaDB_정적_Master_Replica_Inventory_오류.md)**
10. **[TS-010 — Harbor VM 재부팅 후 컨테이너 자동기동 실패](TS-010_Harbor_VM_재부팅_컨테이너_자동기동_실패.md)**
11. **[TS-011 — Kubernetes Check Mode 검증 Skip 문제](TS-011_Kubernetes_Check_Mode_검증_Skip.md)**
12. **[TS-012 — NFS Provisioner RBAC 불필요 Node 권한 제거](TS-012_NFS_Provisioner_RBAC_불필요_Node_권한.md)**
13. **[TS-013 — NFS Export Handler·Check Mode 검증 결함](TS-013_NFS_Export_Handler_Check_Mode_검증_결함.md)**
14. **[TS-014 — GitOps Root 전환 중 Argo CD Bootstrap 연쇄 오류](TS-014_ArgoCD_Root_전환_kubeconfig_tmp_충돌.md)**
15. **[TS-015 — MariaDB read_only·Auto Failover·GTID 충돌](TS-015_MariaDB_read_only_Auto_Failover_GTID_충돌.md)**
16. **[TS-016 — Frontend Node.js 버전 실행환경 검증 결함](TS-016_Frontend_Nodejs_Version_실행환경_검증_결함.md)**

---

현재 게시된 해결 완료 트러블슈팅 보고서: **16건**  
기준 시점: **2026-09-01**
