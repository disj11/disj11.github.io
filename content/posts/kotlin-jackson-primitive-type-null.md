---
title: "Kotlin과 Jackson 사용 시 주의할 점: Primitive Type 필드의 Null 처리"
description: "Kotlin과 Jackson을 사용할 때 primitive 타입의 필드가 누락된 경우, 예기치 않은 동작이 발생할 수 있습니다. 이 글에서는 실제 사례와 테스트를 통해 이 문제를 재현하고, 다양한 해결 방안을 비교 분석합니다."
date: 2025-01-11T16:40:31+09:00
url: "/kotlin-jackson-primitive-type-null/"
tags: [kotlin]
---

## 개요

Kotlin과 Jackson을 사용할 때 primitive 타입의 필드가 누락된 경우, 예기치 않은 동작이 발생할 수 있습니다. 구체적으로:

1. Kotlin의 non-null 타입으로 선언된 primitive 필드가 JSON에서 누락된 경우
2. Jackson이 이를 기본값(0, false 등)으로 처리하는 현상
3. String 과 같은 reference 타입이 누락된 경우는 예외가 발생하므로 혼란의 여지가 있음

이 글에서는 실제 사례와 테스트를 통해 이 문제를 재현하고, 다양한 해결 방안을 비교 분석하겠습니다.

## Kotlin의 타입 시스템 이해

Kotlin의 타입 시스템에서는 모든 타입이 기본적으로 non-null입니다. 그러나 Java와의 상호운용성을 위해 플랫폼 타입이라는 특별한 타입이 존재하며, 이는 nullability가 불명확한 상태를 나타냅니다.

## 예상되는 동작과 실제 동작

**예상되는 동작**
- non-null로 선언된 필드가 누락된 경우 요청이 실패해야 함
- 특히 primitive 타입의 필드가 누락된 경우에도 동일하게 실패해야 함

**실제 동작**
- String과 같은 reference 타입이 누락된 경우 400 Bad Request 발생
- Long과 같은 primitive 타입이 누락된 경우 기본값(0)이 할당되어 요청이 성공

## 문제 상황

현재 서비스에는 다음과 같은 데이터 클래스가 존재합니다:

```kotlin
data class Video(
    val name: String,
    val durationMs: Long,
)
```

이를 REST API의 Request Body로 받고 있습니다:

```kotlin
@RestController
@RequestMapping("/demo")
class DemoController {
    @PostMapping
    fun createVideo(@RequestBody request: Video): String {
        return request.toString()
    }
}
```

## 문제 검증

### API 테스트를 통한 검증

**1. 정상적인 요청**
```bash
curl -X POST \
-H "Content-Type: application/json" \
-d '{
    "name": "Sample Video",
    "durationMs": 3000
}' \
http://localhost:8080/demo
```
결과: 200 OK 응답

**2. name이 누락된 요청**
```bash
curl -X POST \
-H "Content-Type: application/json" \
-d '{
    "durationMs": 3000
}' \
http://localhost:8080/demo
```
결과: 400 Bad Request
```
JSON parse error: Instantiation of [simple type, class com.example.demo.Video] value failed for JSON property name due to missing (therefore NULL) value for creator parameter name which is a non-nullable type
```

**3. durationMs가 누락된 요청**
```bash
curl -X POST \
-H "Content-Type: application/json" \
-d '{
    "name": "Sample Video"
}' \
http://localhost:8080/demo
```
결과: 예상과 달리 200 OK 응답 (durationMs가 0으로 설정됨)

### 단위 테스트를 통한 검증

```kotlin
@WebMvcTest(DemoController::class)
class DemoControllerTest : FunSpec() {
    @Autowired
    private lateinit var mockMvc: MockMvc

    init {
        extension(SpringExtension)

        test("모든 필드가 포함된 경우 200 응답을 반환한다") {
            val validJson = """
                {
                    "name": "Sample Video",
                    "durationMs": 3000
                }
            """.trimIndent()

            mockMvc.perform(
                MockMvcRequestBuilders.post("/demo")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(validJson)
            ).andExpect(MockMvcResultMatchers.status().isOk)
        }

        test("name이 누락된 경우 400 응답을 반환한다") {
            val invalidJson = """
                {
                    "durationMs": 3000
                }
            """.trimIndent()

            mockMvc.perform(
                MockMvcRequestBuilders.post("/demo")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(invalidJson)
            ).andExpect(MockMvcResultMatchers.status().isBadRequest)
        }

        test("durationMs가 누락된 경우 400 응답을 반환한다") {
            val invalidJson = """
                {
                    "name": "Sample Video"
                }
            """.trimIndent()

            mockMvc.perform(
                MockMvcRequestBuilders.post("/demo")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(invalidJson)
            ).andExpect(MockMvcResultMatchers.status().isBadRequest) // 실패
        }
    }
}
```

## Jackson의 동작 방식

Jackson은 missing 필드와 null 필드를 다르게 처리합니다. Missing 필드의 경우 primitive 타입에 대해 다음과 같은 기본값을 할당합니다:
- `Long`, `Int`: 0
- `Boolean`: false
- `Double`, `Float`: 0.0

## 해결 방안

### 1. DeserializationFeature 설정
전역적으로 primitive 타입의 null 처리 방식을 변경하는 방법입니다.

```kotlin
@Configuration
class JacksonConfig {
    @Bean
    fun jackson2ObjectMapperBuilder(): Jackson2ObjectMapperBuilder {
        return Jackson2ObjectMapperBuilder()
            .featuresToEnable(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES)
    }
}
```

### 2. Nullable 타입 사용
Kotlin의 null safety를 활용하여 명시적으로 null을 처리하는 방법입니다.

```kotlin
data class Video(
    val name: String,
    val durationMs: Long?
)
```

### 3. JsonProperty 어노테이션 사용
필드별로 필수 여부를 지정하는 방법입니다.

```kotlin
data class Video(
    val name: String,
    @JsonProperty(required = true)
    val durationMs: Long
)
```

## 각 해결방안의 장단점

**DeserializationFeature 설정**
- 장점: 전역적으로 일관된 동작 보장
- 단점: 기존 코드에 영향을 줄 수 있음

**Nullable 타입 사용**
- 장점: Kotlin의 null safety 활용 가능
- 단점: null 처리 로직 추가 필요

**JsonProperty 어노테이션**
- 장점: 필드별로 세밀한 제어 가능
- 단점: 모든 필드에 개별적으로 설정 필요

## 권장되는 해결방안

1. 새로운 프로젝트 시작 시 `FAIL_ON_NULL_FOR_PRIMITIVES` 설정 고려
2. 기존 프로젝트의 경우 `@JsonProperty(required = true)` 사용 권장
3. Nullable이 허용되는 필드는 명시적으로 `Long?`과 같이 선언

## 결론

Kotlin과 Jackson을 함께 사용할 때는 primitive 타입의 null 처리에 특별한 주의가 필요합니다. 실제 테스트를 통해 확인한 것처럼, 예상치 못한 동작이 발생할 수 있으므로 프로젝트의 요구사항과 상황에 맞는 적절한 해결방안을 선택하고, 일관된 규칙을 적용하는 것이 중요합니다.