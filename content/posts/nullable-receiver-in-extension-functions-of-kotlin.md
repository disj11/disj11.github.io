---
title: "Kotlin: 확장 함수의 수신 객체"
description: "확장 함수의 수신 객체는 null 값이 될 수 있다."
date: 2023-01-08T21:22:27+09:00
tags: [kotlin]
---

확장 함수(extension functions)의 수신 객체는 매개변수이기 때문에 `null` 값이 허용된다.
`anObject.method()` 와 `anObject.extensionFunction()` 는 비슷해 보일 수 있지만 사실은 그렇지 않다.
`anObject`가 `null`인 경우, `method()` 는 호출될 수 없지만 `anObject.extensionFunction()`은 호출될 수 있다.
다음 예제를 살펴보자:

```kotlin
fun String?.extensionFunction() = this?.length

fun main() {
    val str: String? = null
    println(str.extensionFunction()) // prints 'null'
}
```

이 예제에서 확장 함수 내에서 `this` 를 사용할 때 타입 안전 연산자(safe call operator)를 사용하고 있다.
이 코드에는 단점이 하나 있는데, 바로 `null`을 반환할 수 있다는 것이다. 이런 경우 이 함수를 사용할 때 다음과 같은 예외 처리가 필요할 것이다:

```kotlin
val str: String = "Some text"
val length = str.extensionFunction() ?: error("Should never happen")
```

하지만 코드를 사용하는 곳에서 `extensionFunction()` 이 확장 함수라는 것을 인지하지 못한다면 예외 처리를 하지 못할 수도 있다.
이런 경우 버그 발생의 여지가 있다. 따라서 `null`이 가능한 수신 객체를 사용할 경우에는 `null`을 반환하지 않는 것이 좋다.
`null`을 반환하는 확장 함수가 필요한 경우, 해당 확장 함수를 non-nullable 유형의 확장으로 정의하고, 호출할 때 타입 안전 연산자를 사용하는 것이 좋다.
다음 코드를 참고하자:

```kotlin
fun String.extensionFunction(): Int? {
    return // ...
}

val length = str?.extensionFunction()
```

확장 함수에 대해 궁금한 점이 있다면 [공식 문서](https://kotlinlang.org/docs/extensions.html) 더 많은 정보를 확인할 수 있다.
