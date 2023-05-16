---
title: "Java JIT Compiler"
description: ""
date: 2023-05-10T23:56:05+09:00
url: "/java-jit-compiler/"
tags: [development]
---

## JIT Compiler 란?

자바 코드의 실행을 위해서는 바이트 코드로 컴파일이 필요하다. 바이트 코드는 다시 JVM 의 인터프리터를 통해 기계어로 해석되는 과정을 거쳐 실행된다.
이런 이유로 인터프리터를 통해 해석되는 과정없이 실행되는 언어에 비해 많이 느리다. 이러한 성능 차이를 해결하기 위해 JVM 에서는 JIT (Just In Time) Compiler 를 도입하였다.

## JIT Compiler 에 대한 자세한 내용

Oracle 에서는 JDK 1.3 부터 HotSpot 이라는 가상 머신을 포함한다. 여기에는 C1 이라고 하는 클라이언트 컴파일러와 C2 라고 하는 서버 컴파일러 두 개의 JIT 컴파일러가 포함되어 있다.
C1은 더 빠르게 실행되고 조금 덜 최적화 된 코드를 생성하도록 설계되었고, C2 는 실행하는데 시간이 좀 더 소요되지만 더 최적화 된 코드를 생성하도록 설계되었다.

### Tiered Compilation

JVM은 호출되는 메서드를 추적하고 자주 호출되는 메서드를 C1을 사용하여 컴파일한다.
C1으로 컴파일된 메서드의 호출수가 증가하면 C2를 사용하여 한번 더 컴파일한다.
이해를 위해 간단하게 적었지만 세부적인 Compilation Levels 은 다음과 같다:

* Level 0 – Interpreted Code
* Level 1 – Simple C1 Compiled Code
* Level 2 – Limited C1 Compiled Code
* Level 3 – Full C1 Compiled Code
* Level 4 – C2 Compiled Code

각 레벨에 대해 더 자세한 내용을 확인하고 싶다면 [Compilation Levels](https://www.baeldung.com/jvm-tiered-compilation#compilation-levels)을 참고한다.
언제 C1, C2 를 사용하여 컴파일이 일어나는지 궁금하다면 다음 명령어를 통해 임계값을 확인할 수 있다:

```shell
java -XX:+PrintFlagsFinal -version | grep Threshold | grep Tier
```

```shell
openjdk version "17.0.7" 2023-04-18
OpenJDK Runtime Environment Temurin-17.0.7+7 (build 17.0.7+7)
OpenJDK 64-Bit Server VM Temurin-17.0.7+7 (build 17.0.7+7, mixed mode, sharing)
    uintx IncreaseFirstTierCompileThresholdAt      = 50                                        {product} {default}
     intx Tier2BackEdgeThreshold                   = 0                                         {product} {default}
     intx Tier2CompileThreshold                    = 0                                         {product} {default}
     intx Tier3BackEdgeThreshold                   = 60000                                     {product} {default}
     intx Tier3CompileThreshold                    = 2000                                      {product} {default}
     intx Tier3InvocationThreshold                 = 200                                       {product} {default}
     intx Tier3MinInvocationThreshold              = 100                                       {product} {default}
     intx Tier4BackEdgeThreshold                   = 40000                                     {product} {default}
     intx Tier4CompileThreshold                    = 15000                                     {product} {default}
     intx Tier4InvocationThreshold                 = 5000                                      {product} {default}
     intx Tier4MinInvocationThreshold              = 600                                       {product} {default}
```

`Tier3InvocationThreshold` 와 `Tier3BackEdgeThreshold`, `Tier3CompileThreshold` 를 살펴보자.
맨 앞의 Tier3는 Level 3 로 컴파일 되기 위한 임계값임을 나타내며 각각의 의미는 다음과 같다:

* Tier3InvocationThreshold: 메서드 호출 횟수의 임계값을 나타낸다.
* Tier3BackEdgeThreshold: 백 엣지의 임계값을 나타낸다.
* Tier3CompileThreshold: 메서드 호출 횟수 임계값과 백 엣지 임계값의 합

> 백 엣지란, 반복문 등에서 이전 실행한 블록으로 돌아가는 분기 구문을 말한다.
> 예를들어 for 반복문의 경우 조건을 검사하고 조건문이 참이라면 다시 for 반복문의 블록으로 돌아가게 되는데, 이를 백 엣지라고 한다.

[이 페이지](https://mail.openjdk.org/pipermail/hotspot-compiler-dev/2010-November/004239.html)에 메서드를 컴파일해야하는지 여부를 판단하기 위한 로직이 설명되어 있으며, 이를 코드로 표현하면 다음과 같다:

```javascript
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

이 로직과 위의 임계값 정보를 바탕으로 메서드를 몇번 호출해야 컴파일이 일어날지 예측해보자. 먼저 위에서 찾은 Tier3 임계값이 다음과 같았다:

```
intx Tier3CompileThreshold                     = 2000
intx Tier3InvocationThreshold                  = 200
intx Tier3MinInvocationThreshold               = 100
```

만약 어떤 메서드가 있고, 메서드를 한번 호출할 때마다 20개의 loop back-edge count 를 생성한다고 해보자.
이 메서드는 100번 호출될 때 컴파일 될 것을 예측할 수 있다. (정확한 횟수는 아니므로 테스트는 해봐야 함.)

```
100 + (20 * 100) > 2000
---   ----------   ----
1     2            3
```

* 1: 메서드 호출 횟수
* 2: back-edge
* 3: Tier3CompileThreshold

실제로 최적화가 일어나는지 확인해보고 싶다면 아래의 VM Options 를 추가하여 확인할 수 있다:

```
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation
```

옵션을 추가하고 프로그램을 실행하면 `hotspot_pid<pid>.log` 형식의 파일이 생성된다. 샘플 코드를 통해 실제로 최적화가 일어나는지 확인해보자:

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    (1..3000).forEach { i ->
        findMax(arr)
        if (i % 100 == 0) {
            // 너무 빨리 종료되면 c2 컴파일이 안될 수도 있어서
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

위 프로그램을 실행하면 `hotspot_pid<pid>.log` 파일이 생긴다. 파일을 열어보면 다음과 같이 level 3 컴파일을 위해 c1 queue 에 메서드가 적재된 것을 확인할 수 있다.

```xml
<task_queued compile_id='205' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='228' backedge_count='2048' iicount='228' level='3' stamp='0.438' comment='tiered' hot_count='228'/>
```

`count` 와 `backedge_count` 를 통해 메서드가 몇 번 호출되었는지와 백엣지 수를 확인할 수 있다.
조금 더 아래 로그를 살펴보면 다음과 같이 level 3 로 코드 최적화가 된 것을 확인할 수 있다.

```xml
<nmethod compile_id='205' compiler='c1' level='3' entry='0x00000208cb1883a0' size='2560' address='0x00000208cb188190' relocation_offset='344' insts_offset='528' stub_offset='1872' scopes_data_offset='2040' scopes_pcs_offset='2216' dependencies_offset='2520' nul_chk_table_offset='2528' oops_offset='1992' metadata_offset='2008' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='228' backedge_count='2048' iicount='228' stamp='0.438'/>
```

마찬가지로 level 4 로 코드가 최적화 된 것도 로그를 통해 확인할 수 있다.

```xml
<task_queued compile_id='207' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='2048' backedge_count='18432' iicount='2048' stamp='2.914' comment='tiered' hot_count='2048'/>
```

```xml
<nmethod compile_id='207' compiler='c2' level='4' entry='0x00000208d2c30820' size='1232' address='0x00000208d2c30690' relocation_offset='344' insts_offset='400' stub_offset='880' scopes_data_offset='936' scopes_pcs_offset='1056' dependencies_offset='1184' handler_table_offset='1192' nul_chk_table_offset='1216' oops_offset='920' metadata_offset='928' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='2100' backedge_count='18900' iicount='2100' stamp='2.917'/>
```

JIT Compiler 가 정말 성능을 향상시켜주는지 알고 싶다면 `-Xint ` 옵션으로 Jit Compiler 사용을 중지하고 테스트 해 볼 수 있다.
앞에서 살펴본 코드를 조금 변경하여 총 3번 실행해보았다.

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val startTimeMills = System.currentTimeMillis()
    (1..1_0000_0000).forEach { i ->
        findMax(arr)
    }
    println("실행시간: ${System.currentTimeMillis() - startTimeMills}ms")
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

JIT Compiler 를 활성화한 경우:
```
실행시간: 1595ms
실행시간: 875ms
실행시간: 1252ms
```

JIT Compiler 를 비활성화 한 경우:
```
실행시간: 28558ms
실행시간: 32401ms
실행시간: 24760ms
```


## 참고 자료

* [https://www.baeldung.com/jvm-tiered-compilation](https://www.baeldung.com/jvm-tiered-compilation)
* [https://www.baeldung.com/graal-java-jit-compiler](https://www.baeldung.com/graal-java-jit-compiler)
* [https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/](https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/)
* [https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html](https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html)
* [https://hg.openjdk.org/jdk8/jdk8/hotspot/file/104743074675/src/share/vm/runtime/advancedThresholdPolicy.hpp](https://hg.openjdk.org/jdk8/jdk8/hotspot/file/104743074675/src/share/vm/runtime/advancedThresholdPolicy.hpp)
* [https://www.youtube.com/watch?v=CQi3SS2YspY&list=PLyGtIjZ_uWKNT5-ob1TL26hH0KVfoJQzw](https://www.youtube.com/watch?v=CQi3SS2YspY&list=PLyGtIjZ_uWKNT5-ob1TL26hH0KVfoJQzw)
