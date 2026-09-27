# CI/CD·관측

[← 전체 트러블슈팅](../README.md) · [이전 번호 대응표](../LEGACY_ID_MAP.md)

Registry·빌드·배포와 로그·지표 등 관측 경로 관련 사례를 기록합니다.

등록 문서: **14건**

## 문서 목록

- [OPS-001 — Harbor 설치 자동화가 TLS 인증서 부재에서 중단된 뒤 내부 CA/TLS 체계로 연결](OPS-001_Harbor_TLS_인증서_선행조건.md)
- [OPS-002 — Harbor Robot Account 자동화에서 잘못된 API 경로 404와 멱등성 분기 오류가 연쇄적으로 발견됨](OPS-002_Harbor_Robot_API_404_멱등성_오류.md)
- [OPS-003 — Harbor VM 재부팅 후 대부분의 컨테이너가 자동 기동하지 않음](OPS-003_Harbor_VM_재부팅_컨테이너_자동기동_실패.md)
- [OPS-004 — Frontend가 잘못된 Node.js 버전에서도 `npm ci`를 통과하던 실행환경 검증 결함](OPS-004_Frontend_Nodejs_Version_실행환경_검증_결함.md)
- [OPS-005 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패](OPS-005_buildkit-harbor-ca-trust.md)
- [OPS-006 — BuildKit이 Harbor Robot 인증 파일을 찾지 못해 401 Unauthorized 발생](OPS-006_BuildKit_Harbor_Credential_파일_계약_불일치.md)
- [OPS-007 — Rootless BuildKit이 Dockerfile 첫 RUN에서 /proc 권한 문제로 실패](OPS-007_BuildKit_Rootless_nested_RUN_seccomp_실행_실패.md)
- [OPS-008 — Harbor Tag Immutability 미적용으로 동일 Tag 덮어쓰기가 허용됨](OPS-008_Harbor_Tag_Immutability_미적용_동일_Tag_덮어쓰기.md)
- [OPS-009 — Dockerfile syntax directive가 외부 frontend를 사용하게 해 Rootless BuildKit Build가 중단됨](OPS-009_BuildKit_Dockerfile_syntax_frontend_재위임_실패.md)
- [OPS-010 — Loki Deployment가 프로젝트에서 정의한 PVC와 다른 PVC를 참조함](OPS-010_Loki_PVC_Desired_State_불일치.md)
- [OPS-011 — Loki compactor의 `delete_request_store` 누락으로 CONFIG ERROR 발생](OPS-011_Loki_compactor_CONFIG_ERROR.md)
- [OPS-012: Harbor Tag Immutability에서 Candidate Tag(scan-*) 예외 정책이 두 차례 무력화된 문제](OPS-012_Harbor_Tag_Immutability-Candidate_Tag_예외정책_무력화_issue.md)
- [OPS-013 — Jenkins Plugin Lock 파일 확장자(.lock)를 jenkins-plugin-cli가 인식하지 못해 Controller CrashLoopBackOff 발생](OPS-013_Jenkins_Plugin_Lock_파일_확장자를_jenkins-plugin-cli가_인식하지_못해_ControllerCrashLoopBackOff_발생.md)
- [OPS-014 — 원 계획의 maxscale_exporter(공식 REST API exporter)가 존재하지 않아 소스 빌드 방식으로 2차 이관](OPS-014_maxscale_exporter%28%EA%B3%B5%EC%8B%9D%20REST%20API%20exporter%29%EA%B0%80%20%EC%A1%B4%EC%9E%AC%ED%95%98%EC%A7%80%20%EC%95%8A%EC%95%84%20%EC%86%8C%EC%8A%A4%20%EB%B9%8C%EB%93%9C%20%EB%B0%A9%EC%8B%9D%EC%9C%BC%EB%A1%9C%202%EC%B0%A8%20%EC%9D%B4%EA%B4%80.md)

## 다른 영역의 관련 사례

- [DB-006 — MaxScale TLS 적용 중 인증서 파일 권한과 SAN 검증 문제](../database-storage-recovery/DB-006_MaxScale_TLS_인증서_권한_SAN_검증.md)
- [K8S-007 — Kubernetes Pod에서 프로젝트 도메인이 CoreDNS SERVFAIL로 조회 실패](../kubernetes-platform-integration/K8S-007_CoreDNS_Project_Endpoint_SERVFAIL.md)
- [K8S-008 — CoreDNS 설정 줄바꿈 오류로 신규 Pod가 CrashLoopBackOff 발생](../kubernetes-platform-integration/K8S-008_CoreDNS_관리_Block_newline_escaping_CrashLoop.md)
- [K8S-009 — Kubernetes containerd가 Harbor 내부 CA를 신뢰하지 못해 Image Pull 실패](../kubernetes-platform-integration/K8S-009_Kubernetes_containerd_Harbor_CA_Trust_미적용.md)
- [DB-013 — mysqld_exporter slave_status collector의 SLAVE MONITOR 권한 누락으로 Access denied 발생](../database-storage-recovery/DB-013_mysqld_exporter_SLAVE_MONITOR_권한_누락.md)
