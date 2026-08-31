# Deadlock vs Livelock
> 교착 상태(Deadlock)와 라이브락(Livelock) 둘 다 멈춘 건데 뭐가 다른가?!!

### 핵심은 '상태'(State)와 자원(Resource)의 관점에서 설명해야 함.

## 쉬운 비유

### Deadlock (교착 상태)
- 골목길 정면 충돌
- 서로 마주보고 멈춤
- 아무도 움직이지 않음 (정지)
- 상태 : Blocked / Waiting
- CPU 사용률 : 거의 없음 (idle)
- 자원을 얻지 못해 대기 상태에서 영원히 깨어나지 못함

### Livelock (라이브락)
- 복도 댄스 (무한 양보)
- 서로 비켜주려다 계속 부딪힘
- 계속 움직이지만 진정 없음
- 상태 : Running
- CPU 사용률 : 높음
- 실패한 작업을 계속 재시도(Retry)하거나 상태를 변경하느라 바쁨 (진전은 없음)


## Code Example

### Deadlock
```java
Object lockA = new Object();
Object lockB = new Object();

// Thread 1 : A → B 순서로 락 획득 시도
new Thread(() -> {
    synchronized (lockA) {
        System.out.println("T1: lockA 획득");
        sleep(100); // T2가 lockB를 잡을 시간을 벌어줌
        synchronized (lockB) { // T2가 lockB를 쥐고 있어서 영원히 대기
            System.out.println("T1: lockB 획득");
        }
    }
}).start();

// Thread 2 : B → A 순서로 락 획득 시도 (순서가 반대!)
new Thread(() -> {
    synchronized (lockB) {
        System.out.println("T2: lockB 획득");
        sleep(100);
        synchronized (lockA) { // T1이 lockA를 쥐고 있어서 영원히 대기
            System.out.println("T2: lockA 획득");
        }
    }
}).start();
```

결과 : 둘 다 멈춰서 대기함 (서로 상대방이 쥔 락을 기다리며 BLOCKED 상태)

### Livelock (무한 재시도)
```java
Lock lockA = new ReentrantLock();
Lock lockB = new ReentrantLock();

// Thread 1 : A 잡고 → B 시도, 실패하면 A를 놓고 재시도 (양보)
new Thread(() -> {
    while (true) {
        if (lockA.tryLock()) {
            try {
                if (lockB.tryLock()) { // B 획득 실패 → 양보
                    try { doWork(); break; } finally { lockB.unlock(); }
                }
            } finally {
                lockA.unlock(); // A를 놓아줌 (양보)
            }
        }
        // 대기 없이 곧바로 재시도 → 문제의 원인
    }
}).start();

// Thread 2 : B 잡고 → A 시도, 실패하면 B를 놓고 재시도 (양보)
new Thread(() -> {
    while (true) {
        if (lockB.tryLock()) {
            try {
                if (lockA.tryLock()) { // A 획득 실패 → 양보
                    try { doWork(); break; } finally { lockA.unlock(); }
                }
            } finally {
                lockB.unlock(); // B를 놓아줌 (양보)
            }
        }
    }
}).start();
```

결과 : 획득/해제 타이밍이 완벽히 겹침 (둘 다 놨다가, 둘 다 다시 잡는 것을 무한 반복)
- 스레드 상태는 RUNNABLE이라 겉보기엔 "동작 중"이지만 실제 진전(doWork)은 없음
- 해결 : 재시도 전에 `Thread.sleep(random.nextInt(100))` 같은 **Random Backoff**를 넣으면 타이밍이 어긋나면서 한쪽이 두 락을 모두 잡을 수 있게 됨


## 한눈에 비교

| 구분 | Deadlock | Livelock |
|---|---|---|
| 스레드 상태 | BLOCKED / WAITING | RUNNABLE (실행 중) |
| CPU 사용률 | 거의 0 (idle) | 높음 (바쁜 대기) |
| 겉보기 | 완전히 멈춤 | 동작하는 것처럼 보임 |
| 원인 | 자원을 쥔 채 서로를 기다림 | 서로 양보/재시도가 무한 반복 |
| 탐지 난이도 | 상대적으로 쉬움 (스레드 덤프에 명확히 표시) | 어려움 (실행 중이라 정상처럼 보임) |
| 대표 해결책 | 락 순서 고정, 타임아웃, 감지 후 롤백 | Random Backoff, 재시도 횟수 제한 |


## 데드락 발생의 4가지 조건 (Coffman Conditions)
> 4가지가 **모두** 성립해야 데드락이 발생함. 즉, 하나라도 깨면 데드락을 예방할 수 있다.

1. **상호 배제 (Mutual Exclusion)** : 자원은 한 번에 하나의 프로세스만 사용 가능
2. **점유와 대기 (Hold and Wait)** : 자원을 쥔 채로 다른 자원을 기다림
3. **비선점 (No Preemption)** : 다른 프로세스가 쥔 자원을 강제로 빼앗을 수 없음
4. **순환 대기 (Circular Wait)** : P1→P2→P3→P1처럼 자원 대기가 원형을 이룸

### 각 조건을 깨는 예방법
| 깨는 조건 | 방법 | 현실성 |
|---|---|---|
| 상호 배제 | 락 없는 자료구조 (lock-free, CAS) | 제한적 (모든 자원에 적용 불가) |
| 점유와 대기 | 필요한 락을 한 번에 전부 획득 | 자원 활용률 저하 |
| 비선점 | 타임아웃 후 락 반납 (`tryLock(timeout)`) | 실용적 |
| 순환 대기 | **락 획득 순서를 전역적으로 고정** | 가장 널리 쓰이는 방법 ⭐ |


## 해결방법
### Deadlock 해결
1. 자원 순서 정렬 (순서대로만 락 획득) → 순환 대기 조건을 깸
2. 교착 상태 감지 후 하나 강제 종료 (DB의 데드락 감지 + victim 롤백)
3. 타임아웃 (일정 시간 후 포기) → `tryLock(1, TimeUnit.SECONDS)`

### Livelock 해결
1. Random Backoff (임의 시간 대기) → 이더넷 CSMA/CD 충돌 처리와 같은 원리
2. 재시도 횟수 제한 (초과 시 실패 처리 or 에러 전파)
3. 우선순위/중재자 도입 (한쪽에게 우선권을 줘서 대칭성을 깸)


## DB 관점에서의 데드락 (실무에서 훨씬 자주 만남)
> 애플리케이션 코드보다 **DB 트랜잭션 간 데드락**이 실무에서 더 흔하다.

### 발생 예시 (MySQL InnoDB)
```sql
-- Tx1                                  -- Tx2
BEGIN;
UPDATE users SET ... WHERE id = 1;      BEGIN;
                                        UPDATE users SET ... WHERE id = 2;
UPDATE users SET ... WHERE id = 2;      -- Tx1이 id=2 락 대기
                                        UPDATE users SET ... WHERE id = 1;
                                        -- 순환 대기 → InnoDB가 데드락 감지!
-- ERROR 1213: Deadlock found when trying to get lock; try restarting transaction
```

### DB는 데드락을 어떻게 처리하나?
- InnoDB는 **대기 그래프(wait-for graph)** 로 데드락을 자동 감지하고, 롤백 비용이 적은 트랜잭션을 **victim**으로 골라 즉시 롤백시킴 (에러 1213)
- 감지가 꺼져 있거나 감지 못하는 경우 `innodb_lock_wait_timeout` (기본 50초) 이후 타임아웃 에러 발생
- 확인 방법 : `SHOW ENGINE INNODB STATUS;` 의 `LATEST DETECTED DEADLOCK` 섹션

### 애플리케이션에서의 대응
1. 여러 행을 갱신할 때 **항상 같은 순서**(예: PK 오름차순)로 접근
2. 트랜잭션을 짧게 유지 (락 보유 시간 최소화)
3. 데드락 에러(1213)는 재시도 가능한 에러 → **재시도 로직** 구현 (Spring의 `@Retryable` 등)
4. 필요 이상으로 넓은 범위의 락(갭 락, 테이블 락)을 유발하는 쿼리 주의 → 인덱스 없는 UPDATE는 스캔한 모든 행에 락


## Starvation (기아 상태) 와의 비교
> 면접에서 데드락/라이브락과 함께 자주 묶여 나오는 개념

- **Starvation** : 특정 프로세스만 자원을 계속 할당받지 못하는 상태. 시스템 전체는 진전이 있지만, **불운한 일부**만 영원히 대기함 (예: 우선순위 낮은 스레드가 계속 밀림)
- Deadlock/Livelock은 관련된 **모두**가 진전이 없고, Starvation은 **일부**만 진전이 없다는 점이 핵심 차이
- 해결 : Aging (대기 시간이 길수록 우선순위를 점차 올려줌), 공정 락 (`new ReentrantLock(true)`)


## 데드락 탐지 방법 (실무 디버깅)
- **Java** : `jstack <pid>` 스레드 덤프 → `Found one Java-level deadlock` 섹션 자동 표시. IDE나 VisualVM으로도 확인 가능
- **Livelock** : 스레드 덤프에는 안 잡힘 (RUNNABLE이므로). CPU는 높은데 처리량(throughput)이 0에 가까우면 의심 → 프로파일링으로 같은 코드 반복 실행 확인
- **MySQL** : `SHOW ENGINE INNODB STATUS;`, `performance_schema.data_lock_waits` 조회


## 정리
1. 데드락은 프로세스들이 자원을 점유한 채 서로를 기다리며 대기(Waiting) 상태로 멈춰있는 것.
2. 라이브락은 프로세스들이 서로 양보하거나 상태를 바꾸느라 실행(Running) 중이지만, 작업은 진행되지 않은 상태.
3. 데드락은 (해당 스레드가) 대기 상태라 CPU 소모가 거의 없지만, 라이브락은 무한 재시도로 CPU를 많이 쓸 수 있다.

## 꼬리에 꼬리를 무는 질문
1. **데드락 발생의 4가지 조건은? (Coffman Conditions)**
   → 상호 배제, 점유와 대기, 비선점, 순환 대기. 4가지 모두 성립해야 발생하며 하나만 깨도 예방 가능 (위 섹션 참고)

2. **식사하는 철학자 문제(Dining Philosophers)란?**
   → 원탁에 철학자 5명, 포크 5개. 식사하려면 양옆 포크 2개가 필요한데, 모두가 동시에 왼쪽 포크를 집으면 오른쪽 포크를 기다리며 순환 대기 → 데드락.
   대표 해법 : ① 한 명만 오른쪽 포크부터 집기 (락 순서 비대칭화), ② 최대 4명만 착석 (자원보다 적은 동시 접근), ③ 포크 2개를 원자적으로 획득

3. **실제 시스템에서 라이브락은 언제 발생하나요?**
   → ① DB 트랜잭션 데드락 감지 후 양쪽 모두 즉시 재시도를 반복할 때 (backoff 없는 재시도), ② 낙관적 락(Optimistic Lock) 충돌이 잦은 hot row에서 CAS 재시도가 계속 실패할 때, ③ 네트워크 패킷 충돌 후 동시 재전송 (그래서 이더넷은 random backoff 사용)

4. **데드락 처리 전략 3가지(예방/회피/탐지 후 회복)의 차이는?**
   → 예방(Prevention)은 Coffman 조건 자체를 깨는 것, 회피(Avoidance)는 은행원 알고리즘처럼 안전 상태를 유지하며 할당, 탐지 후 회복(Detection & Recovery)은 발생을 허용하되 감지해서 victim을 롤백 (InnoDB 방식)

5. **낙관적 락 vs 비관적 락, 데드락 관점에서 어떤 차이가 있나?**
   → 비관적 락은 락을 오래 쥐므로 데드락 위험이 있고, 낙관적 락은 락을 쥐지 않아 데드락은 없지만 충돌이 잦으면 재시도 반복(라이브락과 유사한 양상)이 발생할 수 있다

