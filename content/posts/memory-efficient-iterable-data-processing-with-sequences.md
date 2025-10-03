---
title: "Kotlin 컬렉션 처리: Iterable과 Sequence, 무엇을 언제 사용해야 할까?"
description: "Kotlin에서 컬렉션을 다루는 두 가지 방식, Iterable과 Sequence의 핵심적인 차이인 즉시 평가(Eager)와 지연 평가(Lazy)에 대해 알아봅니다. 각 방식의 동작 원리와 성능 특성을 비교하고, 언제 어떤 방식을 사용해야 더 효율적인 코드를 작성할 수 있는지 명확한 가이드를 제시합니다."
date: 2023-01-10T23:59:43+09:00
lastmod: 2024-12-29T09:30:43+09:00
tags: ["kotlin", "performance", "collections"]
---

## 들어가며: 컬렉션 처리의 두 가지 접근법

Kotlin에서 `List`나 `Set`과 같은 컬렉션을 다룰 때, 우리는 `filter`, `map`과 같은 다양한 고차 함수를 사용합니다. 이러한 함수들은 대부분 `Iterable` 인터페이스의 확장 함수로 구현되어 있습니다. 그런데 Kotlin 표준 라이브러리에는 `Iterable`과 매우 유사해 보이는 `Sequence`라는 또 다른 인터페이스가 존재합니다.

두 방식은 겉보기에는 비슷하지만, 내부적으로는 데이터를 처리하는 방식에 있어 **즉시 평가(Eager Evaluation)** 와 **지연 평가(Lazy Evaluation)** 라는 결정적인 차이가 있습니다. 이 차이를 이해하는 것은 대용량 데이터를 다루거나 복잡한 연산 체인을 구성할 때 코드의 성능을 최적화하는 데 매우 중요합니다. 이 글에서는 `Iterable`과 `Sequence`의 동작 원리를 비교하고, 각각 어떤 상황에 더 적합한지 알아보겠습니다.

## 1. 즉시 평가 (Eager Evaluation): `Iterable`의 방식

`List`, `Set` 등 우리가 일반적으로 사용하는 컬렉션은 `Iterable`을 구현합니다. `Iterable`에 대한 연산은 **즉시 평가** 방식으로 동작합니다. 이는 각 연산(`filter`, `map` 등)이 호출되는 즉시 전체 컬렉션을 순회하며, 그 결과를 담은 **새로운 중간 컬렉션** 을 생성하여 반환한다는 의미입니다.

### `Iterable`의 동작 과정

아래 코드를 통해 `Iterable`의 동작을 단계별로 살펴보겠습니다.

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")

println("--- Iterable 프로세스 시작 ---")
val lengthsList = words
    .filter { println("filter: $it"); it.length > 3 } // 첫 번째 연산
    .map { println("map: ${it.length}"); it.length }      // 두 번째 연산
    .take(4)                                            // 세 번째 연산

println("\n--- 최종 결과 ---")
println(lengthsList)
```

**출력 결과:**
```
--- Iterable 프로세스 시작 ---
filter: The
filter: quick
filter: brown
filter: fox
filter: jumps
filter: over
filter: the
filter: lazy
filter: dog
map: 5
map: 5
map: 5
map: 4
map: 4

--- 최종 결과 ---
[5, 5, 5, 4]
```

출력 결과를 분석해 보면 `Iterable`의 동작 방식을 명확히 알 수 있습니다.

1.  **`filter` 단계**: `words` 리스트의 **모든 요소(9개)** 에 대해 `filter` 연산을 수행합니다. 그 결과로 `[quick, brown, jumps, over, lazy]`라는 **새로운 리스트(중간 컬렉션 1)** 가 메모리에 생성됩니다.
2.  **`map` 단계**: `filter`에서 생성된 중간 컬렉션의 **모든 요소(5개)** 에 대해 `map` 연산을 수행합니다. 그 결과로 `[5, 5, 5, 4, 4]`라는 **또 다른 새 리스트(중간 컬렉션 2)** 가 생성됩니다.
3.  **`take` 단계**: `map`에서 생성된 리스트에서 앞의 4개 요소를 가져와 `[5, 5, 5, 4]`라는 **최종 리스트** 를 생성합니다.

이처럼 `Iterable`은 각 단계를 수평적으로(horizontally) 처리하며, 단계마다 새로운 컬렉션을 만들어 메모리 사용량이 증가하고, 불필요한 계산(예: `lazy`와 `dog`에 대한 `filter`, `lazy`의 길이를 계산한 `map`)이 발생할 수 있습니다.

## 2. 지연 평가 (Lazy Evaluation): `Sequence`의 방식

`Sequence`는 **지연 평가** 방식으로 동작합니다. `asSequence()`를 통해 컬렉션을 `Sequence`로 변환하면, `filter`나 `map`과 같은 **중간 연산(intermediate operations)** 은 즉시 실행되지 않습니다. 대신, 어떤 연산을 수행해야 하는지에 대한 계획만 세워둡니다.

실제 연산은 `toList()`, `count()`와 같은 **최종 연산(terminal operation)** 이 호출될 때 비로소 시작됩니다. 이때 `Sequence`는 각 요소를 하나씩 가져와 전체 연산 파이프라인을 통과시키는 방식으로 동작합니다.

### `Sequence`의 동작 과정

동일한 작업을 `Sequence`로 처리해 보겠습니다.

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")

println("--- Sequence 프로세스 시작 ---")
val lengthsSequence = words.asSequence() // Sequence로 변환
    .filter { println("filter: $it"); it.length > 3 } // 중간 연산 1
    .map { println("map: ${it.length}"); it.length }      // 중간 연산 2
    .take(4)                                            // 중간 연산 3

println("\n--- 최종 연산(toList) 호출 ---")
println(lengthsSequence.toList()) // 최종 연산
```

**출력 결과:**
```
--- Sequence 프로세스 시작 ---

--- 최종 연산(toList) 호출 ---
filter: The
filter: quick
map: 5
filter: brown
map: 5
filter: fox
filter: jumps
map: 5
filter: over
map: 4
[5, 5, 5, 4]
```

`Iterable`과 전혀 다른 출력 결과를 통해 `Sequence`의 동작 방식을 알 수 있습니다.

1.  **중간 연산 단계**: `asSequence()`, `filter`, `map`, `take`가 호출될 때는 아무런 계산도 일어나지 않습니다. `println` 구문이 실행되지 않은 것을 통해 이를 확인할 수 있습니다.
2.  **최종 연산 단계 (`toList()`)**: `toList()`가 호출되자 비로소 연산이 시작됩니다.
    -   첫 번째 요소 "The"가 `filter`를 통과하지 못합니다.
    -   두 번째 요소 "quick"이 `filter`를 통과하고, **즉시** `map`으로 전달되어 `5`가 됩니다. 이 `5`는 `take`로 전달됩니다.
    -   이 과정이 반복되어 "brown", "jumps", "over"가 차례로 파이프라인을 통과합니다.
    -   `take(4)`가 4개의 요소를 수집하는 순간, 더 이상 뒤의 요소를 처리할 필요가 없으므로 **연산을 중단** 합니다. 따라서 "lazy"와 "dog"는 `filter`조차 되지 않습니다.

이처럼 `Sequence`는 각 요소를 수직적으로(vertically) 처리하며, **중간 컬렉션을 생성하지 않고**, `take`와 같은 단축(short-circuiting) 연산 시 **불필요한 계산을 하지 않아** 매우 효율적입니다.

## `Iterable` vs `Sequence`: 언제 무엇을 써야 할까?

| 상황 | 추천 방식 | 이유 |
| :--- | :--- | :--- |
| **작은 컬렉션** 에 대한 **단순한 연산** | `Iterable` (기본) | `Sequence` 생성에 따른 약간의 오버헤드가 오히려 성능에 불리할 수 있습니다. 코드가 더 간단하고 직관적입니다. |
| **대용량 컬렉션** 또는 **무한한 데이터 스트림** | `Sequence` | 중간 컬렉션을 생성하지 않아 메모리 사용량이 매우 적고, OutOfMemoryError를 방지할 수 있습니다. |
| **복잡한 연산 체인** (여러 단계의 `map`, `filter` 등) | `Sequence` | 각 요소에 대해 전체 파이프라인을 실행하므로 중간 컬렉션 생성 비용이 없고, 전체 순회 횟수가 줄어듭니다. |
| **`find`, `any`, `take` 등 단축 연산** 을 포함할 때 | `Sequence` | 조건을 만족하는 즉시 연산을 중단하므로, 전체 컬렉션을 순회할 필요가 없어 성능이 크게 향상됩니다. |

## 결론

`Iterable`과 `Sequence`는 각각 즉시 평가와 지연 평가라는 뚜렷한 특징을 가집니다. 대부분의 간단한 컬렉션 처리에는 기본 방식인 `Iterable`로 충분하지만, 대용량 데이터를 다루거나 여러 단계의 복잡한 변환을 수행해야 할 때는 `asSequence()`를 사용하여 `Sequence`로 전환하는 것을 적극적으로 고려해야 합니다. 이 두 가지 방식의 차이점을 정확히 이해하고 상황에 맞게 사용한다면, 더 효율적이고 성능이 뛰어난 Kotlin 코드를 작성할 수 있을 것입니다.

---

**참고 자료:**
- [Kotlin 공식 문서 - Sequences](https://kotlinlang.org/docs/sequences.html)
- [ProAndroidDev - Sequences vs Iterable in Kotlin](https://proandroiddev.com/sequences-x-iterable-in-kotlin-b5df65cad2d2)