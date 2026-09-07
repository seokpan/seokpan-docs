[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-018 — MaxScale TLS 적용 중 인증서 권한·상위 디렉터리·SAN 검증 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(Backend DB 연동 경로), 최유준(`tls_deploy`/`internal_ca` role 소유), Backend 전체 |

## 문제 개요

MaxScale `Read-Write-Listener:3306`에 TLS를 적용하는 과정(Issue #102)에서 인증서 자체 발급은 정상 동작했지만, 실제 서비스 기동과 접속 검증 단계에서 서로 다른 원인의 실패가 연속으로 발생했다.

```text
2026-09-04 19:29:09   error  : (Read-Write-Listener); Bad path parameter
'/etc/pki/seokpan-ca/services/maxscale-01.crt': 13, Permission denied
```

## 발생 경위 및 원인 분석

### 1차 — 인증서/키 파일 권한 불일치

`internal_ca` + `tls_deploy` Role은 인증서와 키를 `root:root` 기준으로 배포한다. 그런데 MaxScale은 systemd에서 `--user=maxscale`로 구동되므로 `root:root 0600`인 개인키를 MaxScale Process가 읽을 수 없었다.

```text
maxscale.service 확인 결과: ExecStart=/usr/bin/maxscale --user=maxscale
```

### 2차 — 상위 디렉터리 traverse 차단

파일 자체 권한을 `group: maxscale`, `0640`으로 고친 뒤에도 동일한 Permission denied가 재현됐다.

원인은 인증서가 위치한 경로의 상위 디렉터리 `/etc/pki/seokpan-ca`가 `root:root 0750`이어서 `maxscale` User가 하위 파일까지 접근할 수 없었던 것이었다.

### 3차 — `maxscale --config-check`의 root 실행 거부

앞의 권한 문제를 해결한 뒤 추가한 사전 검증 Task가 다음 오류로 실패했다.

```text
alert  : MaxScale cannot be run as root.
```

`maxscale --config-check`도 실제 서비스 계정인 `maxscale` User로 실행해야 했다. `become_user: maxscale`을 명시해 해결했다.

### 리뷰에서 추가 확인 — Canonical Endpoint SAN 누락과 재발급 조건 한계

리뷰 과정에서 다음 두 항목을 추가 확인했다.

1. 인증서 SAN에 Project DB Canonical Endpoint `db.seokpan.soldesk.store`가 빠져 있었다.
2. 당시 `tls_deploy` Role의 재발급 판정은 만료 임박 여부만 확인해 `tls_service_san`을 변경해도 기존 인증서의 SAN 변경을 감지하지 못했다.

Backend는 MaxScale Host의 실제 IP가 아니라 Common VIP/Canonical Endpoint를 사용하므로 hostname 검증을 활성화하려면 `db.seokpan.soldesk.store`가 SAN에 포함되어야 했다.

당시 PR #137 범위에서는 SAN을 추가한 뒤 기존 인증서를 수동 삭제하고 `tls_deploy`를 재실행해 새 인증서를 발급했다. SAN 변경 자동 감지 자체는 별도 Issue #138로 분리했다.

## 당시 조치

```text
1. maxscale Role에 권한 보정 Task 추가
   - 상위 디렉터리 /etc/pki/seokpan-ca traverse 권한 0711
   - 인증서 디렉터리 group: maxscale, mode: 0750
   - 개인키 group: maxscale, mode: 0640
2. config-check Task를 become_user: maxscale로 실행
3. SAN에 VIP(10.1.93.90) + db.seokpan.soldesk.store 포함
4. 당시 SAN 변경 반영은 기존 인증서 수동 삭제
   → tls_deploy 재실행
   → maxscale Role 재실행으로 권한 보정
   → 서비스 재기동 순서로 처리
```

## 검증

- `--skip-ssl` 없이 VIP `10.1.93.90:3306` 접속 성공
- `Cipher in use is TLS_AES_256_GCM_SHA384, cert is OK` 확인
- `db.seokpan.soldesk.store` + `--ssl-ca` + `--ssl-verify-server-cert` 기준으로 mariadb-01/02 양쪽에서 CA/Hostname 검증 통과
- 인증서 SAN에 다음 항목 포함 확인
  - `DNS:maxscale-01.seokpan.internal`
  - `DNS:db.seokpan.soldesk.store`
  - `IP:192.168.53.40`
  - `IP:10.1.93.90`

## 후속 운영 기준

이 사건 당시에는 SAN 변경을 자동 감지하지 못했기 때문에 기존 인증서를 수동 삭제한 뒤 재발급했다. 이 절차는 당시 문제를 해소하기 위한 일회성 우회였으며 현재 정상 운영 절차는 아니다.

이후 `seokpan-infra#138` / PR #140에서 `tls_deploy` Role이 실제 인증서 SAN과 `tls_service_san`을 정규화해 비교하고, SAN이 다르면 유효기간이 남아 있어도 자동 재발급하도록 보완됐다.

후속 검증에서는 다음을 실제 실행으로 확인했다.

- MaxScale Desired SAN에 테스트 DNS SAN을 추가했을 때 `SAN 불일치=True`, `재발급 필요=True`로 자동 판정
- 수동 인증서 삭제 없이 새 인증서 발급 및 Fingerprint 변경
- 테스트 SAN이 새 인증서에 반영됨
- 테스트 SAN을 제거해 원래 계약으로 복귀했을 때 다시 자동 재발급
- 원래 SAN 4개가 정상 복원됨
- 같은 설정으로 재실행 시 재발급 관련 작업 Skip, Fingerprint 유지
- Harbor에 현재 SAN 설정으로 `tls_deploy`를 실행했을 때 불필요한 재발급 없이 Fingerprint 유지

현재 SAN 변경 절차는 다음을 기준으로 한다.

```text
Desired tls_service_san 변경
→ tls_deploy가 현재 인증서 SAN과 비교
→ SAN mismatch면 자동 재발급
→ 동일 SAN이면 재발급하지 않음
```

따라서 현재 운영에서는 SAN 변경을 반영하기 위해 인증서를 먼저 수동 삭제하지 않는다.

## Before → After

```text
사건 당시 Before
tls_deploy가 배포한 인증서/키를 root:root 그대로 사용
→ MaxScale(non-root User)이 못 읽음
→ 상위 디렉터리까지 traverse 불가
→ config-check도 root 실행 거부
→ SAN에 실제 Canonical Endpoint 누락

사건 당시 After
MaxScale Role에서 디렉터리·키 권한 보정
→ config-check를 maxscale User로 실행
→ SAN에 VIP + Canonical Endpoint 포함
→ TLS/CA/Hostname 검증 통과

후속 운영 기준
SAN 변경 시 수동 인증서 삭제가 필요했던 한계
→ #138 / PR #140에서 SAN mismatch 자동 감지
→ 수동 삭제 없이 자동 재발급 및 동일 설정 수렴 검증
```

## 관련 독립 사건

MaxScale 계정 Credential을 Ansible Vault로 관리한 뒤에도 `maxscale.cnf` Template의 Diff 출력에서 민감값이 노출될 수 있었던 문제는 TLS 인증서 사건과 별도 Root Cause다. 해당 사건은 [TS-025 — MaxScale 설정 배포의 `--check --diff`에서 Credential 노출 위험](TS-025_MaxScale_check_diff_Credential_노출_방지.md)에서 독립적으로 기록한다.

TS-025 재검증 과정에서 이후 MaxScale cert/key 권한 Drift가 다시 확인됐다. 이 후속 사건은 최초 TLS 적용 당시의 권한 누락과 달리, 현재 Runtime에서 Consumer 권한 유실이 확인됐고 공용 `tls_deploy`와 MaxScale Role이 동일 파일·디렉터리에 서로 다른 Desired State를 선언해 실행 순서에 따라 최종 상태가 달라질 수 있었던 구조가 원인이었다. `seokpan-infra#146` / PR #148에서 공용 TLS Role에 서비스별 권한 계약을 추가하고, `#147` / PR #149에서 MaxScale이 동일 권한 계약을 사용하도록 정합화한 뒤 재실행 회귀검증까지 완료했다. 해당 사건은 [TS-026 — 공용 TLS Role과 MaxScale Role의 상충 Desired State로 발생한 권한 Drift](TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md)에서 독립적으로 기록한다.

## 관련 근거

- Issue #102: https://github.com/seokpan/seokpan-infra/issues/102
- PR #137: https://github.com/seokpan/seokpan-infra/pull/137
- 후속 Issue #138: https://github.com/seokpan/seokpan-infra/issues/138
- 후속 PR #140: https://github.com/seokpan/seokpan-infra/pull/140
- 독립 Credential Diff 사건 TS-025: TS-025_MaxScale_check_diff_Credential_노출_방지.md
- 후속 TLS 권한 Drift Docs Issue #63: https://github.com/seokpan/seokpan-docs/issues/63
- 후속 TLS 권한 Drift TS-026: TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md
