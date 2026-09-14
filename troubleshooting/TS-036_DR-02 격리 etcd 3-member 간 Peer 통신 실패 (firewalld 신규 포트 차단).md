[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-036 — DR-02 격리 etcd 3-member 간 Peer 통신 실패(firewalld 신규 포트 차단)

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 격리 검증 환경(loadgen/loadgen2/loadgen3), 운영 서버 영향 없음 |

## 최초 문제

이슈 #156(DR-02 격리 etcd Restore 검증)에서 `etcdutl snapshot restore`로
데이터 디렉터리를 준비한 뒤 3-member etcd 프로세스를 기동했으나, 세 노드가
raft pre-vote만 반복할 뿐 quorum을 형성하지 못했다.

```
dial tcp 192.168.55.20:2380: connect: no route to host
dial tcp 192.168.55.30:2380: i/o timeout
```

SSH(22번 포트)는 정상 동작해 scp/ssh 기반 파일 전송에는 문제가 없었기
때문에, 네트워크 경로 자체는 정상이라고 오판하기 쉬운 상황이었다.

## 원인

격리 VM 3대(loadgen/loadgen2/loadgen3)의 로컬 firewalld가 기본 정책상
etcd가 새로 사용하는 포트(2379 client / 2380 peer)를 차단하고 있었다.
기본 서비스인 SSH만 열려 있어 원격 접속 자체는 문제없었던 것이 오판의
원인이었다.

## 조치

이 환경은 운영망과 물리적으로 완전히 분리된 폐쇄 격리망(외부 노출 없음)
이므로, 포트 단위 예외 규칙을 추가하는 대신 firewalld 자체를 정지·
비활성화했다.

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

## 검증

- 조치 직후 별도 프로세스 재기동 없이, 진행 중이던 raft 재시도 election
  사이클에서 3대가 자동으로 peer 연결에 성공
- `etcdctl endpoint status`로 3-member RAFT TERM/INDEX 동일, leader 1개만
  존재함을 확인
- `etcdctl endpoint health` 3대 전부 `true`

## Before → After

```
Before
격리 VM 로컬 firewalld가 etcd 신규 포트(2379/2380) 차단
→ SSH는 정상이라 네트워크 문제로 인지 못함
→ raft pre-vote만 반복, quorum 미형성

After
격리망(외부 비노출) 전용이므로 firewalld 정지로 조치
→ 재시도 사이클에서 자동 peer 연결 성공, quorum 정상 형성
```

## 관련 근거

- Issue #156: https://github.com/seokpan/seokpan-infra/issues/156

## 후속 운영 기준

향후 격리 DR 테스트 환경을 준비할 때는 firewalld 상태를 사전 점검
항목으로 포함할 것을 권장한다. 이 환경은 검증 완료 후 즉시 폐기/정리되어
firewalld를 재활성화하지 않은 상태로 방치되지 않았다.
