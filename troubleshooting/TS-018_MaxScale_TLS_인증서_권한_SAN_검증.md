[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-018 — MaxScale TLS 적용 중 인증서 파일 권한과 SAN 검증 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | Backend DB 접속 경로, MaxScale TLS 인증서 배포 및 검증 |

## 문제 개요

MaxScale `Read-Write-Listener:3306`에 TLS를 적용하는 과정(Issue #102)에서 인증서 발급 자체는 정상 동작했지만, 실제 서비스 기동과 접속 검증 단계에서 서로 다른 원인의 실패가 연속으로 발생했다.

```text
2026-09-04 19:29:09   error  : (Read-Write-Listener); Bad path parameter
'/etc/pki/seokpan-ca/services/maxscale-01.crt': 13, Permission denied
```

## 발생 경위 및 원인 분석

### 1차 — 인증서와 개인키 파일 권한 불일치

`internal_ca`와 `tls_deploy` Role은 인증서와 개인키를 `root:root` 기준으로 배포한다. 그런데 MaxScale은 systemd에서 `--user=maxscale`로 실행되므로 `root:root 0600`인 개인키를 MaxScale 프로세스가 읽을 수 없었다.

```text
maxscale.service 확인 결과: ExecStart=/usr/bin/maxscale --user=maxscale
```

### 2차 — 상위 디렉터리 접근 권한 부족

파일 자체 권한을 `group: maxscale`, `0640`으로 고친 뒤에도 동일한 `Permission denied`가 재현됐다.

원인은 인증서가 위치한 경로의 상위 디렉터리 `/etc/pki/seokpan-ca`가 `root:root 0750`이어서 `maxscale` 사용자가 하위 파일까지 접근하는 데 필요한 실행 권한을 갖지 못한 것이었다.

### 3차 — `maxscale --config-check`의 root 실행 거부

앞의 권한 문제를 해결한 뒤 추가한 사전 검증 Task가 다음 오류로 실패했다.

```text
alert  : MaxScale cannot be run as root.
```

`maxscale --config-check`도 실제 서비스 계정인 `maxscale` 사용자로 실행해야 했다. `become_user: maxscale`을 명시해 해결했다.

### 리뷰에서 추가 확인 — SAN 누락과 재발급 조건 한계

리뷰 과정에서 다음 두 항목을 추가 확인했다.

1. 인증서 SAN(Subject Alternative Name, 인증서가 유효한 DNS 이름과 IP 목록)에 서비스에서 실제 사용하는 공용 DB 주소 `db.seokpan.soldesk.store`가 빠져 있었다.
2. 당시 `tls_deploy` Role은 인증서 만료 임박 여부만 확인했기 때문에 `tls_service_san`을 변경해도 기존 인증서의 SAN 변경을 감지하지 못했다.

Backend는 MaxScale Host의 실제 IP가 아니라 VIP(Virtual IP, 가상 IP) `10.1.93.90`과 `db.seokpan.soldesk.store`를 사용하므로 hostname 검증을 활성화하려면 해당 DNS 이름이 SAN에 포함되어야 했다.

당시 PR #137 범위에서는 SAN을 추가한 뒤 기존 인증서를 수동 삭제하고 `tls_deploy`를 재실행해 새 인증서를 발급했다. SAN 변경 자동 감지 자체는 별도 Issue #138로 분리했다.

## 당시 조치

```text
1. maxscale Role에 권한 보정 Task 추가
   - 상위 디렉터리 /etc/pki/seokpan-ca 접근 권한 0711
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
- `db.seokpan.soldesk.store` + `--ssl-ca` + `--ssl-verify-server-cert` 기준으로 mariadb-01/02 양쪽에서 CA 및 Hostname 검증 통과
- 인증서 SAN에 다음 항목 포함 확인
  - `DNS:maxscale-01.seokpan.internal`
  - `DNS:db.seokpan.soldesk.store`
  - `IP:192.168.53.40`
  - `IP:10.1.93.90`

## 후속 운영 기준

이 사건 당시에는 SAN 변경을 자동 감지하지 못했기 때문에 기존 인증서를 수동 삭제한 뒤 재발급했다. 이 절차는 당시 문제를 해소하기 위한 일회성 우회였으며 현재 정상 운영 절차는 아니다.

이후 `seokpan-infra#138` / PR #140에서 `tls_deploy` Role이 실제 인증서 SAN과 `tls_service_san`을 정규화해 비교하고, SAN이 다르면 유효기간이 남아 있어도 자동 재발급하도록 보완됐다.

후속 검증에서는 다음을 실제 실행으로 확인했다.

- MaxScale 설정의 SAN 목록에 테스트 DNS 이름을 추가했을 때 `SAN 불일치=True`, `재발급 필요=True`로 자동 판정
- 수동 인증서 삭제 없이 새 인증서 발급 및 인증서 지문(Fingerprint) 변경
- 테스트 SAN이 새 인증서에 반영됨
- 테스트 SAN을 제거해 원래 설정으로 복귀했을 때 다시 자동 재발급
- 원래 SAN 4개가 정상 복원됨
- 같은 설정으로 재실행 시 재발급 관련 작업이 실행되지 않고 Fingerprint 유지
- Harbor에 현재 SAN 설정으로 `tls_deploy`를 실행했을 때 불필요한 재발급 없이 Fingerprint 유지

현재 SAN 변경 절차는 다음을 기준으로 한다.

```text
tls_service_san 변경
→ tls_deploy가 현재 인증서 SAN과 비교
→ SAN이 다르면 자동 재발급
→ SAN이 같으면 재발급하지 않음
```

따라서 현재 운영에서는 SAN 변경을 반영하기 위해 인증서를 먼저 수동 삭제하지 않는다.

## Before → After

```text
사건 당시 Before
tls_deploy가 배포한 인증서/키를 root:root 그대로 사용
→ MaxScale 서비스 계정이 읽지 못함
→ 상위 디렉터리 접근 권한도 부족
→ config-check도 root 실행 거부
→ SAN에 실제 사용하는 공용 DB 주소 누락

사건 당시 After
MaxScale Role에서 디렉터리·키 권한 보정
→ config-check를 maxscale 사용자로 실행
→ SAN에 VIP + db.seokpan.soldesk.store 포함
→ TLS/CA/Hostname 검증 통과

후속 운영 기준
SAN 변경 시 수동 인증서 삭제가 필요했던 한계
→ #138 / PR #140에서 SAN 불일치 자동 감지
→ 수동 삭제 없이 자동 재발급
→ 같은 설정 재실행 시 불필요한 재발급 없음
```

## 관련 독립 사건

MaxScale 계정 인증정보를 Ansible Vault로 관리한 뒤에도 `maxscale.cnf` Template의 Diff 출력에서 민감값이 노출될 수 있었던 문제는 TLS 인증서 사건과 원인이 다르다. 해당 사건은 [TS-025 — MaxScale 설정 배포의 `--check --diff`에서 인증정보 노출 위험](TS-025_MaxScale_check_diff_Credential_노출_방지.md)에서 독립적으로 기록한다.

TS-025 재검증 과정에서는 이후 MaxScale 인증서와 개인키의 접근 권한이 다시 사라진 별도 문제가 확인됐다. 이 사건은 최초 TLS 적용 당시의 권한 누락과 달리, 공용 `tls_deploy`와 MaxScale Role이 같은 파일과 디렉터리에 서로 다른 권한을 적용해 실행 순서에 따라 최종 권한이 달라질 수 있었던 구조가 원인이었다. `seokpan-infra#146` / PR #148에서 공용 TLS Role에 서비스별 권한 설정을 추가하고, `#147` / PR #149에서 MaxScale도 같은 권한 설정을 사용하도록 맞춘 뒤 재실행 검증까지 완료했다. 해당 사건은 [TS-026 — 공용 TLS Role과 MaxScale Role의 권한 설정 충돌로 인증서 접근 권한이 다시 사라짐](TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md)에서 독립적으로 기록한다.

## 관련 근거

- Issue #102: https://github.com/seokpan/seokpan-infra/issues/102
- PR #137: https://github.com/seokpan/seokpan-infra/pull/137
- 후속 Issue #138: https://github.com/seokpan/seokpan-infra/issues/138
- 후속 PR #140: https://github.com/seokpan/seokpan-infra/pull/140
- 독립 인증정보 Diff 사건 TS-025: TS-025_MaxScale_check_diff_Credential_노출_방지.md
- 후속 TLS 권한 문제 Docs Issue #63: https://github.com/seokpan/seokpan-docs/issues/63
- 후속 TLS 권한 문제 TS-026: TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md
