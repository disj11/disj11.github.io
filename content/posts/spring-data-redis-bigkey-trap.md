---
title: "Spring Data Redis @RedisHash가 만들 수 있는 BigKey 문제"
description: "Spring Data Redis의 @RedisHash와 CrudRepository를 사용할 때 내부 인덱스용 Set 때문에 BigKey가 생길 수 있습니다. 원인과 위험성을 살펴보고 RedisTemplate으로 저장 방식을 직접 제어하는 방법을 정리합니다."
date: 2025-12-05T23:40:26+09:00
url: "/spring-data-redis-bigkey-trap/"
tags: ["redis", "spring", "spring-data-redis", "performance"]
---

## 들어가며: 편리한 추상화의 비용

Spring Data Redis는 `@RedisHash` 어노테이션과 `CrudRepository` 인터페이스로 Redis 데이터를 객체 중심으로 다룰 수 있게 해줍니다. 편리하지만 내부 동작을 모른 채 쓰면, 데이터가 쌓이면서 예상하지 못한 'BigKey'가 만들어질 수 있습니다.

이 글에서는 Spring Data Redis Repository 기능에서 BigKey가 생기는 이유와, RedisTemplate으로 저장 방식을 직접 제어하는 방법을 정리합니다.

### `@RedisHash`와 `CrudRepository`의 동작 방식과 문제점

다음과 같이 `@RedisHash("orders")` 어노테이션이 적용된 `Order` 엔티티가 있다고 가정해 보겠습니다.

```kotlin
@RedisHash("orders")
data class Order(
    @Id
    val id: String,
    val status: String,
    val createdAt: LocalDateTime
)
```

이 엔티티를 다루기 위해 `CrudRepository`를 상속받는 리포지토리를 정의합니다.

```kotlin
interface OrderRepository : CrudRepository<Order, String> {
}
```

서비스 로직에서 `orderRepository.save(order)`를 호출하면 `Order` 객체는 Redis에 저장됩니다. 하지만 이때 Spring Data Redis는 단순히 객체의 데이터만 저장하는 것이 아니라, 추가적인 인덱스 데이터를 함께 관리합니다. `findAll()`, `count()`와 같은 Repository 메서드를 지원하기 위해, **저장되는 모든 엔티티의 ID를 별도의 `SET` 자료구조에 자동으로 추가**하는 것입니다.

실제 Redis에는 다음과 같은 두 종류의 데이터가 생성됩니다.

1.  **`orders:<ID>` (HASH 타입)**: `Order` 객체의 실제 데이터가 저장됩니다. (예: `orders:ORDER123`)
2.  **`orders` (SET 타입)**: `save()`가 호출될 때마다 해당 엔티티의 ID(`ORDER123`, `ORDER456` 등)가 이 `SET`에 계속해서 누적됩니다.

```
# 1. 실제 데이터 (HASH)
> HGETALL orders:ORDER123
1) "status"
2) "CREATED"
3) "createdAt"
4) "2025-11-11T10:00:00"

# 2. 모든 엔티티 ID를 저장하는 인덱스용 Set (BigKey 잠재성)
> SMEMBERS orders
1) "ORDER123"
2) "ORDER456"
... (수백만 개의 ID가 누적될 수 있음) ...
```

주문 데이터가 수백만, 수천만 건으로 늘어나면 `orders`라는 단일 키가 많은 멤버를 가진 BigKey가 됩니다.

### BigKey의 위험성

Redis는 싱글 스레드 기반으로 동작하기 때문에, 큰 키 하나가 다음과 같은 문제를 만들 수 있습니다.

*   **메모리 사용량 급증**: 하나의 키에 데이터가 집중되어 특정 Redis 노드나 클러스터 샤드의 메모리를 과도하게 점유합니다.
*   **성능 저하**: BigKey에 대한 연산(조회, 추가 등)은 다른 모든 요청을 지연시켜 Redis 전체의 응답성을 떨어뜨립니다.
*   **운영 리스크**: BigKey는 삭제하는 데에도 수 초에서 수 분이 걸릴 수 있습니다. 이 시간 동안 Redis가 다른 요청을 처리하지 못하면 장애로 이어질 수 있습니다.

### 해결 방안: `RedisTemplate`을 통한 직접 제어

가장 직접적인 해결책은 `CrudRepository`의 자동 ID 관리 기능에 의존하지 않고, `RedisTemplate`으로 데이터를 직접 저장하고 관리하는 것입니다. 이렇게 하면 BigKey를 유발하는 인덱스용 `SET` 키가 만들어지지 않습니다.

#### **AS-IS: `CrudRepository` 사용 코드**

```kotlin
// OrderRepository.kt
interface OrderRepository : CrudRepository<Order, String>

// OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository
) {
    fun saveOrder(order: Order) {
        orderRepository.save(order) // 이 호출이 BigKey를 생성
    }
}
```

#### **TO-BE: `RedisTemplate` 사용 코드로 개선**

`CrudRepository` 상속을 제거하고, `RedisTemplate`을 사용하여 HASH 자료구조에 직접 데이터를 쓰고 읽도록 리포지토리를 재구성합니다.

```kotlin
// OrderRepository.kt (수정 후)
@Repository
class OrderRepository(
    private val redisTemplate: RedisTemplate<String, Any>,
    private val objectMapper: ObjectMapper
) {
    companion object {
        private const val KEY_PREFIX = "orders:"
    }

    fun save(order: Order) {
        val key = KEY_PREFIX + order.id
        val valueMap = objectMapper.convertValue(order, object : TypeReference<Map<String, Any>>() {})
        redisTemplate.opsForHash().putAll(key, valueMap)
    }

    fun findById(id: String): Order? {
        val key = KEY_PREFIX + id
        val valueMap = redisTemplate.opsForHash().entries(key)
        if (valueMap.isEmpty()) {
            return null
        }
        return objectMapper.convertValue(valueMap, Order::class.java)
    }
}
```

수정된 `save` 메서드는 `orders:<ID>` 형태의 HASH 키만 생성하며, 더 이상 인덱스용 `orders` 키를 만들지 않습니다. 만약 `findAll()`이나 `count()`와 같은 전체 조회 기능이 반드시 필요하다면, BigKey를 피할 수 있는 별도의 데이터 구조(예: Time-bucket 기반 인덱싱)를 신중하게 설계해야 합니다.

### 운영 환경의 BigKey 확인 방법

운영 중인 Redis에서 BigKey가 있는지 확인하려면 Redis CLI의 `--bigkeys` 옵션을 사용할 수 있습니다. 이 옵션은 키스페이스를 스캔하면서 데이터 타입별로 가장 큰 키를 출력합니다.

```bash
$ redis-cli --bigkeys

# ... 스캔 진행 ...

-------- summary -------
Sampled 239253 keys in the keyspace!
Total key length in bytes is 8956383 (avg len 37.43)

Biggest string found 'key:2918' has 108 bytes
Biggest   list found 'mylist' has 40 members
Biggest    set found 'orders' has 2000000 members  # <-- BigKey 후보
Biggest   hash found 'user:1000' has 38 fields
```

위와 같이 의심스러운 `SET` 타입의 키를 발견했다면, `SCARD` 명령어로 정확한 멤버 개수를 확인할 수 있습니다.

```bash
> SCARD orders
(integer) 2000000
```

### 결론

Spring Data Redis의 `@RedisHash`와 `CrudRepository`는 생산성을 높여주는 도구입니다. 다만 추상화에는 비용이 있고, 내부 동작을 모른 채 쓰면 데이터 규모가 커졌을 때 성능 문제가 드러날 수 있습니다.

특히 데이터가 계속 누적되는 엔티티라면 `CrudRepository`의 자동 인덱싱 기능에 기대기보다, `RedisTemplate`으로 저장 구조를 명시적으로 설계하는 편이 안전합니다.

---
*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
