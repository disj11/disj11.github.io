---
title: "Kotlin과 Hibernate Proxy: 예측 불가능한 NullPointerException 분석"
description: "Kotlin 엔티티와 Hibernate 지연 로딩(Lazy Loading)을 함께 사용할 때, 간헐적으로 발생하는 NullPointerException의 원인을 심층 분석합니다. 프로퍼티와 동일한 이름의 커스텀 Getter가 Hibernate Proxy의 리플렉션 동작과 충돌하여 생기는 문제를 재현하고, 근본적인 해결 방안을 제시합니다."
date: 2024-12-29T21:36:18+09:00
tags: ["kotlin", "hibernate", "jpa", "bug"]
---

## 문제: 원인을 알 수 없는 NullPointerException

서비스 운영 중 간헐적으로 `NullPointerException`이 발생하는, 추적하기 매우 까다로운 문제를 마주했습니다. 문제의 현상은 다음과 같았습니다.

-   분명히 `null`이 아닌 값을 가진 것으로 로그에 기록된 객체의 특정 필드에 접근할 때 `NullPointerException`이 발생했습니다.
-   더욱 이상한 점은, 동일한 코드를 재배포할 때마다 오류가 발생했다가, 다음 배포에서는 발생하지 않는 등 **예측 불가능한 동작**을 보였습니다.

### 코드 및 로그 분석

문제의 현상은 아래 코드에서 발생했습니다. `adGroup` 객체는 Hibernate에 의해 지연 로딩(Lazy Loading)된 Proxy 객체였습니다.

```kotlin
fun doSomething() {
    // adGroupId를 통해 AdGroup 엔티티를 지연 로딩으로 조회
    val adGroup = findById(adGroupId)

    // (1) 로그 출력 시: adProductId=3 으로 정상 출력됨
    log.info { "광고그룹 정보: ${adGroup}" } // 출력: AdGroup(adProductId=3)

    // (2) 실제 필드 접근 시: adProductId가 null이라며 NullPointerException 또는 IllegalArgumentException 발생
    requireNotNull(adGroup.adProductId) 
}
```

`toString()` 메서드가 호출되어 로그가 찍힐 때는 `adProductId` 필드에 `3`이라는 값이 분명히 존재했습니다. 하지만 바로 다음 줄에서 해당 필드에 접근하자 `null`로 간주되어 예외가 발생했습니다. 원인을 파악하기 위해 로그를 추가하자 더욱 혼란스러운 현상이 나타났습니다.

```kotlin
// adProductId 필드에 직접 접근하여 로그 출력
log.info { "adProductId: ${adGroup.adProductId}" }
```

이번에는 로그에 `adProductId: null`이 출력되었습니다. 이 시점에서 문제의 원인이 `AdGroup` 엔티티 클래스의 구조에 있을 것이라 추측했고, 다음과 같은 커스텀 Getter 메서드를 발견했습니다.

```kotlin
@Entity
class AdGroup(
    @Id
    val id: Long,

    // (A) Nullable Long 타입의 프로퍼티
    @Column
    var adProductId: Long? = null
) {
    // (B) 동일한 이름을 가진 non-null Long 타입의 커스텀 Getter
    fun getAdProductId(): Long = requireNotNull(this.adProductId)
}
```

`adProductId` 프로퍼티와 이름이 완전히 동일한 `getAdProductId()` 메서드가 별도로 정의되어 있었습니다. 이 구조가 어떻게 예측 불가능한 오류를 일으키는지 확인하기 위해 마지막 테스트를 진행했습니다.

```kotlin
// 프로퍼티 접근과 Getter 메서드 호출을 동시에 테스트
println("${adGroup.adProductId} || ${adGroup.getAdProductId()}")
```

빌드 후 실행할 때마다 아래 두 가지 결과가 무작위로 번갈아 출력되었습니다.

-   **실패 케이스**: `null || 3`
-   **성공 케이스**: `3 || 3`

이 테스트를 통해, `adGroup.adProductId` (프로퍼티 접근)가 어떤 때는 `null`을, 어떤 때는 `3`을 반환하는 비정상적인 동작을 명확히 확인했습니다.

## 근본 원인 분석: Kotlin 프로퍼티와 Hibernate Proxy의 충돌

이 문제의 근본 원인은 **Kotlin의 프로퍼티 접근 방식**과 **리플렉션(Reflection)에 의존하는 Hibernate Proxy의 동작 방식**이 충돌했기 때문입니다.

1.  **Kotlin 프로퍼티와 중복된 Getter 생성**
    -   Kotlin에서 `var adProductId: Long?`라는 프로퍼티를 선언하면, 컴파일 시 자동으로 `public final Long getAdProductId()`라는 시그니처를 가진 Getter 메서드가 생성됩니다.
    -   그런데 개발자가 추가로 `fun getAdProductId(): Long`이라는 메서드를 정의했습니다. 이 메서드는 컴파일 후 `public final long getAdProductId()`라는 시그니처를 갖게 됩니다.
    -   결과적으로, 하나의 클래스 안에 이름은 같지만 반환 타입이 다른(`Long` vs `long`) 두 개의 `getAdProductId` 메서드가 공존하게 됩니다. (Java에서는 허용되지 않지만, JVM 레벨에서는 유효한 바이트코드입니다.)

2.  **Hibernate Proxy와 리플렉션의 불확실성**
    -   Hibernate는 지연 로딩을 위해 원본 엔티티를 상속받는 Proxy 클래스를 동적으로 생성합니다. 이 Proxy 객체의 메서드가 호출되면, Hibernate는 **리플렉션**을 사용하여 원본 객체의 실제 메서드를 찾아 호출합니다.
    -   문제는 `Class.getMethods()`와 같은 리플렉션 API가 반환하는 **메서드 배열의 순서를 보장하지 않는다**는 점입니다. JVM 구현이나 실행 시점의 미묘한 차이에 따라 메서드 목록의 순서가 달라질 수 있습니다.

3.  **충돌의 순간**
    -   `adGroup.adProductId` 코드가 실행되면, 내부적으로 Proxy 객체의 `getAdProductId()` 메서드가 호출됩니다.
    -   Hibernate Proxy는 리플렉션을 통해 `getAdProductId`라는 이름의 메서드를 찾습니다. 이때 JVM이 반환하는 메서드 목록의 순서에 따라 호출되는 실제 메서드가 달라집니다.
        -   **실패 시나리오**: 만약 프로퍼티의 Getter(`public Long getAdProductId()`)가 먼저 호출되면, 아직 초기화되지 않은 Proxy 객체의 필드 값을 그대로 반환하여 `null`이 됩니다.
        -   **성공 시나리오**: 만약 커스텀 Getter(`public long getAdProductId()`)가 먼저 호출되면, 이 메서드는 내부적으로 실제 엔티티의 필드에 접근하여 `3`을 반환합니다.

이러한 리플렉션 순서의 불확실성 때문에 동일한 코드임에도 불구하고 실행할 때마다 다른 결과가 나타났던 것입니다.

팀 동료가 [Hibernate 포럼에 문의](https://discourse.hibernate.org/t/title-inconsistent-proxy-behavior-with-kotlin-property-access-and-custom-method/10746)한 결과, Hibernate 팀에서도 이러한 방식의 사용을 권장하지 않는다는 답변을 받았습니다. JPA 명세는 엔티티의 속성 접근자가 **Java Beans 규약**을 따를 것을 요구하며, Hibernate는 이를 준수하는 것을 강력히 권장하고 있습니다.

## 해결 방안

이러한 문제를 해결하고 예측 가능한 코드를 작성하기 위한 방안은 다음과 같습니다.

1.  **커스텀 Getter 메서드 이름 변경 (가장 권장)**
    가장 확실하고 근본적인 해결책은 메서드 이름 충돌을 피하는 것입니다. 프로퍼티 이름과 다른, 의도가 명확히 드러나는 이름으로 변경합니다.
    ```kotlin
    // getAdProductId() -> fetchAdProductId() 로 이름 변경
    fun fetchAdProductId(): Long = requireNotNull(this.adProductId)
    ```

2.  **`@Transient` 애노테이션 사용**
    커스텀 Getter를 JPA가 관리하는 속성으로 인식하지 않도록 `@Transient`를 붙여주는 방법도 있습니다. 이를 통해 Hibernate가 해당 메서드를 영속성 컨텍스트에서 제외하도록 할 수 있습니다.
    ```kotlin
    @Transient
    fun getAdProductId(): Long = requireNotNull(this.adProductId)
    ```

3.  **Java Beans 규약 준수**
    JPA 엔티티를 설계할 때는 항상 Java Beans 규약을 염두에 두어야 합니다. 특히 프로퍼티와 동일한 이름의 Getter/Setter를 중복으로 정의하는 것을 피해야 합니다.

## 결론

이번 사례는 Kotlin의 편리한 기능이 리플렉션 기반의 프레임워크(Hibernate)와 만났을 때 발생할 수 있는 미묘하고 복잡한 문제를 잘 보여줍니다. 프레임워크의 내부 동작 원리에 대한 깊은 이해 없이 관례를 벗어난 코드를 작성하는 것이 얼마나 위험할 수 있는지를 깨닫게 된 경험이었습니다. JPA 엔티티를 작성할 때는 프레임워크가 기대하는 규약을 충실히 따르는 것이 예측 가능하고 안정적인 애플리케이션을 만드는 지름길임을 다시 한번 확인할 수 있었습니다.
