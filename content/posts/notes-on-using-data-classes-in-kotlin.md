---
title: "Kotlin 데이터 클래스의 주의사항"
description: "Kotlin 데이터 클래스에서 copy() 함수의 위험성을 이해하고, 안전하게 설계하는 방법을 알아봅니다. 불변성을 유지해야 하는 경우 적합한 대안을 제시합니다."
date: 2023-01-08T14:39:00+09:00
lastmod: 2024-12-30T12:47:00+09:00
tags: [kotlin]
---

## Kotlin 데이터 클래스에서 주의해야 할 사항: `copy()` 함수와 불변성

Kotlin의 데이터 클래스는 간결하고 편리한 기능을 제공하지만, 설계에 따라 불변성을 위반할 가능성이 있습니다. 특히 자동 생성되는 `copy()` 함수는 데이터 무결성을 위협할 수 있으므로 이를 방지하기 위한 대책이 필요합니다. 이 글에서는 `copy()` 함수로 인한 문제와 이를 해결하기 위한 두 가지 방법(일반 클래스 사용 및 init 블록 활용)을 소개합니다.

### 데이터 클래스와 `copy()` 함수의 문제점

데이터 클래스는 `equals()`, `hashCode()`, `toString()` 메서드와 함께 속성 값을 변경한 새 인스턴스를 생성할 수 있는 `copy()` 함수를 자동으로 제공합니다. 그러나 특정 조건을 만족해야 하는 속성이 있는 경우, `copy()` 함수는 이러한 조건 검증을 우회하여 불변성을 위반할 수 있습니다.

#### 예시: 불변성 위반

아래는 특정 조건(양수 값)을 만족해야 하는 `Point` 클래스를 정의한 코드입니다:

```kotlin
data class Point private constructor(
    val value: Int,
) {
    companion object {
        fun of(value: Int): Point {
            require(value >= 0) { 
                "The value argument should always be set to a positive value, but the current value is $value" 
            }
            return Point(value)
        }
    }
}
```

위 코드에서는 정적 팩토리 메서드(`of`)를 통해 음수 값이 들어오는 것을 방지하고 있습니다. 그러나 `copy()` 함수를 사용하면 아래와 같이 잘못된 객체를 생성할 수 있습니다:

```kotlin
val point = Point.of(10)

// copy()를 사용하여 음수 값을 설정
val invalidPoint = point.copy(value = -10) // 잘못된 값이 허용됨
```

### 해결 방법 1: 일반 클래스를 사용

데이터 클래스 대신 일반 클래스를 사용하여 직접 필요한 메서드를 구현하면, `copy()` 함수가 제공되지 않으므로 불변성을 유지할 수 있습니다.

#### 일반 클래스 구현 예시

```kotlin
class Point private constructor(
    val value: Int,
) {
    companion object {
        fun of(value: Int): Point {
            require(value >= 0) { 
                "The value argument should always be set to a positive value, but the current value is $value" 
            }
            return Point(value)
        }
    }

    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false
        other as Point

        return value == other.value
    }

    override fun hashCode(): Int = value

    override fun toString(): String = "Point(value=$value)"
}
```

이 방식은 안전하지만, 데이터 클래스의 간결함과 편리함을 포기해야 한다는 단점이 있습니다.

### 해결 방법 2: init 블록 활용

데이터 클래스를 유지하면서도 init 블록을 활용하면 객체 생성 시점에 조건 검증 로직을 추가하여 불변성을 보장할 수 있습니다.

#### init 블록 적용 예시

```kotlin
data class Point(val value: Int) {
    init {
        require(value >= 0) { 
            "The value argument should always be set to a positive value, but the current value is $value" 
        }
    }
}
```

위 코드에서는 객체가 생성될 때마다 init 블록이 실행되어 속성 값에 대한 검증이 이루어집니다. 따라서 `copy()` 함수를 사용하더라도 잘못된 값을 설정하는 것을 방지할 수 있습니다:

```kotlin
val point = Point(10)

// copy()를 사용해도 init 블록에서 검증됨
val invalidPoint = point.copy(value = -10) // IllegalArgumentException 발생
```

#### 장점과 단점

- **장점**: 데이터 클래스의 간결함과 기능(`equals()`, `hashCode()`, `toString()`, `copy()`)을 그대로 사용할 수 있습니다.
- **단점**: 조건 검증 로직이 추가될 경우 코드가 복잡해질 수 있습니다.

### 요약 및 결론

Kotlin 데이터 클래스에서 자동 생성되는 `copy()` 함수는 편리하지만, 설계 의도에 따라 불변성을 위반하는 객체를 생성할 위험이 있습니다. 이를 방지하기 위해 다음 두 가지 방법 중 하나를 선택할 수 있습니다:

1. **일반 클래스를 사용**하여 직접 메서드를 구현.
2. **init 블록을 활용**하여 객체 생성 시 조건 검증 로직 추가.

두 방법 모두 상황에 따라 적합하게 선택해야 하며, 설계 의도와 요구 사항에 따라 결정하는 것이 중요합니다.

---
참고자료:
- https://kotlinlang.org/docs/data-classes.html