---
title: "Hibernate Proxy 비정상 동작 사례 분석"
description: "Kotlin 엔티티에서 동일 이름의 Getter 메서드로 인해 발생한 Hibernate Proxy의 비정상 동작 사례를 분석하고, 문제의 원인과 해결 방안을 제시합니다."
date: 2024-12-29T21:36:18+09:00
url: "/inconsistent-hibernate-proxy-behavior-with-kotlin/"
tags: [TIL]
---

## 문제 상황

서비스 운영 중 `NullPointerException`이 발생한 사례를 분석하였습니다. 특정 필드가 `null`이 아닌 상태로 로그에 출력되었음에도 불구하고, 이후 코드 실행 중 `NullPointerException`이 발생하는 이상 현상이 발견되었습니다. 문제는 동일한 코드를 다시 배포할 때마다 오류가 발생하거나 그렇지 않은 경우가 번갈아 나타나는 불안정한 상황이었습니다.

### 코드 및 로그 분석

다음은 문제의 코드와 로그입니다:

```kotlin
fun doSomething() {
    val adGroup = findById(adGroupId)
    log.info { "광고그룹 정보: ${adGroup}" } // 광고그룹 정보: AdGroup(adProductId=3) 출력
    requireNotNull(adGroup.adProductId) // NullPointerException 발생
}
```

위 로그에서는 `adProductId`가 `3`으로 출력되었으나, `requireNotNull(adGroup.adProductId)`에서 `NullPointerException`이 발생했습니다. 로그를 추가하여 확인한 결과 다음과 같은 현상이 나타났습니다:

```kotlin
log.info { "adProductId: ${adGroup.adProductId}" }
```

이번에는 로그에 `null`이 찍혔습니다. 이 시점에서 문제의 원인이 엔티티 클래스에 있다고 생각하였고, 해당 엔티티 클래스에 아래와 같은 메서드가 정의되어 있음을 확인했습니다:

```kotlin
@Entity
class AdGroup(
    @Id
    ...

    @Column
    var adProductId: Long? = null
) {
    fun getAdProductId(): Long = requireNotNull(this.adProductId)
}
```

로그를 다시 변경하여 테스트한 결과는 다음과 같습니다:

```kotlin
println("${adGroup.adProductId} || ${adGroup.getAdProductId()}")
```

빌드 후 실행할 때마다 아래 두 가지 결과가 번갈아 출력되었습니다:
- `null || 3`
- `3 || 3`

### 원인 분석

문제는 Kotlin과 Java 간의 언어적 차이와 Hibernate의 리플렉션 기반 구현 방식에서 비롯된 것으로 보입니다. 특히, 동일한 이름을 가진 여러 Getter 메서드가 존재할 경우 Hibernate Proxy가 예기치 못한 방식으로 동작할 수 있다는 점이 확인되었습니다.

1. **Kotlin과 Java의 차이**  
   Java에서는 동일한 시그니처(메서드 이름과 매개변수 타입)를 가진 메서드를 허용하지 않지만, Kotlin에서는 반환 타입만 다른 메서드를 정의할 수 있습니다. 이는 Kotlin에서 허용되지만, Hibernate Proxy와 같은 리플렉션 기반 라이브러리에서는 혼란을 초래할 수 있습니다.

2. **Hibernate Proxy 동작 방식**  
   Hibernate Proxy는 리플렉션 데이터를 기반으로 메서드를 호출합니다. 이 과정에서 리플렉션 데이터의 메서드 배열(`publicMethods`) 순서에 따라 호출되는 Getter 메서드가 달라질 수 있습니다. 디버깅 결과, 아래와 같은 동작이 확인되었습니다:
    - 실패 시: `public java.lang.Long org.xx.xx.getAdProductId()` 호출 → `null`
    - 성공 시: `public long org.xx.xx.getAdProductId()` 호출 → `3`

팀원 분이 [Hibernate 포럼에 문의](https://discourse.hibernate.org/t/title-inconsistent-proxy-behavior-with-kotlin-property-access-and-custom-method/10746)한 결과, 다음과 같은 답변을 받았습니다:

> 이 문제는 이미 Kotlin 측에서 발생할 수 있는 예외적인 사례로 보이며, Hibernate 엔티티에서 여러 Getter 메서드를 가지는 속성을 사용하는 것은 권장하지 않습니다. 간단한 Java 애플리케이션에서 동일한 문제가 발생하는 사례를 주시면 도움을 드릴 수 있습니다. 하지만 JPA 명세에서 엔티티의 속성 접근자 메서드는 [Java Beans 규약](https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1.html#persistent-fields-and-properties)을 따라야 한다는 점을 유념하시기 바랍니다. Hibernate는 이러한 요구 사항에 대해 다소 관대하게 동작하려고 하지만, 여전히 이를 준수하는 것을 [강력히 권장](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-pojo-accessors)합니다.

원문:

> this already looks like an edge-case on the Kotlin side, and I would certainly discourage having properties with multiple getter methods on Hibernate entities in the first place. We can help you if you can demonstrate a problem that occurs with a simple Java application, but keep in mind that property accessor methods for entities in the JPA specification must follow the [Java Beans conventions](https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1.html#persistent-fields-and-properties), and while Hibernate tries to be more lax about this requirement it’s still [heavily encouraged](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-pojo-accessors).

### 해결 방안

1. **커스텀 Getter 메서드 이름 변경**  
   동일 이름의 Getter 메서드 충돌을 방지하기 위해 커스텀 메서드 이름을 변경합니다.
   ```kotlin
   fun fetchAdProductId(): Long = requireNotNull(this.adProcutId)
   ```

2. **@Transient 애노테이션 추가**  
   `@Transient` 애노테이션을 추가합니다.
   ```kotlin
   @Transient
   fun getAdProductId(): Long = requireNotNull(this.adProcutId)
   ```

3. **Java Beans 규약 준수**  
   JPA 엔티티 설계 시 Java Beans 규약을 준수하며, 동일 이름의 Getter 메서드를 피해야 합니다.

### 결론

이번 사례는 Kotlin과 Hibernate 간의 호환성 문제로 인해 발생한 것으로, 동일 이름의 Getter 메서드 사용이 원인이었습니다. 이를 통해 JPA 엔티티 설계 시 Java Beans 규약을 준수하는 것이 중요함을 다시 한번 확인할 수 있었습니다.
