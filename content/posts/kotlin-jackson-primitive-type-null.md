---
title: "Kotlin과 Jackson: non-null 원시 타입이 0으로 역직렬화되는 문제 분석"
description: "Kotlin 데이터 클래스와 Jackson을 사용할 때, JSON에 non-null 원시 타입(Long, Int) 필드가 누락되면 예외 대신 0이 할당되는 문제를 심층 분석합니다. 참조 타입과 동작이 다른 이유를 알아보고, DeserializationFeature, @JsonProperty, Nullable 타입 등 각 해결 방안의 장단점을 비교하여 최적의 전략을 제시합니다."
date: 2025-01-11T16:40:31+09:00
tags: ["kotlin", "jackson", "spring", "deserialization"]
---

## 들어가며: 혼란을 야기하는 Jackson의 동작

Kotlin의 가장 큰 장점 중 하나는 컴파일 시점에 null 안전성(Null Safety)을 보장하여 `NullPointerException`을 효과적으로 방지하는 것입니다. 하지만 JSON 라이브러리인 Jackson과 함께 사용할 때, 이 null 안전성이 우리가 기대하는 것과 다르게 동작하여 잠재적인 버그를 유발하는 경우가 있습니다.

가장 대표적인 사례가 **JSON 역직렬화(Deserialization) 시 non-null 원시 타입(primitive type) 필드가 누락된 경우** 입니다.

-   `String`과 같은 **참조 타입** 필드가 누락되면: Jackson이 **예외를 발생**시킵니다. (예상대로 동작)
-   `Long`, `Int`와 같은 **원시 타입** 필드가 누락되면: Jackson이 예외 없이 **기본값(0)을 할당** 합니다. (예상과 다른 동작)

이러한 비일관적인 동작은 개발자에게 혼란을 주고, 데이터가 누락되었음에도 요청이 성공한 것처럼 보여 심각한 버그로 이어질 수 있습니다. 이 글에서는 실제 테스트를 통해 이 문제를 재현하고, 근본 원인을 분석하며, 명확한 해결 방안을 제시하겠습니다.

## 문제 상황 재현

다음과 같은 간단한 `Video` 데이터 클래스와 이를 Request Body로 받는 컨트롤러가 있다고 가정해 보겠습니다.

**데이터 클래스:**
```kotlin
data class Video(
    val name: String,      // Non-null 참조 타입
    val durationMs: Long   // Non-null 원시 타입
)
```

**컨트롤러:**
```kotlin
@RestController
@RequestMapping("/demo")
class DemoController {
    @PostMapping
    fun createVideo(@RequestBody request: Video): String {
        // 역직렬화된 객체의 내용을 확인하기 위해 그대로 반환
        return request.toString()
    }
}
```

### API 테스트를 통한 검증

**1. 정상적인 요청 (모든 필드 포함)**
-   **요청**: `{"name": "Sample Video", "durationMs": 3000}`
-   **결과**: `200 OK` 응답, `Video(name=Sample Video, durationMs=3000)` 반환. (정상)

**2. `name`(참조 타입) 필드 누락**
-   **요청**: `{"durationMs": 3000}`
-   **결과**: `400 Bad Request` 응답. non-null 타입인 `name` 파라미터에 `null` 값을 주입할 수 없다는 예외 발생. (예상대로 동작)
    ```
    JSON parse error: Instantiation of [simple type, class com.example.demo.Video] value failed for JSON property name due to missing (therefore NULL) value for creator parameter name which is a non-nullable type
    ```

**3. `durationMs`(원시 타입) 필드 누락**
-   **요청**: `{"name": "Sample Video"}`
-   **결과**: **`200 OK` 응답**, `Video(name=Sample Video, durationMs=0)` 반환. (예상과 다른 동작)

`durationMs` 필드가 누락되었음에도 불구하고 요청이 성공하고, 해당 필드에는 기본값인 `0`이 할당되었습니다. 이는 데이터 무결성을 해칠 수 있는 심각한 문제입니다.

## 근본 원인: Jackson은 왜 다르게 동작할까?

이 문제의 원인을 이해하려면 `jackson-module-kotlin`의 동작 방식과 JVM의 타입 시스템을 함께 알아야 합니다.

`jackson-module-kotlin`은 Jackson이 Kotlin의 non-null 타입을 인식하고, `null` 값이 non-null 필드에 할당되려고 할 때 예외를 발생시키는 역할을 합니다.

1.  **참조 타입의 경우 (`String`)**: JSON에 `name` 필드가 없으면, Jackson은 생성자의 `name` 파라미터에 `null`을 전달하려고 시도합니다. 이때 `jackson-module-kotlin`이 이를 감지하고 "non-null 파라미터에 null을 할당할 수 없다"는 예외를 발생시킵니다.

2.  **원시 타입의 경우 (`Long`)**: JSON에 `durationMs` 필드가 없으면, Jackson의 **기본 동작 방식** 에 따라 `null` 대신 해당 원시 타입의 **기본값(primitive default value)** 인 `0L`을 생성자의 `durationMs` 파라미터에 전달합니다. `null`이 전달된 것이 아니기 때문에, `jackson-module-kotlin`의 null 체크 로직은 동작하지 않고 역직렬화는 그대로 성공하게 됩니다.

결론적으로, 이 현상은 Jackson 라이브러리 자체의 기본 동작 방식과 Kotlin의 null 안전성 메커니즘 간의 상호작용 때문에 발생하는 것입니다.

## 해결 방안

이 문제를 해결하기 위한 세 가지 주요 방법이 있으며, 각각의 장단점을 비교해 보겠습니다.

### 1. `DeserializationFeature` 전역 설정

Jackson의 역직렬화 동작 방식을 애플리케이션 전역에 걸쳐 변경하는 방법입니다. `FAIL_ON_NULL_FOR_PRIMITIVES` 기능을 활성화하면, 원시 타입 필드에 `null`이 할당될 때 예외를 발생시킵니다.

```kotlin
// Spring Boot 환경에서의 설정 예시
@Configuration
class JacksonConfig {
    @Bean
    fun jackson2ObjectMapperBuilder(): Jackson2ObjectMapperBuilder {
        return Jackson2ObjectMapperBuilder()
            .featuresToEnable(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES)
    }
}
```

-   **장점**: 한 번의 설정으로 애플리케이션 전체에 일관된 동작을 보장할 수 있습니다. 가장 확실하고 강력한 방법입니다.
-   **단점**: 기존에 Jackson의 기본 동작(0으로 할당)에 의존하던 코드가 있었다면 예기치 않은 장애를 유발할 수 있습니다. (레거시 프로젝트에 적용 시 주의 필요)

### 2. `@JsonProperty(required = true)` 사용

필드 레벨에서 필수 여부를 명시하는 방법입니다. 이 어노테이션을 사용하면 Jackson은 해당 필드가 JSON에 반드시 존재해야 함을 인지하고, 누락 시 예외를 발생시킵니다.

```kotlin
data class Video(
    val name: String,
    @JsonProperty(required = true)
    val durationMs: Long
)
```

-   **장점**: 특정 필드에만 선별적으로 적용할 수 있어 부작용이 적습니다. 코드상으로 해당 필드가 필수임을 명확히 문서화하는 효과도 있습니다.
-   **단점**: 필수 필드가 많을 경우 모든 필드에 어노테이션을 추가해야 하므로 코드가 다소 장황해질 수 있습니다.

### 3. Nullable 타입 사용 (`Long?`)

데이터 모델 자체를 변경하여 해당 필드가 `null`이 될 수 있음을 명시하는 방법입니다.

```kotlin
data class Video(
    val name: String,
    val durationMs: Long?
)
```

-   **장점**: Kotlin의 타입 시스템을 가장 잘 활용하는 방법입니다. 필드가 선택적(optional)이라는 비즈니스 요구사항을 코드에 명확히 반영할 수 있습니다.
-   **단점**: 필드가 `null`이 될 수 있으므로, 이후 로직에서 `durationMs`를 사용할 때마다 `null` 체크(`?:` 연산자 등)가 필요합니다. 만약 비즈니스 로직상 이 필드가 절대 `null`이어서는 안 된다면, 이 방법은 문제를 해결하는 것이 아니라 단순히 뒤로 미루는 것일 수 있습니다.

## 권장 전략 및 결론

상황에 따른 최적의 해결 방안은 다음과 같이 정리할 수 있습니다.

1.  **새로운 프로젝트를 시작한다면**: 전역적으로 `DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES`를 활성화하여 처음부터 데이터 무결성을 강제하는 것이 가장 좋습니다.
2.  **기존 프로젝트에 적용해야 한다면**: 전역 설정의 영향을 파악하기 어려울 경우, 문제가 되는 특정 필드에 `@JsonProperty(required = true)`를 사용하여 점진적으로 수정하는 것이 가장 안전합니다.
3.  **해당 필드가 비즈니스 로직상 진정으로 선택적이라면**: `Long?`와 같이 Nullable 타입을 사용하여 모델링하는 것이 가장 올바른 접근 방식입니다.

Kotlin과 Jackson의 조합은 매우 강력하지만, 두 시스템의 경계에서 발생하는 이러한 미묘한 동작 차이를 이해하는 것은 중요합니다. 위에서 제시된 해결 방안들을 통해, 코드의 안정성과 데이터의 무결성을 모두 확보하는 견고한 애플리케이션을 만드시길 바랍니다.