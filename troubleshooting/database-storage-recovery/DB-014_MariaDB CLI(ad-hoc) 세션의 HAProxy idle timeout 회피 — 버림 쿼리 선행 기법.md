## TS-044: MariaDB CLI(ad-hoc) 세션의 HAProxy idle timeout 회피 — 버림 쿼리 선행 기법

**증상**: `mariadb` CLI로 접속한 세션이 60초 이상 idle 상태에서 다음 쿼리
실행 시 `ERROR 2013 (HY000): Lost connection to server during query` 또는
`ERROR 2006 (HY000): Server has gone away` 발생. 클라이언트가 자동 재연결
하지만, 재연결 이전 세션에서 설정한 세션 변수(`SET @var = ...`)는 소실되어
그 값을 참조하는 이후 쿼리가 `cannot be null` 등으로 실패.

**원인**: `G_backend_db_connection_guide.md` 6절에 문서화된 HAProxy
`timeout client/server=60s` 문제(seokpan-infra#142)와 동일. 커넥션 풀이 없는
ad-hoc 세션(CLI 직접 사용)은 6절의 재검증/recycle 대응을 적용할 수 없어
이 문제에 그대로 노출됨.

**해결(회피 기법)**: 실행할 SQL 블록의 맨 앞에 결과를 버릴 `SELECT 1;`을
추가한다. 세션이 이미 죽어있었다면 이 문장이 대신 소모되며 즉시 재연결이
일어나고, 그 직후 이어지는 문장들은 방금 열린 새 세션에서 문장 간격 없이
(수 ms 이내) 실행되므로 재차 60초를 넘길 일이 없다.

```sql
SELECT 1;   -- 죽은 세션을 대신 소모, 재연결 트리거용
SET @var = ...;
-- 이후 문장은 한 번에 이어붙여 실행
```

**한계**: 이 기법도 배치 시작 *전* 사람이 명령을 준비하는 동안 세션이 다시
죽는 것까지는 막지 못한다(재발 가능). 근본 해결은 아니며, 배치 실행 직전에
매번 이 패턴을 앞에 붙이는 습관화가 현재로선 유일한 대응.

**발견 경위**: 이슈 #115(DR-03 Redis 복구) 수동 검증 중 반복 발생, 총 4회
관찰 후 패턴 확정.

**관련**: G_backend_db_connection_guide.md 6절, seokpan-infra#142
