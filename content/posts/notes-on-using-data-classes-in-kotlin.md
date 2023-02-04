---
title: "Notes on Using Data Classes in Kotlin"
description: "Take note of the copy() function When using data classes in Kotlin"
date: 2023-01-08T14:39:00+09:00
tags: [kotlin]
---

When using data classes in Kotlin, it is important to keep a few things in mind. Data classes automatically create a `copy()` function, which can be used to create a new instance of the class with modified property values. However, if a property of the class must maintain an invariant, this function may not behave as expected.   
For example, consider the following `Point` class:

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

The `value` property of the `Point` class is intended to be positive. However, the `copy()` function allows us to set an invalid value:

```kotlin
val point = Point.of(10)

// The copy() function allows us to set an invalid value.
val invalidPoint = point.copy(value = -10)
```

To prevent this from happening, we can define the `Point` class as follows:

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

As we have learned in this post, it is important to always consider the `copy()` function when using data classes in Kotlin.

If you have any questions about the data classes, you can find more information [here](https://kotlinlang.org/docs/data-classes.html).
