[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-019 — 빈 Project 재현성 Smoke에서 Network/LB 자동화의 NIC Fact 구조 가정과 Check Mode 검증 결함이 드러남

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-09-01 |
| **상태** | **진행 중** |
| **주 담당** | **이유빈 — 네트워크 및 공통 인프라·Ansible 통합** |
| **영향 역할** | 정태훈(Kubernetes 사용), 김상희(Data Endpoint), 최유준(CI/CD·관측 Endpoint) |
| **핵심 범주** | Ansible Facts / Network Automation / Check Mode / Reproducibility |

## 발견 배경

기존 Project `.venv`와 Collection을 재사용하지 않는 빈 Project 환경에서 Version Matrix를 재생성한 뒤 Common / Network / LB 최소 Smoke를 수행했다.

이 검증은 기존 Runtime이 이미 정상인 상태에서 **코드가 새 실행환경에서도 같은 검증 결과를 내는지** 확인하기 위한 것이었다.

## 문제 1 — NIC Fact 필터링이 모든 `ansible_facts` 항목을 Interface 객체처럼 가정

다음 Role에서 동일한 오류가 발생했다.

- `vrouter_network`
- `vrouter_firewall`
- `lb_network`

오류:

```text
object of type 'list' has no attribute 'address'
```

발생 예:

```text
Find internal interface candidates
Find LB external interface candidates
```

실제 VRouter Fact 수집과 연결 자체는 정상이었다. 예를 들어 `vrouter-01`은:

- `ens160`: `10.1.93.71`
- `ens224`: `192.168.51.10`

으로 올바르게 수집됐다.

따라서 Network Runtime 장애가 아니라 **Fact Filter가 `ansible_facts` 전체 값을 순회하면서 각 값이 항상 `ipv4.address`를 가진 Mapping이라고 가정한 코드 결함**이다. 실제 Fact 안에는 List 등 다른 자료형도 존재하므로 새 실행환경/Fact 구조에서 이 가정이 깨졌다.

## 문제 2 — HAProxy가 정상인데 Check Mode에서 “inactive”로 오판

`lb_haproxy --check --diff`에서는 다음 Assert가 실패했다.

```text
HAProxy service is not active.
```

하지만 실제 확인:

```text
systemctl is-active haproxy
→ rc=0
→ active
```

원인은 직전 조회 Task인 `Check HAProxy service`가 Check Mode에서 `skipping` 되었는데, 후속 `Verify HAProxy service is active` Assert는 그대로 실행된 것이었다.

즉 서비스 장애가 아니라 **조회성 검증 Task와 Assert의 Check Mode 실행 조건이 서로 달라 생긴 False Negative**다.

## 현재 판정

```text
Runtime Network / HAProxy
→ 정상

빈 Project 재현성 Smoke
→ Network/LB 자동화 코드 결함 발견
```

이 사례는 기존 Runtime에서 우연히 정상 동작했다는 사실만으로 자동화의 재현성이 증명되지 않는다는 점을 보여준다. 특히 Version Lock 이후 빈 실행환경 Smoke가 **숨겨진 자료형 가정과 Check Mode 검증 결함을 실제로 드러냈다.**

## 수정 방향

NIC Fact 쪽은:

- Interface 후보를 명확히 한정해서 순회
- 값이 Mapping인지 확인
- `ipv4` 존재 여부 확인
- `address` 존재 여부 확인

등의 방어적 필터링이 필요하다.

HAProxy 쪽은:

- read-only `systemctl is-active` 조회에 `check_mode: false`를 명시하거나
- Check Mode에서 조회 Task가 skip될 경우 후속 Assert도 동일 조건으로 제외

하는 방식으로 실행 의미를 맞춰야 한다.

## 아직 해결이라고 쓰면 안 되는 이유

- Issue #84의 Smoke에서 결함 발견까지는 완료
- 실제 Runtime은 정상
- Network/LB 코드 보완은 후속 작업으로 분리 예정
- 수정 후 빈 Project 환경에서 동일 Smoke 재실행 Evidence가 아직 없음

따라서 상태는 **진행 중**이다.

## 근거

- Clean Environment / Smoke Issue #84: https://github.com/seokpan/seokpan-infra/issues/84

---
