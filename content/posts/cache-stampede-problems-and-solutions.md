---
title: "Cache Stampede가 만드는 장애와 해결 전략"
description: "캐시 만료 직후 같은 키의 요청이 외부 API로 몰릴 때 어떤 문제가 생기는지, Resilience4j HALF_OPEN 사례와 Caffeine single-flight, refresh-ahead, TTL jitter, Redis 분산 락 전략을 함께 정리합니다."
date: 2026-06-25T17:34:18+09:00
url: "/cache-stampede-problems-and-solutions/"
tags: ["cache", "circuit-breaker", "resilience4j", "caffeine", "performance"]
---

캐시는 외부 API 호출을 줄이고 응답 시간을 낮추는 가장 익숙한 도구입니다. 하지만 캐시가 있다고 해서 항상 외부 시스템이 보호되는 것은 아닙니다. 캐시가 비는 순간에 요청이 한꺼번에 몰리면, 평소에는 숨어 있던 병목이 아주 짧은 시간에 터집니다.

이 글은 실제 운영에서 겪은 사례를 바탕으로, **cache stampede가 어떤 장애로 이어질 수 있는지**와 **상황별로 어떤 해결 전략을 선택할 수 있는지**를 정리한 글입니다. Resilience4j 서킷 브레이커가 HALF_OPEN 상태에 오래 머문 문제는 이 글의 출발점이지만, 핵심은 특정 라이브러리 설정 하나가 아니라 캐시 만료 직후의 트래픽 모양입니다.

---

## Cache Stampede란

**Cache stampede**는 같은 캐시 키가 만료되었을 때 여러 요청이 동시에 cache miss를 맞고, 모두 원본 저장소나 외부 API로 몰려가는 현상입니다. "thundering herd"라고도 부릅니다.

평소에는 캐시 적중률이 높아서 외부 API 호출이 거의 없다가, 특정 순간에만 호출이 몰리는 패턴이 됩니다.

```text
평소:   요청 1000개 → 캐시 hit 999개 → 외부 API 1개
만료 직후: 요청 1000개 → 캐시 miss 1000개 → 외부 API 1000개
```

이 문제는 다음 조건이 겹칠 때 특히 잘 드러납니다.

- 같은 키를 많은 요청이 읽는다.
- 여러 인스턴스가 같은 TTL 정책을 사용한다.
- 캐시 키가 정각, 시간 슬롯, 배치 주기처럼 같은 시점에 바뀐다.
- 외부 API timeout이 짧거나, 외부 API가 순간 부하에 약하다.
- fallback이나 서킷 브레이커가 있어 장애가 즉시 드러나지 않는다.

중요한 점은 cache stampede가 단순히 "캐시 miss가 늘어난다"에서 끝나지 않는다는 것입니다. 외부 API timeout, thread 점유, fallback 응답 증가, 서킷 브레이커 OPEN, 그리고 회복 지연까지 이어질 수 있습니다.

## 실제로 발생한 문제

문제가 된 서비스는 외부 API 호출을 로컬 캐시(Caffeine)로 감싸고, 그 앞에 Resilience4j 서킷 브레이커를 둔 구조였습니다.

```text
요청 → 캐시 조회 → cache miss → 서킷 브레이커 → 외부 API
```

운영 중 일부 인스턴스에서 다음 현상이 보였습니다.

> 외부 API는 이미 회복되었는데, 일부 인스턴스의 서킷 브레이커가 HALF_OPEN 상태에서 정상으로 돌아오지 않는다.

서킷 설정은 다음과 같았습니다.

```yaml
slidingWindowType: COUNT_BASED
slidingWindowSize: 100
minimumNumberOfCalls: 20
waitDurationInOpenState: 10s
permittedNumberOfCallsInHalfOpenState: 5
```

각 항목의 의미는 다음과 같습니다.

- `slidingWindowType: COUNT_BASED`: 최근 호출 결과를 개수 기준으로 집계한다. `TIME_BASED`는 최근 N초 기준으로 집계한다.
- `slidingWindowSize: 100`: 최근 100건의 결과를 보관한다.
- `minimumNumberOfCalls: 20`: 최소 20건이 쌓여야 실패율이나 slow call 비율을 계산한다.
- `waitDurationInOpenState: 10s`: OPEN 상태에서 최소 10초를 기다린 뒤 HALF_OPEN으로 전환될 수 있다.
- `permittedNumberOfCallsInHalfOpenState: 5`: HALF_OPEN 상태에서 제한적으로 통과시킬 호출 수다.

여기서 한 가지 주의할 점이 있습니다. Resilience4j의 기본 설정에서는 `automaticTransitionFromOpenToHalfOpenEnabled`가 `false`입니다. 이 경우 10초가 지났다고 백그라운드에서 자동으로 HALF_OPEN이 되는 것이 아니라, **10초가 지난 뒤 들어온 호출이 상태 전환을 유발**합니다. 자동 전환을 켜면 별도 모니터링 스레드가 시간이 지난 서킷을 HALF_OPEN으로 전환합니다.

## 왜 HALF_OPEN에서 회복되지 않았나

### 1. 호출이 없을 때는 COUNT_BASED 윈도우가 비워지지 않는다

이 외부 API는 캐시 적중률이 높았습니다. 평소에는 외부 호출이 거의 없고, 캐시가 만료되어 새 값이 필요한 순간에만 호출이 몰렸습니다.

`COUNT_BASED` 윈도우는 시간이 지났다고 오래된 결과가 사라지지 않습니다. 새 호출이 들어와야 오래된 결과가 밀려납니다. 그래서 한 번 들어간 실패가 한참 동안 남아 있다가, 다음 cache miss 구간에 다시 영향을 줄 수 있습니다.

`TIME_BASED` 윈도우였다면 최근 N초 기준으로 집계하므로 오래된 실패가 시간에 따라 자연스럽게 빠집니다. 물론 이것만으로 외부 API 호출 수가 줄어드는 것은 아니지만, bursty traffic에서는 회복 판정이 덜 왜곡됩니다.

### 2. HALF_OPEN에서 필요한 호출 수가 채워지지 않았다

Resilience4j의 HALF_OPEN은 제한된 수의 호출만 통과시킨 뒤 결과를 보고 CLOSED 또는 OPEN으로 이동합니다. 이 서비스에서는 `permittedNumberOfCallsInHalfOpenState`가 5였으므로, 5건의 결과가 필요했습니다.

문제는 cache stampede 이후의 트래픽 모양이었습니다.

1. 캐시 만료 직후 같은 키의 요청이 한꺼번에 외부 API로 몰린다.
2. 일부 요청이 timeout이 되면서 서킷이 OPEN으로 전환된다.
3. 시간이 지나 HALF_OPEN으로 전환될 수 있는 상태가 된다.
4. 하지만 그 사이 다른 요청 경로에서 캐시가 채워졌거나 fallback 응답이 나가고 있어, 외부 API로 향하는 새 호출이 거의 없다.
5. HALF_OPEN 판정에 필요한 결과 수가 채워지지 않아 상태가 오래 머문다.

여기서 `maxWaitDurationInHalfOpenState`의 기본값은 `0`입니다. Resilience4j에서 이 값의 0은 무제한 대기를 뜻합니다. 값을 명시하면 HALF_OPEN에 너무 오래 머물렀을 때 다시 OPEN으로 전환시킬 수 있습니다.

즉 `maxWaitDurationInHalfOpenState`는 "트래픽이 없어도 CLOSED로 정상화해주는 설정"이 아닙니다. HALF_OPEN에서 판정에 필요한 호출이 끝나지 않을 때 **무한 대기를 피하고 OPEN으로 되돌리는 안전장치**에 가깝습니다.

### 3. 근본 원인은 같은 시점에 발생한 cache miss였다

캐시 키는 `id:202605071230`처럼 시각이 포함된 형태였고, 일정 주기마다 여러 인스턴스에서 같은 시점에 새 키를 읽었습니다. 결과적으로 모든 인스턴스가 거의 동시에 cache miss를 맞았습니다.

외부 API가 평소에는 충분히 빠르더라도, 이 순간에는 상황이 달라집니다. 0.1초만 느려져도 timeout이 늘고, timeout은 서킷 브레이커의 실패율을 끌어올립니다. 서킷 브레이커는 외부 API를 보호하기 위해 열리지만, cache stampede 자체를 없애지는 못합니다.

## 해결 전략을 볼 때의 기준

cache stampede를 다룰 때는 "어떤 설정이 좋다"보다 **무엇을 줄이고 싶은가**를 먼저 봐야 합니다.

| 전략 | 주로 줄이는 것 | 잘 맞는 상황 | 주의할 점 |
|---|---|---|---|
| `TIME_BASED` 윈도우 | 오래된 실패의 영향 | 호출이 듬성듬성 들어오는 외부 API | 외부 API 호출 수는 줄이지 못함 |
| `maxWaitDurationInHalfOpenState` | HALF_OPEN 무한 대기 | HALF_OPEN에 오래 머무는 서킷 | CLOSED 복구가 아니라 OPEN 복귀 안전장치 |
| TTL jitter | 만료 시점의 동시성 | 만료 시각이 정확하지 않아도 되는 캐시 | 정각 데이터에는 적용하기 어려움 |
| refresh-ahead | 사용자 요청 경로의 cache miss | hot key, 읽기 위주 데이터, stale 허용 가능 데이터 | 불필요한 refresh와 오래된 값 정책 필요 |
| stale-while-revalidate | 원본 장애가 사용자 요청에 전파되는 정도 | 잠깐 오래된 값을 보여줘도 되는 데이터 | stale 허용 범위를 명확히 정해야 함 |
| single-flight | 같은 키의 동시 외부 호출 수 | cache miss가 몰리는 모든 구조 | 한 호출이 느리면 같은 키의 요청이 함께 기다림 |
| 분산 락 | 여러 인스턴스 간 중복 외부 호출 | 외부 API 보호가 중요한 분산 환경 | 락 TTL, 해제 원자성, 장애 시 fallback 필요 |

이 전략들은 서로 대체재이기도 하지만, 실제로는 조합해서 쓰는 경우가 많습니다. 예를 들어 hot key에는 refresh-ahead를 적용하고, 그래도 miss가 발생하는 순간은 single-flight로 묶고, 서킷 브레이커는 `TIME_BASED`와 `maxWaitDurationInHalfOpenState`로 오래된 실패와 HALF_OPEN 무한 대기를 줄이는 식입니다.

## 전략 1: 서킷 브레이커 설정 조정

서킷 브레이커 설정은 cache stampede의 근본 해결책은 아니지만, 장애가 길게 이어지지 않도록 도와줍니다.

### `TIME_BASED` 윈도우

트래픽이 계속 고르게 들어오는 서비스라면 `COUNT_BASED`도 충분합니다. 하지만 캐시 뒤의 외부 API처럼 호출이 드문드문 있다가 특정 순간에 몰리는 구조라면 `TIME_BASED`가 더 자연스러울 수 있습니다.

`TIME_BASED`는 최근 N초 기준으로 실패율을 계산하므로, 오래된 실패가 호출 수 부족 때문에 계속 남아 있는 상황을 줄입니다. 다만 외부 API로 나가는 호출 수 자체는 그대로입니다.

### `maxWaitDurationInHalfOpenState`

이 값은 HALF_OPEN 상태에서 허용된 호출이 끝나기를 기다리는 최대 시간을 제한합니다. 기본값 0은 무제한 대기입니다.

값을 설정하면 HALF_OPEN에 계속 머무르는 상황을 피할 수 있습니다. 단, 시간이 지났을 때 CLOSED로 보내는 설정이 아니라 OPEN으로 되돌리는 설정이라는 점을 분명히 이해해야 합니다. 회복 판정을 위한 성공 호출이 충분히 모이지 않았다면, 안전하게 다시 차단하는 쪽에 가깝습니다.

## 전략 2: TTL jitter

TTL jitter는 캐시 만료 시간을 조금씩 다르게 주는 방법입니다. 예를 들어 모든 키를 정확히 1시간 뒤에 만료시키는 대신, 55분에서 65분 사이의 랜덤 TTL을 주는 식입니다.

```text
나쁜 예: 모든 인스턴스가 12:00:00에 만료
좋은 예: 인스턴스마다 11:55:13, 12:01:42, 12:04:08에 만료
```

장점은 단순합니다. 만료 시점이 흩어지면 같은 순간에 cache miss가 몰릴 확률이 줄어듭니다.

하지만 모든 데이터에 쓸 수 있는 것은 아닙니다. 비즈니스상 "매시 정각 데이터"를 정확히 써야 하거나, 키 자체가 시간 슬롯을 포함해 정각에 바뀌는 구조라면 TTL만 흔들어도 효과가 제한적일 수 있습니다. 이 경우에는 키 설계, refresh-ahead, single-flight를 함께 봐야 합니다.

## 전략 3: refresh-ahead와 stale-while-revalidate

`refresh-ahead`는 캐시가 완전히 만료된 뒤 사용자 요청이 새 값을 가져오게 두는 대신, **만료가 가까워졌을 때 미리 새 값을 채워두는 전략**입니다. 목표는 "사용자 요청이 cache miss를 직접 맞는 순간"을 줄이는 것입니다.

예를 들어 TTL이 1시간인 데이터라면 55분쯤 지난 시점에 백그라운드 작업이 외부 API를 호출해 값을 갱신합니다. 갱신이 성공하면 다음 요청부터 새 값을 보고, 실패하면 기존 값을 얼마나 더 허용할지 정책을 따릅니다.

Caffeine의 `refreshAfterWrite`도 비슷한 문제를 푸는 도구입니다. 다만 정확히는 "시간이 되면 자동으로 백그라운드 갱신"이 아니라, **지정한 시간이 지난 엔트리가 다음에 조회될 때 refresh 대상이 되고 refresh가 비동기로 수행**됩니다. refresh 중에는 기존 값이 있으면 기존 값을 반환합니다. 이 동작은 eviction과 다릅니다. eviction은 값을 제거하므로 다음 조회가 새 값 로드를 기다려야 하지만, refresh는 기존 값을 유지한 채 새 값을 준비합니다.

refresh-ahead가 잘 맞는 데이터는 다음과 같습니다.

- 조회가 많다.
- 값이 어느 정도 오래되어도 괜찮다.
- 외부 API latency를 사용자 요청 경로에서 떼어내고 싶다.
- 장애 시 잠깐 stale 값을 제공하는 것이 빈 응답이나 timeout보다 낫다.

반대로 다음 경우에는 조심해야 합니다.

- 요청이 거의 없는 키가 많아 미리 갱신하면 낭비가 크다.
- 현재 재고, 권한, 가격처럼 오래된 값의 비용이 크다.
- 여러 인스턴스가 동시에 refresh를 수행할 수 있다.
- refresh 실패 시 stale 값을 얼마나 오래 허용할지 정하지 않았다.

여러 인스턴스가 동시에 refresh를 수행하면 refresh-ahead 자체가 또 다른 stampede가 될 수 있습니다. 그래서 refresh-ahead도 jitter, 리더 선출, single-flight, 분산 락과 함께 설계해야 합니다.

## 전략 4: single-flight

single-flight는 같은 키에 대한 동시 요청을 하나의 외부 호출로 합치는 패턴입니다.

```text
cache miss 100개 → 외부 API 호출 1개 → 결과를 100개 요청이 함께 사용
```

이 전략은 cache miss 자체를 없애지는 않습니다. 대신 miss가 발생했을 때 외부 API로 빠져나가는 호출 수를 제한합니다. 이 글의 사례처럼 같은 키의 요청이 한꺼번에 몰리는 구조에서는 가장 직접적인 처방입니다.

### Spring `@Cacheable(sync = true)`의 한계

스프링의 `@Cacheable(sync = true)`는 같은 키에 대해 여러 스레드가 동시에 값을 로드하려고 할 때, 메서드 호출을 동기화하라는 힌트입니다. 공식 Javadoc에도 다음 제약이 명시되어 있습니다.

- `unless`를 함께 쓸 수 없다.
- 하나의 캐시만 지정할 수 있다.
- 다른 캐시 작업과 조합할 수 없다.
- 실제 동기화 의미는 cache provider 구현에 의존한다.

외부 API 결과를 캐시할 때는 보통 **실패 결과는 캐시하지 않는다**는 정책이 필요합니다. 예를 들어 외부 API 일시 장애로 `null`을 받았는데 그 값이 TTL 동안 캐시되면, 1초짜리 장애가 긴 사용자 영향으로 늘어납니다.

`@Cacheable` 자체는 `null` 결과를 자동으로 제외해주지 않습니다. 실제로 `null`을 저장할지, 예외로 막을지는 CacheManager 구현과 설정에 따라 달라집니다. 예를 들어 Spring Data Redis는 기본 설정에서 null 캐싱을 허용합니다. 물론 CacheManager 설정으로 null 캐싱을 막을 수는 있지만, `@Cacheable(sync = true)`와 `unless = "#result == null"`를 함께 쓸 수 없다는 제약은 그대로 남습니다. 그래서 이 사례에서는 `@Cacheable(sync = true)`가 적합하지 않았습니다.

### Caffeine `Cache.get(key, mappingFunction)`

Caffeine의 `Cache.get(key, mappingFunction)`은 이 요구에 잘 맞습니다.

- 같은 키에 대한 동시 호출이 들어와도 mapping function은 키당 최대 한 번만 실행된다.
- mapping function이 `null`을 반환하면 캐시에 저장하지 않는다.
- 메서드 전체가 원자적으로 수행되므로 일반적인 "없으면 로드해서 저장" 패턴을 안전하게 대체한다.

그래서 이 서비스에서는 스프링 캐시 추상화 대신 Caffeine API를 직접 호출하는 형태로 바꿨습니다.

```kotlin
fun <K : Any, V : Any> Cache<K, V>.getOrLoadNullable(
    key: K,
    loader: () -> V?,
): V? {
    val nullableFn: Function<K, V?> = Function { loader() }
    @Suppress("UNCHECKED_CAST")
    return get(key, nullableFn as Function<K, V>)
}
```

이 헬퍼는 Kotlin과 Caffeine의 nullability 해석 차이를 다루기 위한 우회입니다. Caffeine은 mapping function이 `null`을 반환하면 저장하지 않는 동작을 지원하지만, Kotlin/JSpecify 환경에서는 `Cache<K, V>`의 `V`를 non-null로 해석해 nullable 반환 loader를 그대로 넘기기 어렵습니다.

위 캐스팅이 가능한 이유는 다음과 같습니다.

- 자바 generics는 런타임에 type 정보가 지워진다.
- `Function<K, V?>`와 `Function<K, V>`는 런타임에는 같은 형태의 객체다.
- Caffeine은 mapping function 결과가 `null`이면 캐시에 넣지 않는다.

이런 코드는 범위를 좁게 두고, 왜 필요한지 주석이나 함수명으로 의도를 드러내는 편이 좋습니다.

## 전략 5: Redis로 옮길 때의 분산 single-flight

지금까지의 single-flight는 **같은 JVM 안**에서만 동작합니다. Caffeine `Cache.get(key, loader)`는 같은 인스턴스 안의 동시 요청을 잘 합쳐주지만, 인스턴스가 10개라면 같은 cache miss가 최대 10번 외부 API로 나갈 수 있습니다.

Redis를 공유 캐시로 두면 인스턴스 간 캐시 값은 공유할 수 있습니다. 하지만 Redis를 쓴다고 해서 자동으로 분산 single-flight가 생기는 것은 아닙니다.

### Spring Data Redis의 기본 캐시 동작

Spring Data Redis의 `RedisCacheManager`는 기본적으로 non-locking `RedisCacheWriter`를 사용합니다. 공식 문서도 기본 writer가 lock-free이며, locking writer를 선택할 수 있지만 이 lock은 cache entry 단위가 아니라 cache level에 적용된다고 설명합니다.

따라서 `@Cacheable(sync = true)`와 Redis 조합을 쓴다고 해서 "모든 인스턴스에서 같은 키의 외부 API 호출이 정확히 1번만 실행된다"고 기대하면 안 됩니다. Spring의 `sync=true`는 provider에 대한 힌트이고, Redis 분산 single-flight 보장은 별도로 설계해야 합니다.

### Redis 분산 락 패턴

가장 기본적인 형태는 Redis `SET NX PX`로 락을 잡는 방식입니다.

```kotlin
fun get(key: String): Value? {
    redis.get(key)?.let { return it }

    val lockKey = "lock:$key"
    val token = UUID.randomUUID().toString()
    val acquired = redis.setIfAbsent(lockKey, token, lockTtl)

    return if (acquired) {
        try {
            redis.get(key)?.let { return it } // 락을 기다리는 동안 채워졌을 수 있으니 재확인
            val value = origin.fetch(key)
            value?.let { redis.set(key, it, cacheTtl) }
            value
        } finally {
            redis.releaseIfOwner(lockKey, token) // Lua script로 원자 실행
        }
    } else {
        waitForCacheOrFallback(key)
    }
}
```

직접 구현할 때는 다음을 반드시 신경 써야 합니다.

- 락 TTL이 너무 짧으면 외부 API 응답 전에 락이 풀린다.
- 락 TTL이 너무 길면 락을 잡은 인스턴스가 죽었을 때 다른 요청이 오래 막힌다.
- 락 해제는 "내가 잡은 락일 때만 삭제"해야 하므로 토큰 비교와 삭제를 Lua script로 원자 실행해야 한다.
- 락을 못 잡은 요청은 무작정 기다리지 말고, 짧게 재조회하거나 stale 값/fallback을 반환하는 정책이 필요하다.

### 두 단계 캐시

실무에서는 다음 구조를 자주 씁니다.

```text
요청 → Caffeine(L1, 로컬) → Redis(L2, 공유) → 외부 API
```

이 구조의 장점은 역할이 분리된다는 점입니다.

- Caffeine은 같은 인스턴스 안의 동시 요청을 묶어 Redis 트래픽을 줄인다.
- Redis는 인스턴스 간 값을 공유해 외부 API 호출을 줄인다.
- 외부 API 보호가 더 중요하면 Redis miss 구간에 분산 락을 추가한다.

단점은 일관성입니다. Redis 값이 갱신되거나 무효화되었을 때 각 인스턴스의 Caffeine 캐시까지 함께 비워야 한다면 pub/sub 같은 무효화 전파가 필요합니다. 데이터 변경이 드물고 TTL로 자연 갱신되어도 괜찮은 데이터에는 잘 맞지만, 강한 실시간 일관성이 필요한 데이터에는 맞지 않을 수 있습니다.

### 라이브러리 사용

직접 분산 락을 구현할 수도 있지만, 락 토큰, lease time, 장애 복구, Redis 클러스터 상황을 모두 직접 다루는 것은 부담이 큽니다. 외부 API 보호가 정말 중요하다면 Redisson 같은 검증된 라이브러리를 검토할 만합니다.

다만 라이브러리를 쓰더라도 "락을 잡으면 모든 문제가 끝난다"는 뜻은 아닙니다. 락 획득 실패 시 사용자에게 무엇을 줄지, stale 값을 허용할지, 락 timeout을 어떻게 잡을지, 외부 API timeout과 락 TTL의 관계를 어떻게 둘지까지 함께 정해야 합니다.

## 어떤 상황에서 무엇을 쓸까

### 1. 단일 인스턴스 또는 인스턴스별 중복 호출이 허용되는 경우

Caffeine `Cache.get(key, loader)`부터 검토합니다. 구현이 단순하고 같은 JVM 안의 stampede를 효과적으로 막습니다.

### 2. 같은 시점에 만료되는 키가 문제인 경우

TTL jitter를 먼저 봅니다. 구현 비용이 낮고 효과가 큽니다. 다만 정각성이 중요한 데이터나 시간 슬롯 키에는 효과가 제한적입니다.

### 3. hot key이고 stale 값을 잠깐 허용할 수 있는 경우

refresh-ahead 또는 stale-while-revalidate를 검토합니다. 사용자 요청 경로에서 외부 API latency를 떼어낼 수 있습니다. 대신 refresh 실패 시 오래된 값을 얼마나 허용할지 명확히 정해야 합니다.

### 4. 외부 API가 비싸거나 장애 전파 비용이 큰 경우

single-flight를 반드시 넣는 편이 좋습니다. 다중 인스턴스라면 Redis 분산 락이나 검증된 라이브러리까지 검토합니다.

### 5. 서킷 브레이커가 회복되지 않는 경우

cache stampede를 줄이는 처방과 별도로, 서킷 브레이커 설정도 조정합니다.

- bursty traffic이면 `TIME_BASED` 윈도우를 검토한다.
- HALF_OPEN 무한 대기를 피하려면 `maxWaitDurationInHalfOpenState`를 명시한다.
- 자동 전환이 필요한 운영 모델이면 `automaticTransitionFromOpenToHalfOpenEnabled`를 검토한다.

다만 이 설정들은 외부 API 호출 수 자체를 줄이지 않습니다. 근본 원인이 cache stampede라면 single-flight, refresh-ahead, TTL jitter 같은 캐시 측 처방이 함께 필요합니다.

## 정리

- cache stampede는 단순한 cache miss 증가가 아니라 외부 API timeout, fallback 증가, 서킷 브레이커 OPEN, 회복 지연으로 이어질 수 있다.
- 서킷 브레이커는 외부 시스템을 보호하는 장치이지, cache stampede를 없애는 장치는 아니다.
- `TIME_BASED` 윈도우와 `maxWaitDurationInHalfOpenState`는 오래된 실패와 HALF_OPEN 무한 대기를 줄이는 데 도움이 되지만, 외부 호출 수를 줄이지는 못한다.
- TTL jitter는 만료 시점을 흩뿌리고, refresh-ahead는 사용자 요청이 miss를 직접 맞는 일을 줄인다.
- single-flight는 miss가 실제로 발생했을 때 같은 키의 동시 외부 호출을 하나로 합치는 가장 직접적인 처방이다.
- Caffeine `Cache.get(key, loader)`는 로컬 single-flight와 null 미저장을 함께 만족하므로 외부 API 캐시에 잘 맞는다.
- Redis를 쓰더라도 분산 single-flight가 자동으로 생기지는 않는다. 인스턴스 간 중복 호출까지 막아야 한다면 분산 락, 두 단계 캐시, 검증된 라이브러리를 함께 검토해야 한다.

---

## 참고 자료

- [Resilience4j CircuitBreaker 공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)
- [Spring `@Cacheable` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)
- [Caffeine `Cache` Javadoc](https://www.javadoc.io/doc/com.github.ben-manes.caffeine/caffeine/latest/com.github.benmanes.caffeine/com/github/benmanes/caffeine/cache/Cache.html)
- [Caffeine Refresh Wiki](https://github.com/ben-manes/caffeine/wiki/Refresh)
- [Spring Data Redis Cache 공식 문서](https://docs.spring.io/spring-data/redis/reference/redis/redis-cache.html)

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
