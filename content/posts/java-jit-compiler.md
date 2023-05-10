---
title: "Java JIT Compiler"
description: ""
date: 2023-05-10T23:56:05+09:00
url: "/java-jit-compiler/"
tags: [development]
---

## JIT Compiler 란?

자바 코드의 실행을 위해서는 1차적으로 바이트 코드로 컴파일이 필요하다. 컴파일 된 바이트 코드는 주로 `jar` 이나 `war` 로 아카이브하여 활용된다.
빌드된 파일을 실행하기 위해서는 JVM 이 필요하며, JVM 에서는 바이트 코드를 해석하는 과정(interpret)이 필요하다.
컴파일 - interpret 과정을 거쳐 실행하는 자바 프로그램은, 컴파일 과정에서 바로 기계어를 만들어 런타임 환경에서 즉시 실행하는 C 같은 언어에 비해 많이 느리다.
이러한 성능 차이를 해결하기 위해 JVM 에서는 JIT (Just In Time) Compiler 를 도입하고 있다.
JIT 컴파일러는 javac 컴파일러보다 훨씬 더 최적화 된 고품질의 기계어를 생성한다.

## JIT Compiler 에 대한 자세한 내용

Oracle 에서는 JDK 1.3 부터 HotSpot 이라는 가상 머신을 포함한다. 여기에는 C1 이라고 하는 클라이언트 컴파일러와 C2 라고 하는 서버 컴파일러 두 개의 JIT 컴파일러가 포함되어 있다.
C1은 더 빠르게 실행되고 조금 덜 최적화 된 코드를 생성하도록 설계되었고, C2 는 실행하는데 시간이 좀 더 소요되지만 더 최적화 된 코드를 생성하도록 설계되었다.

### Tiered Compilation

JVM은 호출되는 메서드를 추적하고 자주 호출되는 메서드를 C1을 사용하여 컴파일한다.
C1으로 컴파일된 메서드의 호출수가 증가하면 C2를 사용하여 한번 더 컴파일한다.
간단하게 적었지만 세부적인 Compilation Levels 은 다음과 같다: 

* Level 0 – Interpreted Code
* Level 1 – Simple C1 Compiled Code
* Level 2 – Limited C1 Compiled Code
* Level 3 – Full C1 Compiled Code
* Level 4 – C2 Compiled Code

각 레벨에 대해 더 자세한 내용을 확인하고 싶다면 [Compilation Levels](https://www.baeldung.com/jvm-tiered-compilation#compilation-levels)을 참고한다.
얼마나 자주 메서드를 호출해야 C1, C2 를 사용하여 컴파일이 되는지 궁금하다면 다음 명령어를 통해 임계값을 확인할 수 있다:

```shell
$ java -XX:+PrintFlagsFinal -version | grep Threshold | grep Tier
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
---    ---------   ----
1      2           3
```

* 1: 메서드 호출 횟수
* 2: back-edge
* 3: Tier3CompileThreshold

## 참고 자료

* [https://www.baeldung.com/graal-java-jit-compiler](https://www.baeldung.com/graal-java-jit-compiler)
* [https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/](https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/)
* [https://www.baeldung.com/jvm-tiered-compilation](https://www.baeldung.com/jvm-tiered-compilation)
* [https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html](https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html)
* [https://hg.openjdk.org/jdk8/jdk8/hotspot/file/104743074675/src/share/vm/runtime/advancedThresholdPolicy.hpp](https://hg.openjdk.org/jdk8/jdk8/hotspot/file/104743074675/src/share/vm/runtime/advancedThresholdPolicy.hpp)
* [https://www.youtube.com/watch?v=CQi3SS2YspY&list=PLyGtIjZ_uWKNT5-ob1TL26hH0KVfoJQzw](https://www.youtube.com/watch?v=CQi3SS2YspY&list=PLyGtIjZ_uWKNT5-ob1TL26hH0KVfoJQzw)
