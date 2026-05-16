---
title: "Read Replica 환경에서의 동시성 문제 해결하기"
description: "Read Replica 환경에서 Replication Lag 때문에 생기는 동시성 문제를 살펴보고, Writer 재조회, 분산 락, SETNX, 캐시 조합을 트래픽 규모에 따라 어떻게 선택할지 정리합니다."
date: 2026-02-13T21:40:00+09:00
tags: ["database", "redis", "concurrency", "read-replica", "distributed-lock"]
---

## 들어가며

읽기 부하를 분산하려고 Read Replica를 두는 구조는 흔합니다. 그런데 이 구조에서 동시성 제어가 필요한 기능을 만들면 **Replication Lag**이 생각보다 쉽게 문제를 일으킵니다.

이 글에서는 "유저당 하루에 한 번만 허용되는 참여" 같은 기능을 예로 들어, 어떤 문제가 생기고 트래픽 규모에 따라 어떤 방식으로 풀 수 있는지 정리합니다.

---

## 문제 상황

### 기본 구조

아래 흐름의 API가 있다고 가정해 보겠습니다.

```
1. 오늘 이 유저가 이미 참여했는지 DB에서 조회
2. 참여 이력이 없으면 → 새로운 이력을 생성하여 저장
3. 참여 이력이 있으면 → 기존 이력을 기반으로 응답
```

중복 참여를 방지하기 위해, 이력 테이블에 `anti_duplicated_key`라는 unique constraint를 걸어두었습니다.

```sql
ALTER TABLE participation_history
    ADD CONSTRAINT uk_anti_duplicated_key UNIQUE (anti_duplicated_key);
```

### 동시 요청 시 발생하는 문제

같은 유저가 거의 동시에 두 번 요청을 보내면 이런 일이 벌어질 수 있습니다.

```
요청 A ── 조회 (없음) ── 저장 (성공) ──────────────── 정상 응답
요청 B ── 조회 (없음) ── 저장 (실패: Duplicate Key) ── ???
```

두 요청 모두 "조회" 시점에는 이력이 없다고 판단했기 때문에 저장을 시도합니다. 요청 A가 먼저 성공하면, 요청 B는 unique constraint 위반으로 실패합니다.

전형적인 **TOCTOU(Time of Check, Time of Use)** 문제입니다. 조회와 저장이 하나의 원자적 작업이 아니기 때문에 생깁니다.

### 가장 직관적인 해결: try-catch로 재조회

```kotlin
try {
    val history = ParticipationHistory(userId = userId, ...)
    commandService.save(history)
    return toResponse(history)
} catch (e: DataIntegrityViolationException) {
    // 다른 요청이 먼저 저장했으니, 그 데이터를 조회해서 반환
    val existing = queryService.findByUserId(userId)
        ?: throw e
    return toResponse(existing)
}
```

간단하고 직관적입니다. 다만 **Read Replica**를 쓰고 있다면 여기서 또 다른 문제가 생깁니다.

---

## Read Replica의 함정: Replication Lag

### Writer/Reader 라우팅 구조

Writer와 Read Replica를 함께 쓰면, 쿼리 성격에 따라 어느 DB로 보낼지 정하는 라우팅이 필요합니다. 구현 방식은 인프라에 따라 다릅니다.

- **Aurora MySQL**: JDBC 드라이버가 커넥션의 `readOnly` 플래그를 감지하여 Writer/Reader 엔드포인트로 자동 라우팅
- **일반 MySQL Read Replica**: `AbstractRoutingDataSource`로 DataSource를 분리하거나, ProxySQL 같은 미들웨어에서 라우팅
- **공통 패턴**: 트랜잭션의 `readOnly` 속성을 라우팅 기준으로 활용

```kotlin
@Transactional(readOnly = true)   // → Read Replica로 라우팅
class QueryService { ... }

@Transactional(readOnly = false)  // → Writer(Primary)로 라우팅
class CommandService { ... }
```

아래 예제는 이 패턴을 기준으로 설명합니다.

### 재조회가 실패하는 이유

요청 B의 catch 블록에서 `QueryService`로 재조회하면 쿼리는 **Read Replica**로 갑니다. 하지만 요청 A가 Writer에 커밋한 데이터가 아직 Replica에 복제되지 않았을 수 있습니다.

```
요청 A ──── Writer에 저장 (커밋 완료)
                   │
                   ├── Replica로 복제 중... (수 ms ~ 수백 ms)
                   │
요청 B ──── Replica에서 조회 → 아직 없음! → null 반환 → 응답 실패
```

이 시간차가 **Replication Lag**입니다. 보통은 수 밀리초 수준이지만, 부하가 높을 때는 수백 밀리초까지 늘어날 수 있습니다.

---

## 해결 방법 1: Writer DB에서 직접 조회 (빠른 해결)

가장 단순한 해결책은 **catch 블록에서의 재조회만 Writer DB로 보내는 것**입니다.

```kotlin
@Transactional(readOnly = false)
class CommandService {

    fun save(history: ParticipationHistory) = repository.save(history)

    // 동시성 예외 후 재조회용: Writer에서 직접 읽기
    fun findByUserIdFromWriter(userId: String) =
        repository.findByUserIdAndCreatedDate(userId, LocalDate.now())
}
```

```kotlin
try {
    commandService.save(history)
    return toResponse(history)
} catch (e: DataIntegrityViolationException) {
    val existing = commandService.findByUserIdFromWriter(userId)
        ?: throw e  // 동시성 이슈가 아닌 다른 제약 조건 위반이면 예외 전파
    return toResponse(existing)
}
```

### 핵심 포인트

- **catch 대상을 `DataIntegrityViolationException`으로 한정합니다.** 모든 `Exception`을 catch하면 동시성 문제가 아닌 예외까지 삼켜버립니다.
- **Writer에서 조회한 결과가 null이면 원래 예외를 rethrow합니다.** `DataIntegrityViolationException`이 발생했는데 Writer에도 데이터가 없다면, unique constraint 위반이 아닌 다른 제약 조건(NOT NULL, FK 등) 문제일 수 있습니다.
- **정상 흐름에서는 여전히 Read Replica를 사용합니다.** Writer 직접 조회는 동시성 예외가 발생한 예외적인 경우에만 타므로, Writer에 불필요한 읽기 부하가 가지 않습니다.

### 주의: 트랜잭션 경계

`save()` 호출 시 `DataIntegrityViolationException`이 발생하면, **해당 트랜잭션은 rollback-only로 마킹**됩니다. 만약 호출자(caller)가 자체 `@Transactional`로 감싸져 있다면, catch 이후의 `findByUserIdFromWriter()` 호출이 같은 트랜잭션에 참여하게 되고, 이미 rollback-only 상태이므로 `UnexpectedRollbackException`이 발생합니다.

따라서 **호출자는 `@Transactional`을 갖지 않아야** 하며, `save()`와 `findByUserIdFromWriter()`가 각각 독립된 트랜잭션에서 실행되도록 해야 합니다.

### 이 방법의 한계

트래픽이 매우 높은 서비스에서는 다음과 같은 문제가 있습니다.

- 동시 요청 N개 중 N-1개가 **불필요한 Write를 시도한 후 실패**합니다.
- 실패한 요청들이 **Writer DB에 조회 요청**을 보냅니다.
- DB unique constraint가 사실상 **동시성 제어 수단**으로 사용되고 있습니다.

---

## 해결 방법 2: 분산 락으로 임계 구역 보호

조회와 저장 전체를 하나의 임계 구역으로 묶으면, TOCTOU 문제가 원천적으로 해결됩니다.

```
요청 A ── Lock 획득 ── 조회 (없음) ── 저장 ── Lock 해제
요청 B ── Lock 대기... ── Lock 획득 ── 조회 (있음) ── 결과 반환 ── Lock 해제
```

### 분산 락의 실체: SETNX + 재시도 + 안전한 해제

Redis에는 "Lock"이라는 명령어가 없습니다. 분산 락은 `SET NX EX`를 기반으로 **애플리케이션 코드에서 구현**하는 패턴입니다.

```
Lock 획득 = SET lockKey uniqueValue NX EX 30   (키가 없을 때만 설정, 30초 TTL)
Lock 해제 = Lua 스크립트로 값 비교 후 DEL       (본인이 건 락만 해제)
```

이를 코드로 풀어쓰면 다음과 같습니다.

```kotlin
fun <T> withLock(
    key: String,
    lockTimeout: Duration,    // 락 보유 시간 (TTL)
    waitTimeout: Duration,    // 락 획득 대기 시간
    action: () -> T,
): T {
    val lockValue = UUID.randomUUID().toString()
    val deadline = System.currentTimeMillis() + waitTimeout.toMillis()

    // 1. 락 획득 시도 (SETNX + TTL) - 대기 시간 내 재시도
    while (System.currentTimeMillis() < deadline) {
        val acquired = redis.set(key, lockValue, SetParams().nx().px(lockTimeout.toMillis()))
        if (acquired != null) break
        Thread.sleep(50)
    } ?: throw LockAcquisitionTimeoutException("락 획득 타임아웃: $key")

    try {
        // 2. 임계 구역 실행
        return action()
    } finally {
        // 3. 락 해제 (Lua 스크립트로 본인 락만 해제)
        redis.eval("""
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        """, listOf(key), listOf(lockValue))
    }
}
```

위 코드는 분산 락의 구조를 이해하기 위한 것이며, 실무에서는 Redisson 등의 라이브러리를 사용하는 것을 권장합니다. Redisson의 `RLock`은 락 재진입(reentrant), Pub/Sub 기반 대기(스핀 락 대신), watchdog을 통한 TTL 자동 연장 등을 지원합니다.

### 락 내부에서의 조회: Writer에서 읽어야 하는 이유

분산 락을 사용하더라도, 락 내부의 조회가 Read Replica로 가면 Replication Lag 문제가 재발할 수 있습니다.

```
요청 A ── Lock 획득 ── Writer에 저장 ── Lock 해제
                                           │ Replica 복제 지연
요청 B ──────────────── Lock 획득 ── Replica 조회 (null!) ── 중복 저장 시도
```

요청 B가 락을 획득한 시점에 요청 A의 커밋은 완료되었지만, Read Replica에는 아직 반영되지 않았을 수 있습니다. 따라서 **락 내부에서의 조회는 반드시 Writer에서 수행**해야 합니다.

```kotlin
fun participate(userId: String): Response {
    val lockKey = "PARTICIPATION:${LocalDate.now()}:$userId"

    return withLock(lockKey, lockTimeout = 5.seconds, waitTimeout = 3.seconds) {
        // Writer에서 조회하여 Replication Lag 회피
        val existing = commandService.findByUserIdFromWriter(userId)
        if (existing != null) {
            return@withLock toResponse(existing)
        }
        val history = ParticipationHistory(userId = userId, ...)
        commandService.save(history)
        toResponse(history)
    }
}
```

### 장점

- 동시 요청 중 **단 하나만 Write를 시도**합니다.
- Writer DB에 불필요한 부하가 가지 않습니다.
- 로직이 직관적이고, 동시성 제어 의도가 코드에 명확히 드러납니다.

### 단점

- Redis 등 외부 인프라 의존성이 추가됩니다.
- Lock 획득 대기로 인한 응답 지연이 발생할 수 있습니다.
- Lock 해제 실패 시 처리가 필요합니다 (TTL로 보완).

---

## 해결 방법 3: SETNX로 선착순 결정 (대기 없는 방식)

해결 방법 2와 동일하게 `SET NX`를 사용하지만, **획득 실패 시 대기하지 않고 즉시 다른 분기로 이동**한다는 점이 다릅니다.

```
요청 A ── SETNX 성공 (처리 담당) ── 저장 ── 응답
요청 B ── SETNX 실패 (조회 담당) ── DB 조회 ── 응답
```

```kotlin
fun participate(userId: String): Response {
    val redisKey = "PARTICIPATION:${LocalDate.now()}:$userId"
    val isFirst = redis.setIfAbsent(redisKey, "PROCESSING", Duration.ofSeconds(30))

    return if (isFirst == true) {
        handleFirstRequest(userId, redisKey)
    } else {
        handleSubsequentRequest(userId)
    }
}

private fun handleFirstRequest(userId: String, redisKey: String): Response {
    try {
        val history = ParticipationHistory(userId = userId, ...)
        commandService.save(history)
        // 저장 성공 시 Redis 값을 결과로 갱신하여 후발 요청이 활용할 수 있게 함
        redis.set(redisKey, history.id.toString(), Duration.ofDays(1))
        return toResponse(history)
    } catch (e: Exception) {
        // 저장 실패 시 Redis 키를 삭제하여 다른 요청이 재시도할 수 있게 함
        redis.delete(redisKey)
        throw e
    }
}

private fun handleSubsequentRequest(userId: String): Response {
    // 첫 요청의 DB 저장이 완료되기까지 약간의 시간이 필요할 수 있음
    // Writer에서 조회하되, 짧은 간격으로 재시도
    repeat(3) { attempt ->
        val existing = commandService.findByUserIdFromWriter(userId)
        if (existing != null) return toResponse(existing)
        Thread.sleep(50L * (attempt + 1))
    }
    throw IllegalStateException("참여 이력 조회 실패: $userId")
}
```

### 장점

- Lock 대기 없이 **즉시 분기**하므로 응답이 빠릅니다.
- DB에 Write 시도 자체가 **1번만** 발생합니다.

### 단점

- **첫 요청 실패 시 Redis 키 롤백**이 반드시 필요합니다. 롤백하지 않으면 Redis에는 키가 남아있고 DB에는 데이터가 없는 상태가 되어, 이후 모든 요청이 "이미 처리됨"으로 판단하지만 실제 데이터는 존재하지 않는 정합성 문제가 발생합니다.
- 후발 요청은 첫 요청의 DB 저장이 완료될 때까지 기다려야 하므로, **재시도 로직의 설계**가 필요합니다.
- Redis와 DB 간 **이중 상태 관리**에 따른 운영 복잡도가 증가합니다.

---

## 해결 방법 4: 조합 - 캐시 + 분산 락 + DB Constraint

트래픽이 가장 높은 환경에서는 여러 레이어를 조합합니다.

```
요청 → Redis 캐시 조회 ── hit → 즉시 응답 (DB 접근 없음)
              │
            miss
              ↓
         분산 락 획득
              ↓
         Writer DB 조회 ── 있으면 → 캐싱 후 응답
              │
            없음
              ↓
         저장 → Redis 캐싱 → 응답
```

| 레이어 | 역할 | 처리하는 트래픽 |
|--------|------|-----------------|
| **Redis 캐시** | 동일 유저의 반복 조회 흡수 | 대부분 (90%+) |
| **분산 락** | 캐시 miss 시 동시 요청 직렬화 | 소수 |
| **DB unique constraint** | 최후의 안전망 | 거의 없음 (방어 목적) |

---

## Redis 분산 락은 완벽한가?

Redis 기반 분산 락을 도입한다면, 그 한계도 정확히 인지해야 합니다.

### SETNX의 원자성

`SETNX`(`SET NX`)는 시스템 시간과 무관한 **순수한 atomic compare-and-set 연산**입니다. Redis Cluster에서도 하나의 키는 항상 동일한 hash slot의 동일한 master 노드에서 처리되므로, 단일 키에 대한 SETNX의 원자성은 보장됩니다.

시간이 관련되는 것은 SETNX 자체가 아니라 **TTL(만료 시간)** 과 **Redlock 알고리즘**입니다.

### 진짜 위험: 비동기 복제와 Failover

Redis Cluster에서 분산 락이 깨질 수 있는 시나리오는 다음과 같습니다.

```
Client A ── SETNX lock (성공) ────────────────→ Master Node
                                                    │
                                               복제 전에 장애 발생
                                                    ↓
Client B ── SETNX lock (성공!) ──────────────→ Replica → Master 승격
```

1. Client A가 Master에서 락 획득
2. Master가 Replica에 **복제하기 전에** 장애 발생
3. Replica가 Master로 승격 → 락 정보 유실
4. Client B가 동일 키로 락 획득 성공 → **두 클라이언트가 동시에 락 보유**

이 문제는 Redis의 복제가 **비동기(asynchronous)** 로 이루어지기 때문에 구조적으로 발생합니다.

### Redlock: 해결 시도와 그에 대한 논쟁

이 문제를 해결하기 위해 Redis 창시자 Antirez가 제안한 **Redlock 알고리즘**은, **독립된 N개의 Redis 인스턴스**(Cluster가 아닌)에서 과반수 이상 락을 획득해야 유효하다고 판단합니다.

이에 대해 Martin Kleppmann은 ["How to do distributed locking"](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)에서 두 가지 핵심 문제를 제기했습니다.

1. **프로세스 정지 문제**: 락을 획득한 프로세스가 GC pause, 네트워크 지연 등으로 멈추면, 락의 TTL이 만료된 후에도 본인이 아직 락을 보유하고 있다고 착각할 수 있습니다.
2. **Fencing token의 부재**: 이러한 상황을 안전하게 처리하려면, 단순한 락이 아니라 단조 증가하는 fencing token을 함께 사용해야 합니다. 하지만 Redlock은 이를 제공하지 않습니다.

Antirez는 ["Is Redlock safe?"](http://antirez.com/news/101)에서, Redlock은 클록 점프가 제한된 환경에서 충분히 안전하다고 반박했습니다. 이 논쟁은 분산 시스템의 근본적인 한계를 보여주는 사례로, 양측 모두 읽어볼 가치가 있습니다.

### 현실적인 접근: 방어 레이어 조합

완벽한 분산 락은 이론적으로 매우 어렵습니다. 하지만 우리에게 필요한 것은 "완벽한 상호 배제"가 아니라 **"중복 저장 방지"** 입니다. 방어 레이어를 겹치면 실용적으로 충분한 안전성을 확보할 수 있습니다.

| 방어 레이어 | 역할 | 실패 확률 |
|---|---|---|
| **Redis 분산 락** | 1차 방어 - 대부분의 동시 요청 차단 | 매우 낮음 (Master 장애 시에만) |
| **DB unique constraint** | 2차 방어 - Redis가 뚫려도 최종 차단 | 사실상 0 |

Redis 락은 "완벽한 상호 배제"가 아니라 **불필요한 DB 부하를 줄이기 위한 최적화 수단**으로 보는 것이 적절합니다. 데이터 정합성의 최종 책임은 DB의 unique constraint가 가지고 있습니다.

---

## Strong Consistency vs Eventual Consistency

지금까지의 해결 방법은 모두 **동기적 흐름에서 즉시 결과를 반환**하는 Strong Consistency 접근입니다. 하지만 요구사항에 따라 Eventual Consistency도 선택지가 됩니다.

### 결과를 즉시 보여줘야 하는 경우 (Strong Consistency)

```
요청 → 분산 락 → 처리 → 결과 응답 (동기)
```

버튼을 누르면 바로 결과를 보여줘야 하는 UX라면, 동기적 흐름 안에서 정합성을 보장해야 합니다. 앞서 다룬 Redis 분산 락 + DB constraint 조합이 적합합니다.

### 결과를 나중에 확인해도 되는 경우 (Eventual Consistency)

```
요청 → MQ에 메시지 발행 → "참여 완료" 즉시 응답
               ↓
        Consumer가 순차 처리 (단건씩)
               ↓
        다음 조회 시 결과 확인
```

예를 들어 Kafka를 사용한다면, `userId`를 파티션 키로 사용하면 동일 유저의 요청이 항상 같은 파티션에서 단일 컨슈머에 의해 순차 처리됩니다. 동시성 문제가 원천적으로 사라집니다.

| | Strong Consistency | Eventual Consistency |
|---|---|---|
| **응답** | 즉시 결과 반환 | "참여 완료" 후 나중에 결과 확인 |
| **동시성 제어** | 분산 락 + DB constraint | MQ 파티션 순서 보장 |
| **복잡도** | 락 관리, 타임아웃 처리 | 비동기 흐름, 상태 관리 |
| **장애 영향** | Redis 장애 시 즉시 영향 | Consumer 지연 시 결과 확인 지연 |
| **적합한 경우** | 결과를 즉시 보여줘야 할 때 | 참여 자체가 중요하고 결과는 나중에 봐도 될 때 |

---

## 정리: 트래픽 규모별 전략

| 단계 | 전략 | 적합한 상황 |
|------|------|-------------|
| **1단계** | try-catch + Writer DB 재조회 | 동시 요청이 간헐적으로 발생하는 수준 |
| **2단계** | 분산 락 (Redis) + DB constraint | 동시 요청이 빈번하고 Writer 부하가 우려될 때 |
| **3단계** | Redis 캐시 + 분산 락 + DB constraint | 대규모 트래픽에서 DB 접근 자체를 최소화해야 할 때 |
| **별도** | MQ 기반 비동기 처리 | 즉시 결과 반환이 필요 없는 경우 |

정답은 **현재 트래픽 규모, 인프라 구성, UX 요구사항**에 따라 달라집니다. 처음부터 복잡한 구조를 들이기보다, 각 방법의 트레이드오프를 알고 필요한 만큼만 선택하는 편이 낫습니다. 동시 요청이 가끔 발생하는 수준이라면 Writer 재조회로 충분할 수 있고, 실제 트래픽이 그 한계를 밀어낼 때 다음 단계로 넘어가도 늦지 않습니다.

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
