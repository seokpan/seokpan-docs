[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-041 — etcd-tools 배포 태스크의 delegate_to 미적용으로 3-node 중 일부 노드 배포 누락

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-15 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 자동화 Snapshot 준비 단계, 운영 서버 영향 없음 |

## 최초 문제
격리 3-node(loadgen/loadgen2/loadgen3) 전체에 etcd-tools를 배포하는
태스크를 실행했으나, 일부 노드에서만 실제로 배포되고 나머지 노드는
누락되는 현상이 발견됐다.

## 원인
배포 태스크가 대상 호스트 반복(loop) 구조에서 `delegate_to`를 명시하지
않아, 의도한 각 노드가 아니라 실행 호스트 기준으로만 처리되는 구간이
있었다.

## 조치
`delegate_to`와 `with_nested`를 함께 적용해 3개 노드 전체에 대해
개별적으로 배포가 수행되도록 태스크 구조를 수정했다.

## 검증
- 수정 후 3-node 전체(loadgen/loadgen2/loadgen3)에서 etcd-tools 배포
  및 버전 확인 STEP이 PASS(PR #191 E2E STEP 1 결과)

## 관련 근거
- Issue #176: https://github.com/seokpan/seokpan-infra/issues/176
- PR #191: https://github.com/seokpan/seokpan-infra/pull/191
