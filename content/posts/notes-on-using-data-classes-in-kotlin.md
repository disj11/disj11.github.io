---
title: "Kotlin에서 데이터 클래스를 사용할 때 주의할 점"
description: ""
date: 2023-01-08T14:39:00+09:00
tags: [kotlin]
---

Kotlin에서 데이터 클래스를 주의해야 할 사항이 있다.
데이터 클래스는 자동으로 copy() 함수를 생성하는데, 이 함수를 통해 속성값을 수정한 새로운 인스턴스를 생성할 수 있게 된다.
클래스의 속성 중에 불변성을 유지해야 하는 속성이 있다면, 데이터 클래스의 사용은 위험할 수 있다.

예를 들어, 다음과 같은 Point 클래스가 있다:

```kotlin
data class Point private constructor(
    val value: Int,
) {
    companion object {
        fun of(value: Int) = if (value < 0) {
            throw IllegalArgumentException("The value argument should always be set to a positive value, but the current value is $value")
        } else {
            Point(value)
        }
    }
}
```

코드에서 보는 바와 같이 `Point` 클래스의 `value` 속성은 양수여야 한다. 그러나 copy() 함수를 사용하면 음수값을 설정할 수 있다:

```kotlin
val point = Point.of(10)

// The copy() function allows us to set an invalid value.
val invalidPoint = point.copy(value = -10)
```

이를 방지하기 위해 데이터 클래스 대신 다음과 같은 `Point` 클래스를 정의할 수 있다:

```kotlin
class Point private constructor(
    val value: Int,
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false
        other as Point

        if (value != other.value) return false
        return true
    }

    override fun hashCode(): Int {
        return value
    }

    override fun toString(): String {
        return "Point(value=$value)"
    }
}
```

이 포스트에서 알아본 것 처럼, Kotlin에서 데이터 클래스를 사용할 때 `copy()` 함수가 자동으로 생성된다는 점을 조심해야한다.
데이터 클래스에 대해 더 많은 정보를 원한다면 [공식 문서](https://kotlinlang.org/docs/data-classes.html)를 참조하다.
