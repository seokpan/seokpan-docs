# 데이터베이스·스토리지·백업·복구

[← 전체 트러블슈팅](../README.md)

데이터베이스와 스토리지 운영, 백업·복원, 데이터 보존 및 DR(재해 복구) 관련 사례를 기록합니다.

등록 문서: **15건**

## 문서 목록

- [DB-001 — MariaDB Backup 검증 중 실제 Master 역전과 DCL(계정·권한 변경 SQL) 복제 충돌로 SQL Thread 정지](DB-001_MariaDB_Master_역전_DCL_복제_충돌.md)
- [DB-002 — MariaDB/MaxScale 버전 불일치(Version Drift)와 주 버전 업그레이드(Major Upgrade) 후 시스템 테이블·Replication Channel 문제](DB-002_MariaDB_MaxScale_Version_Drift_Major_Upgrade.md)
- [DB-003 — Auto Failover 환경에서 정적 `mariadb_master` / `mariadb_replica` Inventory가 잘못된 모델이 됨](DB-003_MariaDB_정적_Master_Replica_Inventory_오류.md)
- [DB-004 — NFS Export 자동화에서 Handler 지연과 Ansible Check Mode Skip 때문에 검증이 실제 반영 상태를 보장하지 못할 수 있었음](DB-004_NFS_Export_Handler_Check_Mode_검증_결함.md)
- [DB-005 — MariaDB `read_only` 정적 설정이 자동 장애조치(Auto Failover)와 충돌하고 GTID Domain 설정도 잘못 분리됨](DB-005_MariaDB_read_only_Auto_Failover_GTID_충돌.md)
- [DB-006 — MaxScale TLS 적용 중 인증서 파일 권한과 SAN 검증 문제](DB-006_MaxScale_TLS_인증서_권한_SAN_검증.md)
- [DB-007 — MariaDB/MaxScale 패키지 저장소 버전 확인 오류로 Dry-run 결과가 실제 실행과 달라짐](DB-007_MariaDB_MaxScale_repo_버전_확인_awk_버그.md)
- [DB-008 — MaxScale 설정 배포의 `--check --diff`에서 인증정보 노출 위험](DB-008_MaxScale_check_diff_Credential_노출_방지.md)
- [DB-009 — 공용 TLS Role과 MaxScale Role의 권한 설정 충돌로 인증서 접근 권한이 다시 사라짐](DB-009_MaxScale_TLS_권한_Desired_State_충돌_Drift.md)
- [DB-010 — MariaDB 백업 체인 상태(state)가 호스트 로컬에 종속되어 auto_failover 시 체인을 인식 못 하던 문제](DB-010_MariaDB_백업_체인_상태_NFS_이전_공유_lock.md)
- [DB-011 — 원인 불명의 immutable(chattr +i)로 백업 디렉터리 생성이 EPERM으로 실패](DB-011_backup_transfer_immutable_속성_Guard.md)
- [DB-012 — MariaDB DR 복구 replication_setup의 role Guard 누락으로 master 경로 fatal 실패](DB-012_backup_transfer_replication_setup_role_guard_누락.md)
- [DB-013 — mysqld_exporter slave_status collector의 SLAVE MONITOR 권한 누락으로 Access denied 발생](DB-013_mysqld_exporter_SLAVE_MONITOR_권한_누락.md)
- [DB-014: MariaDB CLI(ad-hoc) 세션의 HAProxy idle timeout 회피 — 버림 쿼리 선행 기법](DB-014_MariaDB%20CLI%28ad-hoc%29%20%EC%84%B8%EC%85%98%EC%9D%98%20HAProxy%20idle%20timeout%20%ED%9A%8C%ED%94%BC%20%E2%80%94%20%EB%B2%84%EB%A6%BC%20%EC%BF%BC%EB%A6%AC%20%EC%84%A0%ED%96%89%20%EA%B8%B0%EB%B2%95.md)
- [DB-015 — Ansible Vault의 DB 비밀번호가 실제 MariaDB·Kubernetes Secret과 달라진 문제](DB-015_Ansible_Vault_DB_Credential_불일치.md)

## 다른 영역의 관련 사례

- [K8S-009 — Kubernetes containerd가 Harbor 내부 CA를 신뢰하지 못해 Image Pull 실패](../kubernetes-platform-integration/K8S-009_Kubernetes_containerd_Harbor_CA_Trust_미적용.md)
- [K8S-010 — cp-03 etcd bbolt DB 손상으로 etcd와 kube-apiserver가 기동하지 못한 문제](../kubernetes-platform-integration/K8S-010_cp03_etcd_bbolt_DB_손상_Control_Plane_복구.md)
- [NET-002 — DR-02 격리 etcd 3-member 간 Peer 통신 실패(firewalld 신규 포트 차단)](../network-ansible-automation/NET-002_DR-02%20%EA%B2%A9%EB%A6%AC%20etcd%203-member%20%EA%B0%84%20Peer%20%ED%86%B5%EC%8B%A0%20%EC%8B%A4%ED%8C%A8%20%28firewalld%20%EC%8B%A0%EA%B7%9C%20%ED%8F%AC%ED%8A%B8%20%EC%B0%A8%EB%8B%A8%29.md)
- [NET-003 — DR-02 격리 VM 간 9시간 Clock Drift(타임존 UTC 잔존)](../network-ansible-automation/NET-003_DR-02%20%EA%B2%A9%EB%A6%AC%20VM%20%EA%B0%84%209%EC%8B%9C%EA%B0%84%20Clock%20Drift%28%ED%83%80%EC%9E%84%EC%A1%B4%20UTC%20%EC%9E%94%EC%A1%B4%29.md)
- [K8S-015 — Kubernetes Secret 확인 중 DB 비밀번호를 복원할 수 있는 값이 출력된 문제](../kubernetes-platform-integration/K8S-015_Kubernetes_Secret_data_출력_DB_Credential_노출_대응.md)
- [NET-004 — 초기 VM의 DNS Search Domain 설정이 Pod까지 남아 있던 문제](../network-ansible-automation/NET-004_초기_VM_DNS_Search_Domain_잔존.md)
