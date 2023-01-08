---
title: "Nullable Receiver in Extension Functions of Kotlin"
date: 2023-01-08T21:22:27+09:00
url: "/nullable-receiver-in-extension-functions-of-kotlin/"
tags: [kotlin]
---

The receiver object in extension functions allows null values because it is actually a parameter. At first glance, `anObject.method()` and `anObject.extensionFunction()` may look similar, but this is not the case. If `anObject` is null, the `method()` will never be called, but the `extensionFunction()` can still be called. Here is an example:
```kotlin
fun String?.extensionFunction() = this?.length

fun main() {
    val str: String? = null
    println(str.extensionFunction()) // prints 'null'
}
```
In this example, when using `this` inside an extension function, note that the safe call operator should be used.   
This code has a disadvantage: it may return `null`. In these cases, we should write code like the following:
```kotlin
val str: String = "Some text"
val length = str.extensionFunction() ?: error("Should never happen")

```
If the surrounding code changes, it could cause a bug to occur.
To avoid these problems, if you use a nullable receiver inside an extension function, it is best not to return `null`.
If you need an extension function that returns `null`, you can define it as an extension of a non-nullable type and use the safe call operator when calling the extension, as shown in the following code:
```kotlin
fun String.extensionFunction(): Int? {
    return // ...
}

val length = str?.extensionFunction()

```
If you have any questions about the extensions, you can find more information [here](https://kotlinlang.org/docs/extensions.html).
