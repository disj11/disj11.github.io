---
title: "Sparse Fieldsets(Field Mask)로 필요한 필드만 내려주기"
date: 2026-01-05T20:36:31+09:00
url: "/api-field-mask-guide/"
description: "GraphQL까지 도입하기에는 부담스러울 때, REST API 구조를 유지하면서 오버페칭을 줄이는 Sparse Fieldsets(Field Mask) 패턴을 정리합니다. 개념, 적용 기준, Spring Boot 구현 예제를 함께 다룹니다."
tags: ["api-design", "performance", "spring-boot", "kotlin", "pattern"]
---

# API가 필요한 것보다 많은 정보를 준다면

API를 만들다 보면 클라이언트가 실제로 쓰는 것보다 훨씬 많은 데이터를 응답으로 내려주는 경우가 생깁니다. 보통 **오버페칭(Over-fetching)** 이라고 부르는 문제입니다.

화면에는 상품 이름과 가격만 있으면 되는데, API는 상품 설명, 리뷰 목록, 판매자 정보, 배송 정책까지 한꺼번에 내려줍니다. 필드가 몇 개 안 될 때는 넘어갈 수 있지만, 모바일 환경이나 리스트 화면에서는 응답 크기와 렌더링 비용이 곧 체감 속도로 이어집니다.

이 글에서는 이 문제를 줄이기 위한 방법으로 **Sparse Fieldsets** 패턴을 다룹니다.

---

## 1. Sparse Fieldsets란?

**Sparse Fieldsets**는 클라이언트가 API 요청에서 **"필요한 필드만 받겠다"** 고 명시하는 패턴입니다. 예를 들어, 상품 목록 API에서 전체 정보가 아니라 `id`와 `name`만 필요하다면 이렇게 요청합니다.

```sh
# 요청 (필요한 필드만 명시)
GET /products?fields=id,name
```

```json
// 응답 (요청한 필드만 반환)
[
  { "id": 1, "name": "스마트폰" },
  { "id": 2, "name": "노트북" }
]
```

기존 API라면 `description`, `reviews`, `sellerInfo` 같은 필드가 모두 포함되었겠지만, 이 요청에서는 필요한 값만 내려옵니다.

### 📖 같은 개념을 부르는 여러 이름

기술 스택이나 회사마다 이름은 조금씩 다르지만, 아이디어는 같습니다. 클라이언트가 응답에 포함할 필드를 지정합니다.

| 용어               | 사용처                                                                                                               |
| ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| Sparse Fieldsets   | [JSON:API](https://jsonapi.org/format/#fetching-sparse-fieldsets) 스펙 (REST 진영의 표준적 명칭)                     |
| Field Mask         | [Google API (AIP-161)](https://google.aip.dev/161) 및 gRPC/Protobuf 생태계                                           |
| Projections        | [LinkedIn API](https://learn.microsoft.com/en-us/linkedin/shared/api-guide/concepts/projections) 등                  |

> 이 글에서는 REST 환경에서 가장 널리 쓰이는 **Sparse Fieldsets**라는 용어를 사용하되, Google/gRPC 관련 내용에서는 **Field Mask**로도 지칭합니다.

---

## 2. GraphQL과 무엇이 다를까?

오버페칭 문제를 이야기할 때 가장 먼저 떠오르는 기술은 **GraphQL**입니다. Sparse Fieldsets와 어떤 차이가 있는지 먼저 비교해보겠습니다.

### GraphQL: 표현력은 좋지만 도입 비용이 큰 선택

GraphQL은 클라이언트가 필요한 데이터 구조를 쿼리 언어로 정의해서 요청하고, 서버가 그 형태에 맞춰 응답하는 방식입니다.

*   **장점**: 클라이언트가 필요한 데이터를 꽤 세밀하게 고를 수 있습니다. 여러 리소스의 연관 관계(Graph)를 한 번 요청으로 가져올 수 있어 오버페칭과 언더페칭(Under-fetching)을 함께 줄입니다.
*   **단점**: 도입 비용이 큽니다. 기존 REST API와 다른 생태계(Schema, Resolver 등)를 새로 운영해야 합니다.

### Sparse Fieldsets: REST API를 유지하는 접근

[JSON:API 스펙](https://jsonapi.org/format/#fetching-sparse-fieldsets)에서 정의한 이 방식은, REST API 요청에 `fields=id,name`과 같이 필요한 필드 목록을 함께 보내는 것입니다.

*   **장점**: 기존 REST API 구조를 크게 바꾸지 않고 적용할 수 있습니다. 구현이 비교적 단순하고, 클라이언트도 URL 쿼리 파라미터 형태로 받아들이기 쉽습니다.
*   **단점**: GraphQL처럼 복잡한 그래프 관계를 깊이 탐색하기에는 표현력이 부족합니다.

### 한눈에 비교

| 기준 | GraphQL | Sparse Fieldsets |
|---|---|---|
| 도입 비용 | 높음 (새 스택 구축 필요) | 낮음 (기존 REST에 추가) |
| 유연성 | 매우 높음 (중첩 쿼리 가능) | 중간 (단일 리소스 필드 선택) |
| 학습 곡선 | 가파름 | 완만함 |
| 적합한 상황 | 신규 프로젝트, 복잡한 관계형 데이터 | 기존 REST API 개선 |

---

## 3. 언제 선택해야 할까?

그렇다면 언제 Sparse Fieldsets를 선택하는 편이 좋을까요?

### ✅ 이런 상황이라면 좋은 선택입니다

1.  **이미 운영 중인 REST API가 있는 경우**: 시스템을 크게 갈아엎지 않고 응답 크기와 조회 비용만 줄이고 싶을 때 적용하기 좋습니다.
2.  **클라이언트마다 필요한 데이터가 다른 경우**: 모바일 리스트 화면에서는 썸네일과 제목만, 웹 상세 화면에서는 전체 정보가 필요한 것처럼, 동일한 리소스에 대해 클라이언트마다 필요한 정보의 양이 다를 때 유연하게 대처할 수 있습니다.
3.  **Public API를 만드는 경우**: [Google](https://google.aip.dev/161), [Netflix](https://netflixtechblog.com/practical-api-design-at-netflix-part-1-using-protobuf-fieldmask-310fd639cd80), [LinkedIn](https://learn.microsoft.com/en-us/linkedin/shared/api-guide/concepts/projections) 등에서도 비슷한 패턴을 사용합니다.

### ❌ 다른 대안을 고려해야 하는 경우

1.  **새 프로젝트를 시작하며 복잡한 관계형 데이터를 다루는 경우**: 처음부터 설계한다면 GraphQL이 더 나은 개발 경험을 제공할 수 있습니다.
2.  **단순한 내부 관리 도구(Admin)인 경우**: 트래픽이 적고 성능이 중요하지 않다면, 굳이 이런 최적화를 적용해 개발 복잡도를 높일 필요가 없습니다.

---

## 4. 조회 말고도 쓸 수 있는 곳

Sparse Fieldsets는 단순히 응답 필드를 줄이는 데서 끝나지 않습니다.

### API 진화와 하위 호환성

Netflix 기술 블로그에서도 강조하듯, 이 패턴의 숨겨진 장점은 **API 변경에 대한 내성**입니다. 클라이언트가 필요한 필드를 명시적으로 요청하기 때문에, 서버가 새로운 필드를 추가하더라도 기존 클라이언트는 영향을 받지 않습니다.

즉, 새 필드가 추가될 때마다 `v1`, `v2`를 나누지 않아도 기존 클라이언트를 비교적 안전하게 유지할 수 있습니다.

### 쓰기 작업(Update/PATCH)에서의 활용

[Google AIP-161](https://google.aip.dev/161)에서는 이 패턴을 조회(GET)뿐 아니라 수정(PATCH) 작업에도 사용합니다.

```sh
# "status 필드만 변경하겠다"고 명시적으로 선언
PATCH /orders/123?update_mask=status
Content-Type: application/json

{ "status": "shipped" }
```

`update_mask` 필드를 사용하면 "이 리소스의 `status` 필드만 바꾸겠다"는 의도를 명시할 수 있습니다. 실수로 다른 필드를 덮어쓰는 사고도 줄일 수 있습니다.

### 지연 로딩과 함께 써야 효과가 난다

필드를 선택해서 내려준다고 해도, 서버에서 모든 필드를 미리 조회해둔다면 성능 이점이 반감됩니다. 예를 들어, 클라이언트가 `reviews` 필드를 요청하지 않았는데도 서버가 DB에서 리뷰 데이터를 조회한다면 불필요한 연산이 발생하죠.

이 문제를 피하려면 **지연 로딩(Lazy Loading)** 을 함께 써야 합니다. 요청된 필드일 때만 실제 데이터를 조회하도록 만들면, 네트워크 비용뿐 아니라 서버의 DB 조회 비용도 줄일 수 있습니다. 아래 구현 섹션에서는 `Supplier`를 사용해 이 부분을 처리합니다.

> ⚠️ **주의**: 프로덕션 환경에서는 와일드카드(`*`) 사용을 지양하세요. 모든 필드를 요청하는 것은 오버페칭을 유발할 뿐만 아니라, 향후 필드가 추가되었을 때 예상치 못한 데이터 노출이나 성능 저하로 이어질 수 있습니다.

---

## 5. Sparse Fieldsets 구현하기 (Kotlin & Spring)

이제 구현으로 넘어가겠습니다. 이 패턴을 제대로 구현하려면 **'값이 `null`인 필드'** 와 **'요청하지 않아서 생략된 필드'** 를 구분해야 합니다. 이를 위해 `JsonWrapper`라는 작은 래퍼를 둡니다.

### 전체 흐름 미리보기

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. 클라이언트 요청                                                       │
│    GET /products/1?fields=id,name                                       │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. Interceptor가 요청 파라미터를 ThreadLocal에 저장                      │
│    SparseFieldsetsContext.setRequestedFields(["id", "name"])            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 컨트롤러 → 서비스 → DTO 생성 (모든 필드를 JsonWrapper로 감쌈)           │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Serializer가 ThreadLocal 확인 후, 요청된 필드만 JSON에 출력             │
│    → { "id": 1, "name": "스마트폰" }                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1단계: 요청 정보 저장하기 (`SparseFieldsetsInterceptor`)

먼저 클라이언트가 요청한 필드 목록을 스레드별로 저장해둘 공간이 필요합니다.

```kotlin
// 1. ThreadLocal로 요청 컨텍스트 관리
object SparseFieldsetsContext {
    private val requestedFieldsHolder = ThreadLocal<Set<String>>()
    
    fun setRequestedFields(fields: Set<String>) { requestedFieldsHolder.set(fields) }
    fun getRequestedFields(): Set<String> = requestedFieldsHolder.get() ?: emptySet()
    fun clear() { requestedFieldsHolder.remove() }
}

// 2. Interceptor에서 파라미터 파싱
@Component
class SparseFieldsetsInterceptor : HandlerInterceptor {
    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        request.getParameter("fields")?.let {
            val fields = it.split(",").map(String::trim).toSet()
            SparseFieldsetsContext.setRequestedFields(fields)
        }
        return true
    }

    override fun afterCompletion(request: HttpServletRequest, response: HttpServletResponse, handler: Any, ex: Exception?) {
        SparseFieldsetsContext.clear() // 메모리 누수 방지, 필수!
    }
}
```

### 2단계: `JsonWrapper`와 DTO 설계

모든 필드를 그냥 내보내는 것이 아니라, `JsonWrapper`로 감싸서 직렬화 로직이 개입할 틈을 만들어줍니다.

```kotlin
// Sparse Fieldsets 대상 필드를 감싸는 Wrapper
class JsonWrapper<T>(val value: T)

@JsonInclude(JsonInclude.Include.NON_EMPTY)
data class ProductResponse(
    val id: JsonWrapper<Long>,
    val name: JsonWrapper<String>,
    // 다른 엔티티 조회가 필요한 필드는 Supplier로 감싸서 지연 로딩
    val reviews: JsonWrapper<Supplier<List<ReviewDto>>>
)
```

> ⚠️ **포인트**: DTO는 **서비스 레이어**에서 생성됩니다. 이때 `reviews`처럼 별도 조회가 필요한 필드는 실제 DB 조회 로직을 `Supplier` 안에 넣어둡니다. 이 시점에서는 **아직 리뷰 조회가 실행되지 않습니다.**

```kotlin
// 서비스 레이어 - DTO 생성 시점
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val reviewRepository: ReviewRepository
) {
    fun getProduct(id: Long): ProductResponse {
        val product = productRepository.findById(id) ?: throw NotFoundException()
        
        return ProductResponse(
            id = JsonWrapper(product.id),
            name = JsonWrapper(product.name),
            // ⚠️ 여기서 리뷰를 조회하지 않음! Supplier 안에 로직만 담아둠
            reviews = JsonWrapper(memoize { 
                reviewRepository.findByProductId(id).map { it.toDto() }
            })
        )
    }
}

// 컨트롤러 - 서비스에서 받은 DTO를 그대로 반환
@RestController
class ProductController(private val productService: ProductService) {
    @GetMapping("/products/{id}")
    @UseSparseFieldsets
    fun getProduct(@PathVariable id: Long): ProductResponse {
        return productService.getProduct(id)
    }
}
```

**클라이언트가 `?fields=id,name`으로 요청하면:**
- Serializer가 `reviews` 필드는 요청되지 않았음을 확인
- `Supplier.get()`을 호출하지 않음 → **리뷰 DB 조회 자체가 발생하지 않음!**

**클라이언트가 `?fields=id,name,reviews`로 요청하면:**
- Serializer가 `reviews` 필드도 요청되었음을 확인
- `Supplier.get()` 호출 → 이때 비로소 리뷰 조회 실행

### 3단계: 마법이 일어나는 `SparseFieldsetsSerializer`

이제 Jackson Serializer가 실제로 필드를 걸러내는 로직을 수행합니다.

```kotlin
class SparseFieldsetsSerializer : JsonSerializer<JsonWrapper<*>>() {
    override fun serialize(wrapper: JsonWrapper<*>, gen: JsonGenerator, serializers: SerializerProvider) {
        val currentFieldName = gen.outputContext.currentName
        val requestedFields = SparseFieldsetsContext.getRequestedFields()

        // 요청된 필드가 아니면 빈 값을 써서 @JsonInclude(NON_EMPTY)에 의해 제외되도록 함
        if (requestedFields.isNotEmpty() && currentFieldName !in requestedFields) {
            return // 아무것도 쓰지 않으면 NON_EMPTY 설정과 함께 필드가 제외됨
        }

        val rawValue = wrapper.value
        // Supplier라면 이때 실행 (지연 로딩 효과!)
        val finalValue = if (rawValue is Supplier<*>) rawValue.get() else rawValue

        if (finalValue != null) {
            serializers.defaultSerializeValue(finalValue, gen)
        } else {
            gen.writeNull()
        }
    }

    // 빈 값일 때 필드를 제외하기 위한 설정
    override fun isEmpty(provider: SerializerProvider, value: JsonWrapper<*>): Boolean {
        val requestedFields = SparseFieldsetsContext.getRequestedFields()
        // 요청된 필드 목록이 있고, 현재 필드가 포함되지 않으면 "비어있는 것"으로 처리
        return requestedFields.isNotEmpty()
    }
}
```

> **💡 동작 원리**: 요청되지 않은 필드는 `isEmpty()`가 `true`를 반환하고, DTO의 `@JsonInclude(NON_EMPTY)` 설정과 결합되어 JSON 출력에서 완전히 제외됩니다.

### 4단계: Update/PATCH에 적용하기 (update_mask)

앞서 설명한 것처럼, 이 패턴은 수정 작업에도 유용합니다. `update_mask`로 지정된 필드만 업데이트하도록 구현해봅시다.

먼저 리플렉션을 사용한 범용 유틸리티를 만들어두면, 필드가 추가되어도 코드 수정 없이 처리할 수 있습니다.

```kotlin
import kotlin.reflect.KMutableProperty1
import kotlin.reflect.full.memberProperties

// 범용 Field Mask 적용 유틸리티
// source에서 target으로, fieldMask에 포함된 필드만 복사
fun <T : Any> applyFieldMask(
    target: T,
    source: Map<String, Any?>,  // Request를 Map으로 변환하여 전달
    fieldMask: Set<String>
) {
    target::class.memberProperties
        .filterIsInstance<KMutableProperty1<T, Any?>>()
        .filter { it.name in fieldMask && it.name in source }
        .forEach { prop ->
            prop.set(target, source[prop.name])
        }
}
```

이제 서비스 계층에서 간단하게 사용할 수 있습니다.

```kotlin
// 컨트롤러
@PatchMapping("/orders/{id}")
fun updateOrder(
    @PathVariable id: Long,
    @RequestParam("update_mask") updateMask: String,
    @RequestBody request: OrderUpdateRequest
): OrderResponse {
    val fieldsToUpdate = updateMask.split(",").map(String::trim).toSet()
    return orderService.partialUpdate(id, request, fieldsToUpdate)
}

// 서비스
fun partialUpdate(id: Long, request: OrderUpdateRequest, fieldsToUpdate: Set<String>): OrderResponse {
    val order = orderRepository.findById(id) ?: throw NotFoundException()
    
    // fieldMask에 포함된 필드만 request에서 order로 복사
    applyFieldMask(
        target = order,
        source = request.toMap(),  // { "status": "shipped", "memo": "빠른 배송 요청" }
        fieldMask = fieldsToUpdate
    )
    
    return orderRepository.save(order).toResponse()
}

// Request DTO에 toMap() 확장 함수 추가
fun OrderUpdateRequest.toMap(): Map<String, Any?> = mapOf(
    "status" to this.status,
    "memo" to this.memo,
    // ... 다른 필드들
)
```

이 방식의 장점은 필드가 늘어나도 유틸리티 코드를 수정할 필요가 없다는 점입니다. 클라이언트가 의도하지 않은 필드를 실수로 덮어쓰는 것도 방지할 수 있습니다.

---

## 6. 실무 적용 팁

실제 운영 환경에 적용할 때 유용한 팁들을 모았습니다.

### 팁 1: 어노테이션 자동 적용 (`BeanSerializerModifier`)

모든 필드마다 `@field:JsonSerialize(...)`를 붙이는 건 번거롭습니다. Jackson의 `BeanSerializerModifier`를 사용하면 `JsonWrapper` 타입을 자동으로 인식해 Serializer를 적용할 수 있습니다.

```kotlin
@Component
class SparseFieldsetsModule : SimpleModule() {
    override fun setupModule(context: SetupContext) {
        context.addBeanSerializerModifier(object : BeanSerializerModifier() {
            override fun changeProperties(
                config: SerializationConfig,
                beanDesc: BeanDescription,
                beanProperties: List<BeanPropertyWriter>
            ): List<BeanPropertyWriter> {
                for (writer in beanProperties) {
                    if (JsonWrapper::class.java.isAssignableFrom(writer.type.rawClass)) {
                        writer.assignSerializer(SparseFieldsetsSerializer())
                    }
                }
                return beanProperties
            }
        })
    }
}
```
이제 DTO가 깔끔해집니다:
```kotlin
data class ProductResponse(
    val id: JsonWrapper<Long>,
    val name: JsonWrapper<String> // 어노테이션 생략 가능!
)
```

### 팁 2: 필요한 곳에만 적용하기 (`@UseSparseFieldsets`)

모든 API가 이 기능을 필요로 하지는 않습니다. 커스텀 어노테이션으로 특정 API에만 적용하도록 제한하는 것이 안전합니다.

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class UseSparseFieldsets

// Interceptor에서 어노테이션 체크
override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
    if (handler is HandlerMethod && handler.hasMethodAnnotation(UseSparseFieldsets::class.java)) {
        // Sparse Fieldsets 로직 수행
    }
    return true
}
```

### 팁 3: Supplier 중복 호출 방지 (`memoize`)

`Supplier.get()`이 직렬화 과정에서 여러 번 호출될 수 있습니다. 무거운 로직(DB 조회 등)은 최초 1회만 실행되도록 캐싱해야 합니다.

```kotlin
// Memoization을 위한 Wrapper 클래스
class Memoized<T>(private val supplier: () -> T) : Supplier<T> {
    private val cached: T by lazy(LazyThreadSafetyMode.NONE) { supplier() }
    override fun get(): T = cached
}

// 편의 함수
fun <T> memoize(supplier: () -> T): Supplier<T> = Memoized(supplier)

// 사용 예시
val reviews = JsonWrapper(memoize { reviewRepository.findByProductId(id) })
```

---

## 7. 마치며

Sparse Fieldsets 패턴은 **API의 유연함** 과 **시스템의 단순함** 사이에서 타협점을 찾는 방법입니다. GraphQL을 새로 들이지 않더라도, REST API에서 자주 생기는 오버페칭 문제를 꽤 줄일 수 있습니다.

특히 지연 로딩과 함께 쓰면 네트워크 비용뿐 아니라 서버 쪽의 불필요한 조회도 줄일 수 있습니다.

API가 매번 너무 많은 값을 내려주고 있다면, 먼저 몇몇 조회 API부터 Sparse Fieldsets를 적용해볼 만합니다.

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
