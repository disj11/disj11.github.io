---
title: "효율적인 Kotlin 컬렉션 처리: Iterable과 Sequence의 차이점과 활용법"
description: "Kotlin의 Iterable과 Sequence는 컬렉션을 처리하는 두 가지 주요 방식으로, 즉시 평가와 지연 평가라는 서로 다른 전략을 사용합니다. 이 글에서는 두 방식의 차이점, 성능 비교, 적합한 사용 사례를 중심으로 효율적인 Kotlin 코드를 작성하는 방법을 알아봅니다."
date: 2023-01-10T23:59:43+09:00
lastmod: 2024-12-29T09:30:43+09:00
tags: [kotlin]
---

Kotlin의 `Iterable`과 `Sequence`는 컬렉션을 처리하는 두 가지 주요 방식으로, 각기 다른 평가 전략과 성능 특성을 가집니다. 이 글에서는 두 방식의 차이점을 중심으로, 각각의 장단점과 적합한 사용 사례를 설명합니다.

## **`Iterable` vs `Sequence`: 평가 방식의 차이**

### **1. 즉시 평가 (Eager Evaluation) - Iterable**
`Iterable`은 각 처리 단계에서 전체 컬렉션에 대해 연산을 수행하며, 중간 결과를 새로운 컬렉션으로 반환합니다. 이 방식은 각 단계가 독립적으로 실행되므로 직관적이고 간단하지만, 중간 결과로 인해 메모리 사용량이 증가할 수 있습니다.

#### **작동 방식**
- 각 연산(`filter`, `map` 등)이 전체 컬렉션에 대해 실행됩니다.
- 중간 결과로 새로운 리스트가 생성됩니다.
- 모든 작업이 완료된 후 최종 결과를 얻습니다.

예를 들어, 다음 코드는 `Iterable` 방식으로 작동합니다:

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")
val lengthsList = words.filter { println("filter: $it"); it.length > 3 }
    .map { println("length: ${it.length}"); it.length }
    .take(4)

println("Lengths of first 4 words longer than 3 chars:")
println(lengthsList)
```

**출력 결과:**

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
Lengths of first 4 words longer than 3 chars:
[5, 5, 5, 4]
```

위 예제에서 볼 수 있듯이, `filter`와 `map` 연산은 모든 요소에 대해 수행되고, 중간 리스트가 생성됩니다.

---

### **2. 지연 평가 (Lazy Evaluation) - Sequence**
`Sequence`는 연산을 필요할 때까지 미루며, 각 요소에 대해 연산 체인을 순차적으로 적용합니다. 이는 중간 결과를 저장하지 않으므로 메모리 효율성이 높아지고, 불필요한 계산을 피할 수 있습니다.

#### **작동 방식**
- 각 요소가 처리 체인을 통과하며 필요한 만큼만 계산됩니다.
- 중간 리스트를 생성하지 않습니다.
- 최종 결과를 요청하는 시점(터미널 연산)에서 계산이 수행됩니다.

다음 코드는 동일한 작업을 `Sequence`로 처리하는 방법입니다:

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")
val lengthsSequence = words.asSequence()
    .filter { println("filter: $it"); it.length > 3 }
    .map { println("length: ${it.length}"); it.length }
    .take(4)

println("Lengths of first 4 words longer than 3 chars:")
println(lengthsSequence.toList())
```

**출력 결과:**

```bash
Lengths of first 4 words longer than 3 chars:
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

여기서 볼 수 있듯이, `Sequence`는 필요한 만큼만 데이터를 처리하며 불필요한 연산을 피합니다.

---

## **성능 비교**

### **1. 작은 데이터셋**
작은 크기의 컬렉션에서는 `Iterable`이 더 유리할 수 있습니다. 즉시 평가 방식은 CPU 캐시 활용성이 높아 간단한 작업에서는 더 빠르게 동작합니다.

### **2. 큰 데이터셋**
큰 데이터셋이나 복잡한 연산 체인에서는 `Sequence`가 더 적합합니다. 중간 결과를 저장하지 않고 필요한 만큼만 계산하므로 메모리 사용량과 성능 면에서 효율적입니다.

#### **예제 비교**
- **Iterable:** 모든 요소를 처리하고 중간 리스트를 생성.
- **Sequence:** 필요한 요소까지만 계산하고 중간 리스트 생성을 피함.

실제 벤치마크에 따르면, 큰 데이터셋에서 Sequence는 수백 배 더 빠른 성능을 보일 수 있습니다.

---

## **사용 시점**

| 상황                              | 추천 방식      |
|-----------------------------------|----------------|
| 단순하고 작은 데이터셋            | `Iterable`     |
| 복잡한 연산 체인 또는 큰 데이터셋 | `Sequence`     |
| 중간 결과 저장 필요               | `Iterable`     |
| 불필요한 계산 최소화 필요         | `Sequence`     |

---

## **결론**

Kotlin의 `Iterable`과 `Sequence`는 각각 다른 방식으로 컬렉션을 처리하며, 적절한 선택은 작업의 특성과 데이터 크기에 따라 달라집니다. 작은 데이터셋이나 간단한 작업에는 `Iterable`, 메모리 효율성과 성능 최적화가 중요한 경우에는 `Sequence`를 사용하는 것이 좋습니다. 이러한 차이를 이해하고 적절히 활용하면 더 효율적인 Kotlin 코드를 작성할 수 있습니다.

참고자료:

* https://kotlinlang.org/docs/sequences.html
* https://proandroiddev.com/sequences-x-iterable-in-kotlin-b5df65cad2d2?gi=59a6e33d99d9
* https://stackoverflow.com/questions/35629159/kotlins-iterable-and-sequence-look-exactly-same-why-are-two-types-required/35630670
* https://kt.academy/article/ek-sequence