---
title: "JIT Compiler: JVM 성능 최적화의 핵심 기술"
description: "JIT Compiler는 자바 코드 실행 속도를 획기적으로 향상시키는 JVM의 핵심 기술입니다. 이 글에서는 JIT Compiler의 작동 원리, Tiered Compilation, 임계값 설정 방법, 그리고 실제 성능 테스트를 통해 JIT의 효과를 자세히 살펴봅니다."
date: 2023-05-10T23:56:05+09:00
lastmod: 2024-12-29T19:45:00+09:00
url: "/java-jit-compiler/"
tags: [development]
---

## JIT Compiler란?

자바 코드를 실행하기 위해서는 바이트 코드로 컴파일이 필요합니다. 바이트 코드는 JVM의 인터프리터를 통해 기계어로 해석된 후 실행됩니다. 하지만 인터프리터를 통해 해석되는 과정 때문에, 기계어로 직접 실행되는 언어에 비해 속도가 느릴 수 있습니다. 이러한 성능 차이를 해결하기 위해 JVM에서는 **JIT(Just-In-Time) Compiler**를 도입하였습니다.

## JIT Compiler의 작동 방식

Oracle은 JDK 1.3부터 **HotSpot**이라는 가상 머신을 포함하고 있으며, 여기에는 두 가지 JIT 컴파일러가 포함되어 있습니다.

- **C1(Client Compiler)**: 빠르게 실행되지만 최적화 수준이 낮은 코드를 생성합니다.
- **C2(Server Compiler)**: 실행 시간이 더 오래 걸리지만, 더 최적화된 코드를 생성합니다.

### Tiered Compilation

JVM은 호출되는 메서드를 추적하며, 자주 호출되는 메서드를 C1 컴파일러를 사용해 컴파일합니다. 이후 호출 횟수가 증가하면 C2 컴파일러를 사용하여 다시 컴파일합니다. 이를 **Tiered Compilation**이라고 하며, 세부적인 컴파일 레벨은 다음과 같습니다:

- Level 0 – Interpreted Code
- Level 1 – Simple C1 Compiled Code
- Level 2 – Limited C1 Compiled Code
- Level 3 – Full C1 Compiled Code
- Level 4 – C2 Compiled Code

각 레벨에 대해 더 자세한 내용을 확인하고 싶다면 [Compilation Levels](https://www.baeldung.com/jvm-tiered-compilation#compilation-levels)을 참고합니다.

### 임계값 확인 및 설정

JVM에서 메서드가 특정 레벨로 컴파일되기 위한 임계값은 다음 명령어로 확인할 수 있습니다:

```shell
java -XX:+PrintFlagsFinal -version | grep Threshold | grep Tier
```

출력 예시:
```shell
intx Tier3CompileThreshold = 2000 {product} {default}
intx Tier3InvocationThreshold = 200 {product} {default}
intx Tier3MinInvocationThreshold = 100 {product} {default}
```

여기서 Tier3 는 Level 3로 컴파일 되기 위한 임계값임을 나타내며 각 값의 의미는 아래와 같습니다:

- **Tier3InvocationThreshold**: 메서드 호출 횟수 임계값.
- **Tier3BackEdgeThreshold**: 반복문 등에서 이전 블록으로 돌아가는 분기 구문의 임계값.
- **Tier3CompileThreshold**: 메서드 호출 횟수와 백 엣지(back-edge) 임계값의 합.

[tiered compilation and CompileThreshold](https://mail.openjdk.org/pipermail/hotspot-compiler-dev/2010-November/004239.html) 에 메서드를 컴파일해야 하는지를 판단하기 위한 로직이 설명되어있습니다. 이를 간단히 코드로 표현해보면 다음과 같습니다:

```kotlin
function shouldCompileMethod(invocationCount, backEdgeCount) {
    if (invocationCount > Tier3InvocationThreshold) {
        return true;
    }

    if (invocationCount > Tier3MinInvocationThreshold && invocationCount + backEdgeCount > Tier3CompileThreshold ) {
        return true;
    }

    return false;
}
```

이를 바탕으로 메서드가 호출될 때마다 20개의 back-edge가 생성된다고 가정하면, 이 메서드는 약 100번 호출될 때 Level 3로 컴파일될 수 있음을 파악할 수 있습니다.:

$$
100 + (20 \times 100) > 2000
$$

```
100 + (20 * 100) > 2000
---   ----------   ----
1     2            3
```

- 1: 메서드 호출 횟수
- 2: back-edge
- 3: Tier3CompileThreshold

### 최적화 확인 방법

JIT Compiler의 최적화를 확인하려면 아래 VM 옵션을 추가하여 로그를 생성할 수 있습니다:

```shell
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation
```

옵션을 추가하고 아래의 샘플 코드를 통해 실제로 최적화가 일어나는지 확인해보았습니다:

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    (1..3000).forEach { i ->
        findMax(arr)
        if (i % 100 == 0) {
            // 너무 빨리 종료되면 c2 컴파일이 안될 수도 있기 때문에 sleep 을 넣어줌
            Thread.sleep(100)
        }
    }
}

fun findMax(arr: IntArray): Int {
    var max = arr[0]
    for (i in 1 until arr.size) {
        if (max < arr[i]) {
            max = arr[i]
        }
    }
    return max
}
```

프로그램 실행 후 생성된 로그 파일(`hotspot_pid<pid>.log`)에서 다음과 같이 level 3 컴파일을 위해 c1 queue 에 메서드가 적재된 것을 확인할 수 있었습니다:

```xml
<task_queued compile_id='205' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='228' backedge_count='2048' iicount='228' level='3' stamp='0.438' comment='tiered' hot_count='228'/>
```

count 와 backedge_count 를 통해 메서드가 몇 번 호출되었는지와 백엣지 수를 확인할 수 있으며, 조금 더 아래 로그를 살펴보니 다음과 같이 level 3 로 코드 최적화가 된 것을 확인할 수 있었습니다:

```xml
<nmethod compile_id='205' compiler='c1' level='3' entry='0x00000208cb1883a0' size='2560' address='0x00000208cb188190' relocation_offset='344' insts_offset='528' stub_offset='1872' scopes_data_offset='2040' scopes_pcs_offset='2216' dependencies_offset='2520' nul_chk_table_offset='2528' oops_offset='1992' metadata_offset='2008' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='228' backedge_count='2048' iicount='228' stamp='0.438'/>
```

### JIT Compiler 활성화/비활성화 성능 비교

아래 코드는 JIT Compiler 활성화 여부에 따라 실행 시간을 비교한 결과입니다. `-Xint` 옵션으로 Jit Compiler 사용을 중지하고 테스트 해 볼 수 있습니다:

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val startTimeMills = System.currentTimeMillis()
    (1..1_0000_0000).forEach { findMax(arr) }
    println("실행시간: ${System.currentTimeMillis() - startTimeMills}ms")
}

fun findMax(arr: IntArray): Int {
    var max = arr[0]
    for (i in arr.indices) {
        if (max < arr[i]) max = arr[i]
    }
    return max
}
```

#### 결과

- **JIT Compiler 활성화**
    - 실행시간: 약 875~1595ms
- **JIT Compiler 비활성화 (`-Xint` 옵션 사용)**
    - 실행시간: 약 24,760~32,401ms

이처럼 JIT Compiler는 프로그램의 성능을 크게 향상시킵니다.

**주의사항**:
Spring Boot 애플리케이션을 IntelliJ에서 실행할 경우 `-XX:TieredStopAtLevel=1` 옵션이 자동 추가되어 Level 1까지만 컴파일됩니다([IDEA-297872](https://youtrack.jetbrains.com/issue/IDEA-297872)). 정확한 테스트를 위해서는 Run/Degub Configurations -> Modify options -> Disabled launch Optimization 옵션을 체크하여 실행해야 합니다.

참고 자료:
- https://www.baeldung.com/jvm-tiered-compilation
- https://www.baeldung.com/graal-java-jit-compiler
- https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/
- https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html
- https://hg.openjdk.org/jdk8/jdk8/hotspot/file/104743074675/src/share/vm/runtime/advancedThresholdPolicy.hpp
- https://www.youtube.com/watch?v=CQi3SS2YspY&list=PLyGtIjZ_uWKNT5-ob1TL26hH0KVfoJQzw
