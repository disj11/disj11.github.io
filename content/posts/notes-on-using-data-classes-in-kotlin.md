---
title: "Notes on Using Data Classes in Kotlin"
date: 2023-01-08T14:39:00+09:00
url: "/notes-on-using-data-classes-in-kotlin/"
tags: [kotlin]
---

When we use data classes in Kotlin, there are a few things to keep in mind.   
Kotlin data classes automatically create a `copy()`  function. If a property of a class must maintain an invariant, this function may not behave as expected. For example, we have the following code.
```
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
Is the `value` property of the `Point` class instance always positive? The answer is "No".   
We can set a negative value using the `copy()` function.
```
val point = Point.of(10)

// The copy() function allows us to set an invalid value.
val invalidPoint = point.copy(value = -10)

```
To avoid these problems, we should write code like the following
```
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
As we learned in this post, it is important to always consider the `copy()` function when using data classes in Kotlin.

If you are wondering about the `copy()` function, you can find a reference [here](https://kotlinlang.org/docs/data-classes.html).   
