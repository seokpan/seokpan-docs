[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-037 — DR-02 격리 VM 간 9시간 Clock Drift(타임존 UTC 잔존)

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 격리 검증 환경(loadgen2/loadgen3), 운영 서버 영향 없음 |

## 최초 문제

TS-036 조치(firewalld 정지) 후 3-member etcd의 peer 연결 자체는
성공했으나, 로그에 다음과 같은 clock drift 경고가 다수 발생했다.

```
prober found high clock drift ... clock-drift: 8h59m59.161586822s
```

## 원인

격리 VM 3대 중 2대(loadgen2/loadgen3)의 시스템 타임존이 `UTC`로 설정된
상태(운영 표준 KST 대비 정확히 9시간 차이)였다. 격리망이라 외부 NTP
서버에도 닿지 않아(`NTP service: inactive`) 시계가 스스로 동기화될
방법이 없었다.

## 조치

3대 모두 타임존을 `Asia/Seoul`로 통일하고, NTP 없이 수동으로 시각을
맞췄다.

```bash
sudo timedatectl set-timezone Asia/Seoul
sudo timedatectl set-ntp false
sudo date -s "<pc2 기준 현재 시각>"
```

data-dir(`/var/lib/etcd-dr`)은 그대로 유지한 채 etcd 프로세스만
재기동했다(snapshot restore 재실행 불필요).

## 검증

- `timedatectl status`로 3대 `Local time`이 수 초 이내 오차로 일치함을
  확인
- 재기동 후 로그에서 clock-drift 경고가 9시간대에서 수 초~22초 수준으로
  감소, 이후 quorum 정상 유지 확인(raft 자체는 시계와 무관하게
  동작하므로 quorum 형성에는 영향 없었음)

## Before → After

```
Before
loadgen2/3 타임존이 UTC로 설정된 채 방치, NTP도 비활성
→ KST 기준 9시간 clock drift 경고 지속 발생

After
3대 타임존을 Asia/Seoul로 통일 + 수동 시각 동기화
→ clock drift가 정상 범위(수 초)로 수렴
```

## 관련 근거

- Issue #156: https://github.com/seokpan/seokpan-infra/issues/156

## 후속 운영 기준

raft 합의 자체는 시계 오차와 무관하게 동작하지만, 이후 단계(TLS 인증서
유효기간, RTO 측정 등)에는 영향을 줄 수 있다. 향후 격리 DR 테스트용 VM은
기동 직후 `timedatectl status` 확인을 사전 점검 항목으로 포함할 것을
권장한다.
