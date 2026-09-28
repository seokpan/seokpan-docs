# NET-006 VRouter 영구 정적 Route 멱등성 오류

## 개요

- 발생/발견 시기: 2026-09-04
- 영역: Network / Ansible Automation
- 대상: VRouter-01 ~ VRouter-04
- 주 담당: 이유빈 — 네트워크 및 공통 인프라·Ansible 통합
- 관련 Infra Issue: #132 `[Ansible] vrouter_network Persistent Static Route 멱등성 보완`
- 관련 Infra PR: #134 `[Ansible] vrouter 영구 정적 라우트 멱등성 보완 (#132)`

## 문제

VRouter의 Runtime Route와 Persistent Route가 이미 Inventory의 Desired State와 동일한 상태에서도
`vrouter_network` Role을 Actual Run으로 반복 실행하면 다음 Task가 계속 실행됐다.

```text
Configure persistent static routes
```

그 결과 실제 Route 구성에는 변경이 필요하지 않은데도 매 실행마다 다음과 같이 변경이 발생했다.

```text
changed=1
```

이 문제는 실제 Static Route 장애나 Route 설계 오류가 아니라,
현재 상태와 목표 상태가 동일한지를 Ansible이 정확하게 판정하지 못해 발생한
**strict idempotency 문제**였다.

## 원인

기존 흐름에서는 현재 Persistent Route와 Desired Route를 조회하고 생성했지만,
두 값을 동일한 형식으로 정규화한 뒤 비교하여 변경 여부를 결정하는 조건이 없었다.

기존 동작 흐름은 다음과 같았다.

```text
현재 Persistent Route 조회
        ↓
Desired Route 생성
        ↓
현재값과 목표값의 동일 여부를 변경 조건으로 사용하지 않음
        ↓
nmcli connection modify 실행
        ↓
정상 상태에서도 changed=1
```

따라서 실제 Route 상태가 이미 올바른 경우에도
`nmcli connection modify`가 다시 실행되면서 불필요한 변경으로 기록됐다.

## 조치

현재 Persistent Route와 Desired Route를 동일한 방식으로 정규화한 뒤
두 List가 다를 때만 Persistent Route 변경 Task를 실행하도록 수정했다.

현재값은 다음 기준으로 정규화했다.

```yaml
current_persistent_routes.stdout.split(',')
| map('trim')
| reject('equalto', '')
| sort
| list
```

Desired Route도 동일하게 다음 과정을 적용했다.

```text
쉼표(,) 기준 분리
→ 앞뒤 공백 제거
→ 빈 값 제거
→ 정렬
→ List 변환
```

이후 실제 변경 Task에는 다음 조건을 적용했다.

```yaml
when:
  current_persistent_routes_normalized
  !=
  desired_persistent_routes_normalized
```

최종 동작은 다음과 같다.

```text
Current Persistent Route
        ↓
정규화
        ↓
Desired Persistent Route
        ↓
정규화
        ↓
List 비교
   ┌────┴────┐
 동일       다름
   ↓          ↓
 SKIP      nmcli 실행
changed=0  changed=1
```

이를 통해 Persistent Route가 Desired State와 동일한 경우에는
불필요한 `nmcli connection modify` 실행을 방지하도록 했다.

## 검증

### 1. 정상 상태 Actual Run 재실행

VRouter-01 ~ VRouter-04의 Route가 이미 Desired State와 동일한 상태에서
`vrouter_network`를 Actual Run으로 실행했다.

모든 VRouter에서 다음 결과를 확인했다.

```text
changed=0
failed=0
unreachable=0
```

동일한 상태에서 Actual Run을 다시 반복해도 결과는 동일했다.

따라서 정상 상태에서는 Persistent Route Task가 불필요하게 변경되지 않는 것을 확인했다.

### 2. 의도적 Drift 복구 검증

정상 상태 재실행만으로는 실제 Route 차이가 발생했을 때
자동화가 변경을 감지하고 복구하는지 확인할 수 없으므로
별도의 Drift 시험을 수행했다.

VRouter-01의 Persistent Route 중 1개를 의도적으로 누락시켜
Current State와 Desired State가 다른 상태를 만들었다.

```text
Current != Desired
```

이후 Actual Run을 실행했을 때 다음 Task가 실제로 수행됐다.

```text
Configure persistent static routes
```

결과:

```text
changed=1
```

누락시킨 Route가 Desired State에 맞게 다시 복구된 것을 확인했다.

복구 이후 다시 Actual Run을 실행한 결과:

```text
changed=0
failed=0
unreachable=0
```

을 확인했다.

즉,

```text
정상 상태
→ 변경하지 않음

Drift 발생
→ 차이 감지
→ 변경 수행
→ Desired State 복구

복구 후 재실행
→ 다시 변경하지 않음
```

의 흐름을 검증했다.

### 3. Runtime / Persistent Route 검증

자동화 결과뿐 아니라 Runtime Route와 Persistent Route 검증 Task도
정상적으로 PASS하는 것을 확인했다.

이를 통해 Ansible 실행 결과만 `changed=0`으로 만든 것이 아니라,
실제 Route 상태도 Desired State와 일치함을 확인했다.

### 4. Gateway 통신 검증

외부 Gateway에 대해 VRouter-01 ~ VRouter-04 모두 다음 결과를 확인했다.

```text
0% packet loss
```

또한 VRouter-01에서 다음 내부망 Gateway 통신을 검증했다.

```text
192.168.52.10
192.168.53.10
192.168.54.10
```

모두 다음 결과를 확인했다.

```text
0% packet loss
```

## 결과

Persistent Route 변경 여부를 Current State와 Desired State의
정규화된 값 비교를 기반으로 판단하도록 변경했다.

그 결과:

- 정상 상태 반복 실행 시 `changed=0`
- 실제 Drift 발생 시에만 `changed=1`
- Drift 발생 후 Desired State 자동 복구
- 복구 후 재실행 시 다시 `changed=0`
- Runtime / Persistent Route 검증 PASS
- Gateway 통신 정상

을 확인했다.

따라서 VRouter Persistent Static Route에 대해
실제 변경이 있을 때만 변경하도록 strict idempotency를 확보했다.

## 판단 기준 및 주의사항

이번 사례는 **Static Route 자체의 장애를 해결한 사례가 아니다.**

실제 Route와 Gateway 통신은 정상인 상태에서
Ansible이 불필요한 변경을 반복해서 기록하던 문제를 해결한 사례다.

따라서 다음과 같이 구분해야 한다.

```text
Route 통신 장애
≠
Persistent Route 변경 판정 오류
```

또한 Check Mode에서 `changed=0`이 나온다는 사실만으로
멱등성 확보를 완료 판단하지 않았다.

최종 검증은 다음 두 Evidence를 함께 사용했다.

1. 정상 상태에서 반복 Actual Run 시 `changed=0`
2. 의도적인 Drift 발생 후 Actual Run으로 복구되고, 재실행 시 다시 `changed=0`

이를 통해 단순히 변경 Task가 실행되지 않는 것이 아니라,
**필요할 때는 변경하고 필요하지 않을 때는 변경하지 않는 상태**를 검증했다.
