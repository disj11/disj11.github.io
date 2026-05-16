---
title: "캐시 만료 직후 외부 API 호출이 한꺼번에 몰려 서킷 브레이커가 정상화되지 않는 문제"
description: "캐시 만료 직후 같은 키의 요청이 외부 API로 몰리면서 Resilience4j 서킷 브레이커가 HALF_OPEN 상태에 갇힌 사례를 정리하고, Caffeine의 single-flight 패턴으로 해결한 과정을 다룹니다."
date: 2026-05-07T18:31:32+09:00
url: "/cache-stampede-and-circuit-breaker-half-open/"
tags: ["cache", "circuit-breaker", "resilience4j", "caffeine", "performance"]
---

외부 API를 호출하는 서비스에서 실제로 겪은 운영 문제를 정리한 글입니다. 서킷 브레이커, 캐시, 동시성이 함께 얽혀 있었고, 겉으로 보이는 증상만 봐서는 원인이 바로 드러나지 않았습니다. 필요한 배경부터 짚고, 문제가 어디서 시작됐는지 순서대로 풀어보겠습니다.

---

## 사전 지식

### 서킷 브레이커란

외부 API가 느려지거나 장애 상태가 되면, 응답을 기다리는 동안 우리 서버의 스레드와 메모리도 묶입니다. 호출이 계속 쌓이면 외부 API뿐 아니라 우리 서버까지 같이 흔들립니다.

**서킷 브레이커(Circuit Breaker)** 는 이런 상황을 막기 위한 디자인 패턴입니다. 이름처럼 전기 회로의 차단기에서 따왔고, 보통 세 가지 상태로 동작합니다.

- **CLOSED (정상)**: 외부 API 호출이 그대로 통과한다.
- **OPEN (차단)**: 실패가 많아져 차단기가 열린 상태. 외부 API를 호출하지 않고 즉시 실패를 반환한다. 외부 API에 더 부담을 주지 않고, 우리 서버 자원도 보호한다.
- **HALF_OPEN (확인 중)**: OPEN으로 일정 시간이 지나면 진입한다. 정해진 수의 요청만 통과시켜 보고, 결과가 좋으면 CLOSED로 돌아가고 나쁘면 다시 OPEN으로 떨어진다.

자바/코틀린 환경에서는 [Resilience4j](https://resilience4j.readme.io/) 라이브러리가 표준처럼 쓰입니다.

### 캐시와 TTL

자주 조회되지만 자주 바뀌지 않는 데이터를 매번 외부 API에서 받아오면 낭비가 큽니다. 한 번 받아온 결과를 메모리에 저장해 두고 다음 요청에서 재사용하는 것이 캐시입니다. 캐시 값이 영원히 유효할 수는 없으므로 일정 시간이 지나면 사라지게 두는데, 이 시간을 **TTL(Time To Live)** 이라고 부릅니다.

이 글의 서비스는 자바 진영에서 가장 많이 쓰이는 로컬 캐시 라이브러리 [Caffeine](https://github.com/ben-manes/caffeine)을 사용합니다.

---

## 문제 발생

문제가 된 서비스는 외부 API를 캐시로 감싸고, 그 앞에 서킷 브레이커를 둔 구조였습니다. 어느 시점부터 이런 현상이 보였습니다.

> 일부 인스턴스의 서킷이 한 번 OPEN으로 떨어진 뒤, **HALF_OPEN 상태에 영구히 갇혀 정상으로 돌아오지 않는다.** 외부 API는 이미 회복되었는데도 그 인스턴스만 호출을 차단한 상태로 남아 있다.

서킷 설정은 다음과 같았습니다.

```yaml
slidingWindowType: COUNT_BASED
slidingWindowSize: 100
minimumNumberOfCalls: 20
waitDurationInOpenState: 10s
permittedNumberOfCallsInHalfOpenState: 5
```

각 항목의 의미는 다음과 같습니다.

- `slidingWindowType: COUNT_BASED` — 최근 호출의 결과(성공/실패)를 **개수 기준**으로 쌓아둔다. (대안: `TIME_BASED`는 시간 단위로 쌓아둔다.)
- `slidingWindowSize: 100` — 최근 100건의 결과를 보관한다.
- `minimumNumberOfCalls: 20` — 최소 20건이 쌓여야 실패율 판정을 시작한다. 그 미만이면 판정 보류.
- `waitDurationInOpenState: 10s` — OPEN으로 떨어진 뒤 10초가 지나야 HALF_OPEN으로 전환한다.
- `permittedNumberOfCallsInHalfOpenState: 5` — HALF_OPEN 상태에서 5건의 결과를 모아 그걸로 다음 상태(CLOSED 또는 OPEN)를 판정한다.

## 원인 파악

### 1. 트래픽 패턴과 윈도우 종류가 안 맞는다

이 외부 API는 캐시 적중률이 높았습니다. 평소에는 캐시가 대부분 막아주니 외부 호출이 거의 없고, **캐시가 만료되어 새 값이 필요한 순간에만 호출이 한꺼번에 몰리는 패턴**이었습니다. 흔히 "bursty traffic"이라고 부르는 형태입니다.

이런 트래픽에는 `COUNT_BASED` 윈도우가 잘 맞지 않습니다. 개수 기준 윈도우는 시간이 지난다고 오래된 결과가 사라지지 않고, 새 호출이 들어와야만 밀려납니다. 한 번 들어간 실패가 그대로 남아 있다가, 다음 트래픽이 몰리는 시점에 다시 더해져 OPEN으로 떨어지기 쉽습니다.

`TIME_BASED`였다면 60초 같은 시간 단위로 윈도우가 자연스럽게 비워지므로 이 문제가 덜합니다.

### 2. HALF_OPEN에서 5건이 채워지지 않는다

Resilience4j의 HALF_OPEN은 **`permittedNumberOfCallsInHalfOpenState`만큼 결과가 모일 때까지 그 상태에 머무릅니다.** `maxWaitDurationInHalfOpenState`(일정 시간이 지나면 강제로 다음 상태로 전환)의 기본값이 `0`인데, 여기서 0은 "무제한 대기"를 뜻합니다. 우리 설정에는 이 값을 따로 지정하지 않았습니다.

시간 순서로 보면 이렇습니다.

1. 캐시 만료 직후 외부 호출이 한꺼번에 몰림 → 일부 timeout 발생 → 실패율이 임계치를 넘어 서킷 OPEN
2. 10초 뒤 HALF_OPEN 진입. 이때부터는 5건만 통과시켜 그 결과로 판정한다.
3. 그러나 그 사이 다른 요청 경로에서 캐시가 이미 채워졌거나 이전 요청들이 fallback 응답을 받고 있어, **새로운 외부 호출 자체가 발생하지 않는다.**
4. 5건이 안 채워지므로 영원히 HALF_OPEN.

다음 캐시 만료가 와야 비로소 5건이 채워지는데, 그때는 또 한꺼번에 몰리는 시점이라 5건 중 일부가 또 timeout이 나면 OPEN으로 되돌아갑니다. 정상화될 기회가 운에 맡겨지는 셈입니다.

### 3. 근본 원인: cache stampede

위 두 현상의 공통 원인은 **캐시 만료 직후 같은 키의 요청들이 한꺼번에 외부 API로 빠져나가는 현상**입니다. 보통 "cache stampede" 또는 "thundering herd"라고 부릅니다. 평소에는 잠잠하다가 한순간에 몰리는 모양새입니다.

캐시 키가 `id:202605071230` 처럼 시각이 들어간 형태로 매시간 회전하고 있었기 때문에, **모든 인스턴스의 캐시가 정확히 같은 시점에 일제히 무효화**되는 구조였습니다. 외부 API가 0.1초만 느려져도 다수의 호출이 timeout으로 떨어지고, 서킷이 OPEN으로 가버립니다.

## 해결

### 검토한 방향

| 방향 | 효과 | 비고 |
|---|---|---|
| 윈도우를 `TIME_BASED`로 전환 | 오래된 실패가 시간으로 자연 소멸 | 증상 완화 |
| `maxWaitDurationInHalfOpenState` 명시 | 트래픽이 없어도 강제로 다음 상태로 전환 | 증상 완화 |
| TTL을 무작위로 흩뿌리기(jitter) | 만료 시점이 분산되어 stampede 완화 | 만료 시각이 비즈니스상 정확해야 하면 적용 불가 |
| **같은 시점에 들어온 요청을 1번의 외부 호출로 합치기 (single-flight)** | stampede 자체를 차단 | **근본 처방** |

서킷 설정을 손보면 증상은 누그러질 수 있습니다. 하지만 **외부 API에 도달하는 호출 수 자체를 줄이지 않으면** 같은 일이 다시 벌어집니다. 결국 외부 호출 수를 줄여야 합니다.

### 처방: single-flight

"single-flight"는 같은 목적지로 가는 요청을 한 번에 태운다는 뜻에 가까운 이름입니다. 같은 키에 대한 동시 cache miss 요청이 100개 들어와도 **외부 API로는 1번만 호출하고**, 나머지 요청은 그 결과를 함께 쓰게 만드는 방식입니다. Go 언어에는 표준 라이브러리에 `singleflight` 패키지가 있을 정도로 흔히 쓰이는 패턴입니다.

스프링 환경에서는 보통 두 가지 선택지를 검토하게 됩니다.

#### 선택지 A: `@Cacheable(sync = true)`

스프링의 `@Cacheable` 어노테이션에는 `sync = true` 옵션이 있습니다. 이 옵션을 켜면 같은 키에 대한 동시 호출이 들어왔을 때 내부적으로 락을 잡아 메서드를 1번만 실행합니다. 겉으로는 가장 간단한 선택지입니다.

그러나 결정적인 제약이 있습니다.

- **`sync = true`는 `unless` 옵션과 함께 쓸 수 없다** (KDoc에 명시).
- 그런데 `@Cacheable`은 **기본적으로 `null` 반환도 캐시한다**. (내부적으로 `NullValue`라는 sentinel 객체로 감싸 저장한다. 즉 실제 `null`을 저장하는 것은 아니지만 사용자 입장에서는 `null`이 캐시된 것처럼 동작한다.)
- 따라서 `unless = "#result == null"`이 없으면 외부 API 일시 장애로 받은 `null` 결과가 다음 캐시 만료까지 그대로 박제됩니다. 1초의 장애가 한 시간짜리 사용자 영향으로 증폭됩니다.

외부 호출을 감싸는 캐시처럼 **"실패는 캐시하지 않는다"** 가 필요한 곳에서는 부적합합니다.

> 참고로 `unless`는 메서드 결과를 본 뒤 "이런 결과면 캐시하지 마"라고 지정하는 옵션이고, `#result`는 그 결과 값을 가리키는 SpEL 표현식입니다.

#### 선택지 B: Caffeine `Cache.get(key, loader)`

Caffeine의 `Cache.get(key, mappingFunction)` 메서드를 직접 사용하면 다음 두 가지가 함께 보장됩니다.

- 같은 키에 대한 동시 호출이 들어와도 **mappingFunction은 단 1번만 실행** (single-flight)
- mappingFunction이 `null`을 반환하면 **그 결과는 캐시에 저장하지 않음** (Caffeine 공식 문서: *"enters it into this cache unless null"*)

두 요구가 추가 옵션 없이 한 번에 충족됩니다. 그래서 이 메서드를 직접 호출하는 형태로 캐시 서비스를 다시 구성하기로 했습니다.

### 구현 시 마주친 Kotlin과 Caffeine 사이의 작은 간극

여기서부터는 위 결정을 코틀린에서 그대로 옮길 때 생긴 두 가지 작은 문제에 관한 이야기입니다.

#### 문제 1: `null`을 반환하는 mappingFunction을 코틀린이 거부한다

Caffeine의 캐시는 `Cache<K, V>`로 정의되어 있고, V를 non-null로 강제합니다. 그런데 우리는 외부 호출이 실패했을 때 mappingFunction이 `null`을 반환하길 원합니다(그래야 캐시에 저장 안 함). 코틀린의 null safety가 이 조합을 막아섭니다.

런타임 동작은 우리 의도대로 작동하는데(자바 generics는 컴파일 후 사라지므로 런타임에는 type 정보가 없음), 컴파일 단계에서 코틀린이 막는 것이 문제입니다. 그래서 한 단계 우회하는 작은 헬퍼를 만들었습니다.

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

여기서 중요한 부분은 `nullableFn as Function<K, V>` 라인입니다. `Function<K, V?>`(nullable 반환)를 `Function<K, V>`(non-null 반환)로 강제 변환합니다. 위험해 보이지만 이 경우에는 문제가 없습니다. 이유는 다음과 같습니다.

- 자바 generics는 컴파일이 끝나면 type 정보가 모두 사라진다(이것을 type erasure라고 부른다). 즉 `Function<K, V?>`와 `Function<K, V>`는 실행 시점에는 똑같은 객체다.
- 변환은 컴파일러를 통과시키기 위한 표시일 뿐 실제 검사를 일으키지 않는다.
- Caffeine이 mappingFunction을 호출하고 결과가 `null`이면 자체적으로 "저장하지 않음" 분기를 탄다.

#### 문제 2: IntelliJ가 "이 함수는 항상 non-null만 반환한다"고 경고한다

위 헬퍼 함수에 IntelliJ가 노란 줄을 긋습니다. "선언상 nullable인데 실제로는 항상 non-null만 반환하니 굳이 nullable로 선언할 필요 없다"는 취지의 경고입니다. 분명 우리는 `null`이 반환될 수 있다는 걸 알고 있는데 왜 이런 경고가 뜰까요?

원인을 추적해 보면 라이브러리 측 어노테이션에 누락이 있습니다. Caffeine 3.2.x는 [JSpecify](https://jspecify.dev/) 라는 nullness 어노테이션 표준을 사용하는데, 클래스 단에 `@NullMarked`가 적용되어 있어 **명시적으로 `@Nullable`을 붙이지 않은 모든 type은 non-null로 간주됩니다.** 그런데 `Cache.get` 두 종류 중 한 쪽에만 `@Nullable`이 붙어 있습니다.

```
public abstract V get(K);                  // @Nullable 있음
public abstract V get(K, Function<...>);   // @Nullable 없음
```

문서(KDoc)에는 두 메서드 모두 *"or null if the computed value is null"* 이라고 명확히 쓰여 있습니다. **즉 라이브러리 코드의 어노테이션과 문서가 어긋난 상태**이고, 코틀린 컴파일러는 어노테이션을 따라가므로 nullable이 아닌 것으로 해석합니다.

이런 경우 가장 정직한 해법은 "우리는 라이브러리 동작과 어노테이션이 어긋난 사실을 알고 있고, 문서를 신뢰한다"고 코드에 명시하는 것입니다.

```kotlin
@Suppress("RedundantNullableReturnType")
fun <K : Any, V : Any> Cache<K, V>.getOrLoadNullable(...) { ... }
```

향후 라이브러리가 `@Nullable`을 보완하면 경고가 자연스럽게 사라지므로, 이 `@Suppress`는 기술 부채로 남지 않습니다.

> `@Suppress`는 컴파일러나 IDE가 띄우는 특정 경고를 그 범위 안에서만 끄는 코틀린 기본 어노테이션입니다.

## 정리

- 외부 API + 캐시 + 서킷 브레이커 조합에서, 서킷이 HALF_OPEN에 갇히는 현상의 본질은 대개 **캐시 만료 직후의 호출 폭주(cache stampede)** 다.
- 서킷 설정 조정은 증상 완화에 도움이 되지만, **외부 호출 수 자체를 줄이는 처방이 함께 가야** 근본이 해결된다.
- **single-flight** — 같은 키의 동시 호출을 1번의 외부 호출로 합치는 패턴 — 이 가장 직접적인 처방이다.
- 스프링의 `@Cacheable(sync = true)`는 `unless`와 함께 쓸 수 없어 외부 호출 캐시에 부적합하다. **Caffeine `Cache.get(key, loader)`가 "single-flight + null 미저장"을 한 번에 만족하는 자연스러운 선택**이다.
- 라이브러리의 nullness 어노테이션과 문서가 어긋나 IDE가 잘못된 경고를 띄우는 경우, 좁은 범위의 `@Suppress`로 의도를 명시하는 것이 가장 정직한 해법이다.

---

## 부록: 로컬 캐시(Caffeine)에서 Redis로 옮기면 single-flight는 어떻게 달라지는가

지금까지 다룬 내용은 모두 **한 서버 안의 메모리 캐시(Caffeine)** 기준입니다. 향후 캐시를 인스턴스 간에 공유해야 한다면 Redis로 옮기게 되는데, 이 순간 single-flight 문제는 격이 달라집니다.

### 무엇이 달라지는가

Caffeine 시절에는 `Cache.get(key, loader)` 한 줄이 자동으로 처리해주던 일을, Redis로 옮기면 다음과 같은 명시적인 단계로 풀어 써야 합니다.

1. Redis에 `GET`을 보내 캐시 조회
2. 값이 없으면 **분산 락**을 잡으려 시도 (다른 서버 인스턴스의 동시 접근을 막기 위해)
3. 락을 잡은 뒤 다시 `GET` (그 사이 다른 누가 채워뒀을 수 있으므로)
4. 그래도 없으면 외부 API 호출
5. 결과를 `SET`으로 캐시에 저장
6. 락 해제

각 단계가 모두 네트워크를 타며, 단계마다 실패 가능성이 있습니다.

> **분산 락(distributed lock)** 이란 여러 서버 인스턴스가 공유 자원에 동시에 접근하지 못하게, 외부 저장소(Redis 등)에 "내가 먼저 잡았다"는 표시를 남기는 방식의 락입니다. 한 인스턴스 안의 자바 `synchronized`나 `ReentrantLock`으로는 다른 인스턴스를 막을 수 없으므로, 인스턴스 간 동기화에 필수입니다.

여기서 하나 더 짚을 점이 있습니다. **스프링의 `@Cacheable(sync = true)`도 이 문제를 풀어주지 못합니다.** Spring Data Redis의 `RedisCache.get(key, valueLoader)`는 자바의 `synchronized` 블록을 사용하는데, 이는 **같은 JVM 안에서만** 동작합니다. 인스턴스가 N개면 외부 API는 여전히 N번 호출됩니다.

### 흔히 쓰이는 세 가지 패턴

#### 1. 분산 락을 직접 만들기 (Redis `SET NX PX` 사용)

가장 기본적인 형태. 의사 코드 수준으로 보면 다음과 같습니다.

```kotlin
fun get(key: String): Value? {
    redis.get(key)?.let { return it }

    val lockKey = "lock:$key"
    val token = UUID.randomUUID().toString()
    // SET key value NX PX <ttl>: key가 없을 때만 set하고 ttl 후 자동 삭제
    val acquired = redis.setIfAbsent(lockKey, token, lockTtl)

    return if (acquired) {
        try {
            redis.get(key)?.let { return it } // 다시 한 번 확인
            val value = origin.fetch(key)
            value?.let { redis.set(key, it, cacheTtl) }
            value
        } finally {
            redis.releaseIfOwner(lockKey, token) // Lua script로 원자 실행
        }
    } else {
        // 락 못 잡은 쪽은: 잠깐 기다렸다 다시 캐시 확인 / 임시 응답 / 빈 응답
        waitForCacheOrFallback(key)
    }
}
```

직접 만들 때 자주 빠뜨리는 디테일들:

- **락 TTL 설정.** 너무 짧으면 외부 API가 응답하기 전에 락이 만료되어 다른 인스턴스가 그 사이 끼어들 수 있다. 너무 길면 락 잡은 인스턴스가 죽었을 때 그만큼 다른 요청들이 막힌다.
- **락 해제는 반드시 Lua script로 원자 실행.** 락을 잡을 때 발급한 토큰과 현재 락 값이 같을 때만 삭제해야 한다. 그러지 않으면 TTL 만료 후 다른 인스턴스가 다시 잡은 락을 잘못 삭제할 위험이 있다.
- **락을 못 잡은 인스턴스의 동작 방식.** 무작정 기다리면 락 잡은 쪽이 죽었을 때 줄줄이 막힌다. 짧은 간격으로 캐시를 다시 조회해 보거나, 임시 응답을 주는 등의 전략이 필요하다.

> **Lua script**는 Redis가 지원하는 짧은 프로그램입니다. 여러 명령을 묶어서 "다른 명령이 끼어들지 못하게" 한 덩어리로 실행할 수 있어, 락 해제처럼 "확인 후 삭제"를 안전하게 하고 싶을 때 씁니다.

#### 2. 두 단계 캐시 (Caffeine + Redis)

실무에서 가장 자주 쓰이는 형태이고, 보통 이걸로 충분합니다. 그림으로 표현하면 이렇습니다.

```
요청 → Caffeine (1단계: 로컬) → Redis (2단계: 공유) → 외부 API
```

각 단계가 자기 책임의 single-flight를 담당합니다.

- **1단계 Caffeine**: 같은 인스턴스 안에서 동시에 들어온 호출을 1번으로 합쳐 Redis 트래픽을 줄인다.
- **2단계 Redis**: 인스턴스 간 동시 호출은 분산 락으로 합쳐 외부 API 트래픽을 줄인다.

이렇게 두면 외부 API가 받는 부하는 본질적으로 1로 수렴하고, 분산 락의 비용은 "1단계가 못 막은 첫 호출"에만 발생합니다. 같은 인스턴스 안의 다른 요청들은 1단계에서 끝나니까요.

단점은 **두 캐시의 일관성 문제**입니다. 어떤 데이터가 무효화될 때 1단계 캐시까지 함께 비워야 한다면, Redis pub/sub 같은 추가 메커니즘이 필요합니다. 데이터 변경이 드물고 TTL로 자연 갱신되는 경우에는 잘 맞고, 실시간 일관성이 중요한 데이터에는 부적합합니다.

> **Redis pub/sub**: Redis가 제공하는 발행/구독 메시지 기능. 한 인스턴스가 "이 키 무효화"라는 메시지를 보내면 그걸 구독한 모든 인스턴스가 받아서 자기 로컬 캐시에서 지웁니다.

#### 3. 검증된 라이브러리(Redisson 등) 사용

직접 구현하는 대신 [Redisson](https://github.com/redisson/redisson) 같은 분산 데이터 구조 라이브러리를 도입하는 방법.

- `RLock` — 분산 락. 락을 잡은 인스턴스가 작업 중인 동안 락 만료를 자동으로 갱신해주는 watchdog 기능이 있다.
- `RLocalCachedMap` — 위 "두 단계 캐시 + 무효화 자동 동기화"가 통째로 패키지화된 형태.
- `RMapCache.computeIfAbsent` — 분산 환경에서의 compute-if-absent를 Lua script로 안전하게 구현해 마치 단일 메서드처럼 호출 가능.

장점은 **edge case들이 라이브러리 안에 이미 처리되어 있다는 점**입니다. 락 토큰의 안전 해제, watchdog 자동 갱신, Redis 클러스터 fail-over 시의 동작 등은 손으로 만들면 빠뜨리기 쉽지만 라이브러리에서는 검증되어 있습니다.

> **watchdog**: 락 잡은 작업이 락 TTL보다 오래 걸릴 것 같으면 백그라운드에서 락의 만료 시각을 자동으로 연장해주는 장치. "주기적으로 아직 살아있다고 알리는" 동작이라 watchdog(감시견)이라는 이름이 붙었습니다.

### 그럼 Lettuce는?

스프링 부트에서 Spring Data Redis를 쓰면 기본 Redis 클라이언트로 [Lettuce](https://github.com/lettuce-io/lettuce-core)가 들어옵니다. 그래서 "Lettuce도 Redisson 기능을 다 지원하지 않나?"라는 의문이 자연스럽게 생깁니다.

답은 "아니오"입니다. **Lettuce는 Redis 명령어를 잘 던져주는 클라이언트(드라이버)고, Redisson은 그 명령어들로 만든 분산 자료구조 프레임워크**입니다. 추상화 수준이 다릅니다.

| 항목 | Lettuce | Redisson |
|---|---|---|
| 본질 | Redis 클라이언트 | 분산 데이터 구조 프레임워크 |
| 제공 단위 | Redis 명령어(`SET`, `GET`, `EVAL` 등) | Java 추상화(`RLock`, `RMap`, `RBucket` 등) |
| 분산 락 | **직접 구현 필요** | `RLock.lock()` 한 줄 |
| 분산 캐시 (entry별 TTL) | 직접 구현 | `RMapCache` |
| L1+L2 캐시 + 무효화 동기화 | 직접 조립 | `RLocalCachedMap` |
| 비동기/리액티브 지원 | 풍부함 | 지원 |
| 스프링 부트 기본 채택 | **○** | 별도 추가 |

Lettuce를 그대로 쓰면서 분산 single-flight를 구현하려면 다음 중 하나를 선택하게 됩니다.

- **`RedisTemplate` + 직접 작성한 Lua script로 락 구현.** 가장 원시적. 토큰 안전 해제는 반드시 Lua로.
- **Spring Integration의 `RedisLockRegistry` 사용.** 스프링 생태계에서 가장 무난한 선택. Lettuce 위에서 동작하고 락 해제 안전성이 이미 처리되어 있다. 다만 watchdog 자동 갱신 같은 기능은 없으므로 락 TTL을 충분히 길게 잡아야 한다.
- **Lettuce는 그대로 두고 Redisson만 추가로 도입해 락에만 사용.** 두 클라이언트가 같은 Redis를 공유해도 충돌하지 않는다. 의존성이 늘어나는 부담만 감수하면 가장 매끄럽다.

### Caffeine 시절과의 차이 한눈에

| 항목 | Caffeine (로컬) | Redis (분산) |
|---|---|---|
| single-flight 범위 | 같은 JVM 안 | 모든 인스턴스 사이 |
| 구현 단위 | `Cache.get(key, loader)` 한 줄 | 락 획득/해제 + 재조회 + 실패 시 처리 |
| 실패 시나리오 | 거의 없음 (같은 프로세스 안) | 락 잡은 인스턴스 장애, 락 만료 중 외부 API 응답, 네트워크 단절 |
| 조정해야 할 TTL 개수 | 캐시 TTL 1개 | 캐시 TTL + 락 TTL 2개 |
| 외부 API가 받는 부하 | (인스턴스 수)배 | 1에 수렴 (락이 동작하는 한) |
| `@Cacheable(sync=true)`의 효과 | single-flight 동작함 | **같은 JVM 안에서만** 동작 (분산 single-flight 아님) |

### 실무 권장 순서

1. **Caffeine만으로 충분한지 먼저 측정한다.** 인스턴스 수가 많지 않거나 외부 API가 그 정도의 동시 호출은 견딘다면, 굳이 Redis로 가지 않아도 된다.
2. **넘어가야 한다면 두 단계 캐시(Caffeine + Redis)부터.** Redis 단독보다 외부 API 보호와 응답 속도 양쪽에서 유리하다.
3. **분산 락은 외부 API 보호가 정말 critical할 때만 추가한다.** 두 단계 캐시에 락 없이 Redis 캐시만 둬도 폭주가 어느 정도는 흡수된다. 그 정도로 충분한 경우가 많다.
4. **락이 필요하다면 Redisson 같은 검증된 라이브러리를 우선 검토한다.** 직접 구현은 토큰 관리, lease 갱신, 해제 원자성 같은 부분에서 사고가 잘 난다.

요약하면, Caffeine에서는 라이브러리 한 줄에 숨어 있던 동기화가 Redis로 옮겨가는 순간 분산 시스템 설계 문제로 드러납니다. 그래서 "Redis로 바꾸자"는 결정 자체보다 **"우리에게 어느 정도의 single-flight 보장이 필요한가"** 를 먼저 정해야 합니다.

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
