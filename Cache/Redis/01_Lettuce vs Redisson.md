# Lettuce vs Redisson

> Java Redis 클라이언트 선택 기준 정리. 결론부터: **기본은 Lettuce, 분산 락이 필요해지는 순간 Redisson 추가.**
> "Lettuce vs Redisson"이 아니라 "Lettuce + (필요시) Redisson"이 실무 감각에 가깝다.

## 한눈에 비교

| 구분 | Lettuce | Redisson |
|---|---|---|
| 성격 | Redis 명령 중심의 저수준 클라이언트 | Redis 위에 분산 객체/서비스를 얹은 고수준 프레임워크 |
| 내부 구조 | Netty 기반 비동기/논블로킹 | Netty 기반 비동기/논블로킹 (동일) |
| 스레드 안전성 | 커넥션 하나를 여러 스레드가 공유 가능 (풀 불필요) | 동일 |
| 클러스터 지원 | MOVED/ASK 리다이렉트, 슬롯 맵 캐싱/갱신, failover 자동 처리 | 동일하게 전부 지원 — 변별점 아님 |
| Spring Boot | `spring-boot-starter-data-redis` **기본 클라이언트** | 별도 의존성 (`redisson-spring-boot-starter`) |
| API 스타일 | GET/SET 등 Redis 명령이 투명하게 보임 | `RLock`, `RMap`, `RQueue` 등 추상화된 객체 |
| 대표 용도 | 캐시, 세션, 일반 조회/저장, Pub/Sub | 분산 락, RateLimiter, 분산 세마포어, 분산 자료구조 |

핵심: 클러스터 대응력은 둘 다 성숙해서 선택 기준이 못 된다. 실제 기준은 **API 추상화 레벨**.

## Lettuce를 기본으로 두는 경우 (대부분)

- Redis를 캐시, 단순 조회/저장, Pub/Sub으로 쓰는 일반적인 경우
- Spring 생태계(`RedisTemplate`, `@Cacheable`, Spring Session)가 전부 Lettuce 위에서 검증돼 있어 설정 없이 동작
- Redis 명령이 투명하게 보여서 장애 시 MONITOR, slowlog로 추적하기 쉬움

## Redisson을 쓰는 경우 (특정 기능이 필요할 때)

존재 이유는 저수준 명령이 아니라 **직접 구현하면 틀리기 쉬운 분산 패턴**.

- **분산 락 (`RLock`)** — 가장 흔한 도입 이유
    - 직접 구현 시: SETNX + 만료시간 + 해제 시 소유자 확인(Lua) + 대기 폴링 → 하나만 빼먹어도 데드락, 남의 락 해제 버그
    - Redisson 제공: pub/sub 기반 대기(폴링 없음), watchdog 락 자동 연장, 재진입 락, 공정 락
- RateLimiter, 분산 세마포어, RMap 등 분산 자료구조가 필요할 때
- 단점: GET/SET 같은 저수준 명령 API는 상대적으로 불편

## 실무의 흔한 정답: 둘 다 사용

충돌 없이 공존 가능하며, 실제로 가장 많은 패턴.

```
캐시, 세션, 일반 데이터  → Lettuce (Spring 기본, RedisTemplate)
분산 락, 중복 실행 방지   → Redisson (RLock만 사용)
```

예시: 다중 인스턴스 스케줄러에서 배치 작업 1회 실행 보장, 재고 차감 같은 동시성 제어 지점에만 Redisson을 붙이고 나머지는 Lettuce.

## 참고: Jedis는?

- 동기(blocking) 방식, 인스턴스가 스레드 세이프하지 않아 커넥션 풀(JedisPool) 필수
- 동시 요청 증가 시 풀 고갈이 병목
- 레거시 호환이 아니면 신규 프로젝트에서 선택할 이유가 거의 없음

## 판단 플로우

1. Redis 용도가 캐시/세션/단순 조회? → **Lettuce만** (Spring Boot면 이미 기본값)
2. 분산 락, 중복 실행 방지, RateLimiter가 필요해졌다? → **Redisson을 락 용도로만 추가**
3. 온프레미스 제품이라면 의존성 추가 자체가 비용이므로, 필요가 생기기 전에 미리 넣지 않는다