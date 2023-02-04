---
title: "Kotlin: Memory Efficient Iterable Data Processing with Sequences"
description: "Let's find out how do they differ in behavior between Iterable and Sequence in Kotlin."
date: 2023-01-10T23:59:43+09:00
tags: [kotlin]
---

Kotlin provides extension functions for the Iterable interface, such as the `filter` function. Here is an example of using the `filter` function:

```kotlin
fun averageNonBlankLength(strings: List<String>): Double =
    strings
        .filter { it.isNotBlank() }
        .map(String::length)
        .sum() / strings.size.toDouble()
```

It's worth noting that the `filter` and `map` functions in Kotlin return new `List`s, unlike their counterparts `Stream.filter` and `Stream.map` in Java, which return a `Stream`. In this example, two new intermediate lists are created in memory as the result of calling `filter`  and `map`  on the original list. However, this overhead may not be significant depending on the size of the original list and the complexity of the operations.

If the size of the original list or complexity of the operation chain is high and memory usage is a concern, you might consider using `sequence`  type instread of `List` . you can convert a list to a sequence via `asSequence()`  function which will create a lazy sequence and perform operations on it. Another option is to use the `sequenceOf(list)`  function which will create a sequence from the list.   
Here's an example of using the `sequence`  type to perform the same operation as the previous example:

```kotlin
fun averageNonBlankLength(strings: List<String>): Double =
    strings.asSequence()
        .filter { it.isNotBlank() }
        .map(String::length)
        .sum() / strings.size.toDouble()
```

In this example, the `asSequence()`  function is used to convert the original `List`  of strings to a `Sequence`  of strings. Then, the `filter`  and `map`  functions are used to perform the same operations as before, but instead of creating intermediate `List`s, a `Sequence`  is returned from each function call.   
It's worth noting that using a sequence allows for lazy evaluation, meaning that the elements of the sequence are only evaluated as they are needed, this can help to reduce the memory usage. Using a sequence can be beneficial when memory efficiency is a concern, particularly in
situations where the size of the result set of data is not known ahead of time or when the operations are complex and intermediate results are unnecessary to be stored in memory.

This is how the list processing goes:   
![list-processing.png](https://kotlinlang.org/docs/images/list-processing.png)

The sequence processing goes like this:   
![sequence-processing.png](https://kotlinlang.org/docs/images/sequence-processing.png)

If you would like to learn more about the `sequence` type in Kotlin, you can refer to the [official documentation](https://kotlinlang.org/docs/sequences.html)   
   
