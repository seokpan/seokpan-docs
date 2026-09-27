# 네트워크·Ansible 자동화

[← 전체 트러블슈팅](../README.md) · [이전 번호 대응표](../LEGACY_ID_MAP.md)

네트워크와 공용 Ansible 실행환경·Inventory·공통 변수·통합 자동화 기반 관련 사례를 기록합니다.

등록 문서: **4건**

## 문서 목록

- [NET-001 — Ansible 공용 실행환경이 버전 고정 없이 시스템 기본 환경에 의존](NET-001_Ansible_공용_실행환경_Version_Lock.md)
- [NET-002 — DR-02 격리 etcd 3-member 간 Peer 통신 실패(firewalld 신규 포트 차단)](NET-002_DR-02%20%EA%B2%A9%EB%A6%AC%20etcd%203-member%20%EA%B0%84%20Peer%20%ED%86%B5%EC%8B%A0%20%EC%8B%A4%ED%8C%A8%20%28firewalld%20%EC%8B%A0%EA%B7%9C%20%ED%8F%AC%ED%8A%B8%20%EC%B0%A8%EB%8B%A8%29.md)
- [NET-003 — DR-02 격리 VM 간 9시간 Clock Drift(타임존 UTC 잔존)](NET-003_DR-02%20%EA%B2%A9%EB%A6%AC%20VM%20%EA%B0%84%209%EC%8B%9C%EA%B0%84%20Clock%20Drift%28%ED%83%80%EC%9E%84%EC%A1%B4%20UTC%20%EC%9E%94%EC%A1%B4%29.md)
- [NET-004 — 초기 VM의 DNS Search Domain 설정이 Pod까지 남아 있던 문제](NET-004_초기_VM_DNS_Search_Domain_잔존.md)

## 다른 영역의 관련 사례

- [K8S-010 — cp-03 etcd bbolt DB 손상으로 etcd와 kube-apiserver가 기동하지 못한 문제](../kubernetes-platform-integration/K8S-010_cp03_etcd_bbolt_DB_손상_Control_Plane_복구.md)
- [K8S-012 — Ansible copy 모듈 내장 checksum(SHA-1 고정)과 SHA-256 무결성 검증 요구 충돌](../kubernetes-platform-integration/K8S-012_Ansible%20copy%20%EB%AA%A8%EB%93%88%20%EB%82%B4%EC%9E%A5%20checksum%28SHA-1%20%EA%B3%A0%EC%A0%95%29%EA%B3%BC%20SHA-256%20%EB%AC%B4%EA%B2%B0%EC%84%B1%20%EA%B2%80%EC%A6%9D%20%EC%9A%94%EA%B5%AC%20%EC%B6%A9%EB%8F%8C.md)
- [K8S-013 — Jinja2 JSON 파싱 시 `.items`가 dict.items() 메서드로 오인됨](../kubernetes-platform-integration/K8S-013_Jinja2%EC%97%90%EC%84%9C%20.items%EA%B0%80%20dict%EC%9D%98%20.items%28%29%20%EB%A9%94%EC%84%9C%EB%93%9C%EB%A1%9C%20%EC%98%A4%EC%9D%B8%EB%90%98%EB%8A%94%20%EB%B2%84%EA%B7%B8.md)
- [K8S-014 — etcd-tools 배포 태스크의 delegate_to 미적용으로 3-node 중 일부 노드 배포 누락](../kubernetes-platform-integration/K8S-014_etcd-tools%203-node%20%EB%B0%B0%ED%8F%AC%20%EC%8B%9C%20delegate_to%20%EB%88%84%EB%9D%BD%EC%9C%BC%EB%A1%9C%20%EC%9D%BC%EB%B6%80%20%EB%85%B8%EB%93%9C%20%EB%AF%B8%EB%B0%B0%ED%8F%AC.md)
