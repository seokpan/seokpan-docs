[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-032 — 원인 불명의 immutable(chattr +i)로 백업 디렉터리 생성이 EPERM으로 실패

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-10 02:00 |
| **상태** | **해결(재발 방지 Guard) / 근본 원인 미상** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MariaDB 백업 자동화(#55/#114/#143) |

## 최초 문제

mariadb-01의 `/srv/nfs/db-backup`에 `chattr +i`(immutable) 속성이 걸려
있어, Incremental→Full 승격 과정에서 신규 체인 디렉터리 생성이 `mkdir`
EPERM으로 실패했다.

```
lsattr -d /srv/nfs/db-backup
----i----------------- /srv/nfs/db-backup   (mariadb-01)
---------------------- /srv/nfs/db-backup   (mariadb-02, 정상)
```

이 속성은 `backup_transfer`의 `prepare.yml` 어디에서도 설정한 적이 없는
host drift였다 — 즉 Ansible 재배포로도 감지되지 않는 종류의 문제였다.
`getenforce`(Permissive)로 SELinux는 배제했고, `sudo -u root mkdir`로도
동일 EPERM이 재현돼 immutable 속성 자체가 원인임을 확정했다.

## 조치

당장의 장애는 ad-hoc `chattr -i`로 즉시 해제했다. 재발 방지를 위해
`prepare.yml`의 백업 Base 디렉터리 생성 직후에 다음 Guard를 추가했다.

1. `lsattr -d <디렉터리>` 실행(`check_mode: false`로 Check Mode에서도
   실제 속성을 검사)
2. 결과에 `i` 플래그가 포함돼 있으면 `assert`로 Play를 즉시 실패시킴
3. immutable 속성을 자동으로 해제하지 않음 — 원인 파악 없이 코드로
   걷어내면 재발 원인이 은폐될 위험이 있기 때문

## 검증

- 정상 상태(immutable 없음)에서 Guard 통과, `changed=false` 확인
- `chattr +i`로 장애 의도적 재현 → 실제 `mkdir` EPERM 재현 확인
- 같은 상태에서 Ansible Check Mode 실행 → Guard가 `lsattr`로 정상 탐지,
  `assert` 실패로 Fail-Fast, 명확한 에러 메시지 출력, 이후 태스크로
  진행하지 않음을 확인
- 원복(`chattr -i`) 후 재실행 → Guard 정상 통과

## Before → After

```
Before
host drift(immutable)가 재배포로도 감지되지 않음
+ 실패 시점은 실제 백업 실행 중(EPERM)

After
배포 단계에서 lsattr 기반 Fail-Fast Guard로 조기 탐지
+ 자동 해제하지 않고 원인 확인을 강제
```

## 남은 과제

mariadb-01에만 왜 이 속성이 걸려 있었는지 원인은 여전히 미상이다. 팀
채널 공유 후 이력을 확인할 예정이며, 이번 Guard는 재발 시 조기 감지만
보장한다.

## 관련 근거

- Issue #167: https://github.com/seokpan/seokpan-infra/issues/167
- PR #171: https://github.com/seokpan/seokpan-infra/pull/171
- TS-011(Check Mode 검증 Skip 선례), TS-031(같은 장애의 근본 원인 쪽)