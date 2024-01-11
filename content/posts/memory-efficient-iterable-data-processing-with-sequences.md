---
title: "Kotlin: 시퀀스를 활용한 메모리 효율적인 데이터 처리 방법"
description: "Kotlin에서 Iterable과 Sequence 간의 동작이 어떻게 다른지 알아보자."
date: 2023-01-10T23:59:43+09:00
lastmod: 2024-01-11T18:00:43+09:00
tags: [kotlin]
---

Kotlin은 filter 함수와 같은 Iterable 인터페이스를 위한 확장 함수를 제공한다. filter 함수를 사용하는 예제는 다음과 같다:

```kotlin
fun averageNonBlankLength(strings: List<String>): Double =
    strings
        .filter { it.isNotBlank() }
        .map(String::length)
        .sum() / strings.size.toDouble()
```

한 가지 주목할 점은 Kotlin의 `filter` 및 `map` 함수는 `Stream`을 반환하는 Java의 `Stream.filter` 및 `Stream.map`과 달리 새로운 `List`를 반환한다는 점이다.
이 예제에서는 원본 리스트에서 `filter` 및 `map`을 호출한 결과로 메모리에 두 개의 새로운 리스트가 생성된다.
그러나 원본 리스트의 크기와 작업의 복잡성에 따라 이 오버헤드는 중요하지 않을 수 있다.

원본 리스트의 크기가 크거나 작업 체인의 복잡성이 커서 메모리 사용량이 우려되는 경우, `List` 대신 `Sequence` 타입을 사용할 수 있다.
`asSequence()` 함수를 사용하여 리스트를 시퀀스로 변환하거나 `sequenceOf(list)` 함수를 사용하여 리스트에서 시퀀스를 생성할 수 있다.
아래 예제는 이전 예제와 동일한 작업을 수행하는 데 sequence 유형을 사용하는 방법을 보여준다:

```kotlin
fun averageNonBlankLength(strings: List<String>): Double =
    strings.asSequence()
        .filter { it.isNotBlank() }
        .map(String::length)
        .sum() / strings.size.toDouble()
```

이 예제에서는 `asSequence()` 함수를 사용하여 `List`를 `Sequence`로 변환한다.
그런 다음 `filter` 및 `map` 함수를 사용하여 이전과 동일한 작업을 수행하지만 중간 `List`를 생성하는 대신 각 함수 호출에서 `Sequence`가 반환된다.

시퀀스를 사용하면 lazy evaluation로 동작하므로 필요없는 계산을 하지 않는다. 이는 메모리 사용량을 줄이는 데 도움이 될 수 있다.
[공식 문서](https://kotlinlang.org/docs/sequences.html)에 나오는 예제를 보자.

Iterable:

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")
val lengthsList = words.filter { println("filter: $it"); it.length > 3 }
.map { println("length: ${it.length}"); it.length }
.take(4)

println("Lengths of first 4 words longer than 3 chars:")
println(lengthsList)
```

```bash
filter: The
filter: quick
filter: brown
filter: fox
filter: jumps
filter: over
filter: the
filter: lazy
filter: dog
length: 5
length: 5
length: 5
length: 4
length: 4
Lengths of first 4 words longer than 3 chars:
[5, 5, 5, 4]
```

![list-processing.png](https://kotlinlang.org/docs/images/list-processing.png)

Sequence:

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")
//convert the List to a Sequence
val wordsSequence = words.asSequence()

val lengthsSequence = wordsSequence.filter { println("filter: $it"); it.length > 3 }
.map { println("length: ${it.length}"); it.length }
.take(4)

println("Lengths of first 4 words longer than 3 chars")
// terminal operation: obtaining the result as a List
println(lengthsSequence.toList())
```

```bash
Lengths of first 4 words longer than 3 chars
filter: The
filter: quick
length: 5
filter: brown
length: 5
filter: fox
filter: jumps
length: 5
filter: over
length: 4
[5, 5, 5, 4]
```

![sequence-processing.png](https://kotlinlang.org/docs/images/sequence-processing.png)

시퀀스는 결과 데이터 집합의 크기를 미리 알 수 없는 상황이나, 작업이 복잡하고 중간 결과를 메모리에 저장할 필요가 없는 상황 등 메모리 효율성이 중요한 경우에 유용하다.
