[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-026 — 공용 TLS Role과 MaxScale Role의 권한 설정 충돌로 인증서 접근 권한이 다시 사라짐

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-07 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MaxScale TLS 파일 권한, 공용 `tls_deploy` 재실행, Backend DB 접속 경로 |

## 문제 개요

TS-025의 MaxScale `--check --diff` 인증정보 미노출 재검증 과정에서 `maxscale-01`의 TLS 인증서와 개인키 권한이 MaxScale 서비스 계정이 읽을 수 없는 상태로 확인됐다.

당시 실제 파일 권한은 다음과 같았다.

```text
/etc/pki/seokpan-ca/services  root:root 0750
maxscale-01.crt               root:root 0644
maxscale-01.key               root:root 0600
```

`maxscale` 사용자로 인증서와 개인키 읽기가 모두 실패했고, `maxscale --config-check`도 TLS 파일 `Permission denied`로 실패했다.

한편 실행 중인 MaxScale Service와 `Read-Write-Listener:3306`은 계속 동작하고 있었다. 기존 프로세스가 이미 읽어 둔 인증서와 키를 사용 중일 가능성이 있으므로 단순 확인을 위해 임의로 restart/reload하지 않고 파일 권한이 다시 바뀐 원인부터 확인했다.

## 기존 TS-018과의 사건 구분

[TS-018](TS-018_MaxScale_TLS_인증서_권한_SAN_검증.md)은 MaxScale TLS를 **처음 적용하던 당시** 공용 `tls_deploy`가 배포한 root 전용 권한과 `maxscale` 서비스 계정이 필요로 하는 권한이 맞지 않았던 문제를 다룬다.

당시에는 MaxScale Role이 나중에 실행되면서 다음 권한으로 다시 맞춰 정상화했다.

```text
/etc/pki/seokpan-ca           root:root     0711
services                      root:maxscale 0750
maxscale-01.key               root:maxscale 0640
```

이번 사건은 그 이후에 같은 권한 문제가 다시 나타났다는 점이 다르다. 코드를 대조한 결과 공용 `tls_deploy`와 MaxScale Role이 동일한 파일과 디렉터리에 서로 다른 권한을 적용하고 있었다.

```text
TS-018
→ 최초 TLS 적용 시 MaxScale 서비스 계정용 권한이 부족
→ MaxScale Role이 나중에 권한을 다시 맞춤
→ TLS 정상화

이번 사건
→ 이후 인증서/키 접근 권한이 다시 사라짐
→ tls_deploy와 MaxScale Role이 서로 다른 권한을 적용하는 구조 확인
→ 실행 순서에 따라 최종 권한이 달라질 수 있음
```

따라서 같은 MaxScale TLS 영역이지만 최초 적용 시의 권한 누락과, 이후 두 Role의 권한 설정 충돌로 다시 발생한 문제는 원인과 재발 방식이 달라 별도 TS로 기록한다.

## 원인 분석

당시 공용 `tls_deploy`와 MaxScale Role이 같은 인증서 디렉터리와 개인키에 서로 다른 권한을 적용했다.

### 공용 `tls_deploy`

```text
certificate directory → root:root 0750
certificate           → root:root 0644
private key           → root:root 0600
```

### MaxScale Role

```text
certificate directory → root:maxscale 0750
private key           → root:maxscale 0640
```

실행 순서가:

```text
tls_deploy → maxscale Role
```

이면 MaxScale Role이 마지막에 권한을 다시 맞추므로 정상 동작한다. 하지만 인증서 재발급 등으로 `tls_deploy`가 다시 실행된 뒤 MaxScale Role이 이어서 실행되지 않으면 파일 권한이 다시 `root:root` 기준으로 바뀔 수 있었다.

문제의 핵심은 단순한 파일 권한 오타가 아니라 **같은 TLS 파일을 관리하는 두 Ansible Role이 서로 다른 권한을 적용해 실행 순서에 따라 최종 권한이 달라지는 구조**였다.

## 조치

### 1. 공용 `tls_deploy`에 서비스별 권한 설정 추가

Infra #146 / PR #148에서 공용 Role이 다음 값을 서비스별로 지정할 수 있도록 변경했다.

```text
tls_service_dir_group
tls_service_dir_mode
tls_service_key_group
tls_service_key_mode
```

기본값은 기존 `root` 전용 권한을 유지해 다른 서비스의 동작은 바꾸지 않았다.

### 2. MaxScale용 권한을 명시

Infra #147 / PR #149에서 `maxscale-01`에 다음 값을 지정했다.

```yaml
tls_service_dir_group: maxscale
tls_service_dir_mode: "0750"
tls_service_key_group: maxscale
tls_service_key_mode: "0640"
```

이후 공용 `tls_deploy`와 MaxScale Role 모두 같은 최종 권한을 적용하도록 맞췄다.

## 검증

PR #149에서 실제 서버 상태를 다시 확인했다.

### 최종 파일 권한

```text
/etc/pki/seokpan-ca                      root:root       0711
/etc/pki/seokpan-ca/services             root:maxscale   0750
maxscale-01.crt                          root:root       0644
maxscale-01.key                          root:maxscale   0640
```

### MaxScale 계정 접근

```text
CRT=0
KEY=0
```

인증서와 개인키 모두 읽기 성공.

### Config 검증

```text
maxscale --config-check --config=/etc/maxscale.cnf
→ Configuration was successfully verified.
```

### 서비스 상태

```text
systemctl is-active maxscale
→ active

Read-Write-Listener : 3306 : Running
```

### VIP TLS

`db.seokpan.soldesk.store`를 통해 실제 TLS 세션을 확인했다.

```text
SSL: Cipher in use is TLS_AES_256_GCM_SHA384, cert is OK
Connection: db.seokpan.soldesk.store via TCP/IP
```

### 재발 방지 검증

같은 설정으로 `tls_deploy`를 다시 실행했다.

```text
재발급 필요 = False
ok=13
changed=0
failed=0
```

재실행 이후에도 다음 권한이 유지됐다.

```text
root:root       711  /etc/pki/seokpan-ca
root:maxscale   750  /etc/pki/seokpan-ca/services
root:root       644  /etc/pki/seokpan-ca/services/maxscale-01.crt
root:maxscale   640  /etc/pki/seokpan-ca/services/maxscale-01.key
```

그리고 다시 `CRT=0`, `KEY=0`으로 MaxScale 계정이 인증서와 개인키를 계속 읽을 수 있음을 확인했다.

## Before → Change → After

```text
Before
TS-018에서 MaxScale Role이 나중에 권한을 다시 맞춰 정상화
→ 공용 tls_deploy와 MaxScale Role은 서로 다른 권한 설정을 계속 사용
→ 실행 순서에 따라 최종 권한이 달라질 수 있음
→ 실제 cert/key read 실패 + config-check Permission denied 확인

Change
공용 tls_deploy에 서비스별 directory/key 권한 설정 추가
→ maxscale-01에 root:maxscale 권한 명시
→ 두 Role이 같은 권한을 적용하도록 통일

After
MaxScale cert/key read PASS
→ config-check PASS
→ Service / Listener 정상
→ VIP TLS 검증 PASS
→ tls_deploy 재실행 changed=0
→ 재실행 후에도 파일 권한과 서비스 상태 유지
```

## 운영 기준

MaxScale TLS 권한은 더 이상 "공용 Role 적용 후 MaxScale Role이 다시 고쳐주는 실행 순서"에 의존하지 않는다.

현재는 `tls_deploy` 단계부터 MaxScale에 필요한 디렉터리와 개인키 권한을 적용하고, MaxScale Role도 같은 값을 사용한다.

```text
공용 tls_deploy
→ MaxScale에 필요한 root:maxscale 권한 적용

MaxScale Role
→ 같은 권한 유지

결과
→ 어느 Role을 다시 실행해도 최종 권한이 달라지지 않음
```

## 관련 근거

- Docs Issue #63: https://github.com/seokpan/seokpan-docs/issues/63
- 기존 최초 TLS 권한 사건 TS-018: TS-018_MaxScale_TLS_인증서_권한_SAN_검증.md
- Infra Issue #146: https://github.com/seokpan/seokpan-infra/issues/146
- Infra PR #148: https://github.com/seokpan/seokpan-infra/pull/148
- Infra Issue #147: https://github.com/seokpan/seokpan-infra/issues/147
- Infra PR #149: https://github.com/seokpan/seokpan-infra/pull/149
- PR #149 Merge Commit: https://github.com/seokpan/seokpan-infra/commit/9ce9f979d937caf643fcd702b841c6e896604344
