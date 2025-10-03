---
title: "Java 성능의 비밀, JIT 컴파일러: 원리와 최적화 방법 알아보기"
description: "자바가 C++, C#과 같은 네이티브 언어에 버금가는 성능을 내는 비결, JIT(Just-In-Time) 컴파일러에 대해 알아봅니다. JIT 컴파일러의 작동 원리, 계층적 컴파일(Tiered Compilation), 그리고 실제 성능 테스트를 통해 JIT의 강력한 효과를 자세히 살펴봅니다."
date: 2023-05-10T23:56:05+09:00
lastmod: 2024-12-29T19:45:00+09:00
tags: ["java", "jvm", "performance"]
---

## 들어가며: 자바는 어떻게 빠른 속도를 얻게 되었을까?

자바 코드는 먼저 자바 컴파일러(`javac`)를 통해 자바 바이트코드(.class 파일)로 변환됩니다. 이 바이트코드는 특정 하드웨어가 아닌 JVM(자바 가상 머신) 위에서 동작하므로, 어떤 운영체제에서든 실행될 수 있는 '플랫폼 독립성'을 가집니다. JVM은 이 바이트코드를 한 줄씩 해석(interpreting)하여 기계어로 번역하고 실행합니다.

하지만 인터프리터 방식은 코드를 실행할 때마다 번역 과정을 거쳐야 하므로, 기계어로 직접 컴파일되어 실행되는 네이티브 언어(C, C++ 등)에 비해 속도가 느리다는 단점이 있었습니다. 이러한 성능 문제를 해결하기 위해 현대의 JVM은 **JIT(Just-In-Time) 컴파일러** 라는 강력한 기술을 도입했습니다.

JIT 컴파일러는 프로그램 실행 중에 자주 사용되는 '뜨거운(hot)' 코드를 실시간으로 감지하여, 해당 부분만 네이티브 기계어로 컴파일합니다. 이렇게 컴파일된 코드는 다음부터는 번역 과정 없이 기계에서 직접 실행되므로, 인터프리터 방식보다 훨씬 빠른 속도를 낼 수 있습니다. 이 글에서는 JVM 성능의 핵심인 JIT 컴파일러의 작동 원리와 최적화 방식을 자세히 알아보겠습니다.

## JIT 컴파일러의 작동 방식: HotSpot VM

Oracle의 HotSpot VM에는 두 가지 종류의 JIT 컴파일러가 포함되어 있습니다.

-   **C1 (Client Compiler)**: 컴파일 속도가 빠르지만, 최적화 수준은 상대적으로 낮습니다. 애플리케이션의 빠른 시작에 중점을 둡니다.
-   **C2 (Server Compiler)**: C1보다 컴파일하는 데 시간이 더 오래 걸리지만, 매우 높은 수준의 최적화를 수행하여 가장 빠른 코드를 생성합니다. 장시간 실행되는 서버 애플리케이션의 최대 성능에 중점을 둡니다.

## 계층적 컴파일 (Tiered Compilation)

최신 JVM은 C1과 C2 컴파일러의 장점을 모두 활용하기 위해 **계층적 컴파일(Tiered Compilation)** 이라는 전략을 사용합니다. 이는 메서드가 얼마나 자주 호출되는지에 따라 컴파일 수준을 점진적으로 높여가는 방식입니다.

-   **Level 0**: 인터프리터 모드로 실행 (프로파일링 정보 수집)
-   **Level 1**: C1 컴파일러로 간단하게 컴파일 (최적화 없음)
-   **Level 2**: C1 컴파일러로 제한적인 최적화와 함께 컴파일
-   **Level 3**: C1 컴파일러로 모든 최적화를 적용하여 컴파일
-   **Level 4**: C2 컴파일러로 최대 수준의 최적화를 적용하여 컴파일

마치 신입사원(메서드)이 처음에는 매뉴얼(인터프리터)을 보고 일을 하다가, 특정 업무를 자주 하게 되면 간단한 직무 교육(C1 컴파일)을 받고, 그 업무의 전문가가 되면 최고 수준의 전문 교육(C2 컴파일)을 받는 과정과 유사합니다. 이 방식을 통해 JVM은 애플리케이션의 시작 시간과 최대 성능 사이의 균형을 맞춥니다.

### 컴파일 임계값 확인 및 설정

JVM이 특정 메서드를 언제 컴파일할지 결정하는 기준을 **임계값(Threshold)** 이라고 합니다. 다음 명령어를 통해 현재 JVM의 기본 임계값을 확인할 수 있습니다.

```shell
java -XX:+PrintFlagsFinal -version | grep Threshold | grep Tier
```

**출력 예시:**
```shell
intx Tier3CompileThreshold      = 2000 {product} {default}
intx Tier3InvocationThreshold   = 200  {product} {default}
intx Tier3MinInvocationThreshold = 100  {product} {default}
```

-   `Tier3InvocationThreshold`: 메서드가 C1(Level 3)으로 컴파일되기 위한 최소 호출 횟수입니다. (기본값: 200회)
-   `Tier3CompileThreshold`: 메서드 호출 횟수와 메서드 내 루프가 반복 실행된 횟수(back-edge count)의 합이 이 값을 초과하면 컴파일됩니다. (기본값: 2000)

간단히 말해, JVM은 메서드가 **자주 호출되거나(Invocation)**, 내부의 **루프가 많이 실행될 때(Back-edge)** 해당 메서드를 '뜨거운 코드'로 판단하고 JIT 컴파일을 수행합니다.

### JIT 컴파일 최적화 확인 방법

JIT 컴파일러의 동작을 직접 확인하려면, 자바 애플리케이션 실행 시 다음 VM 옵션을 추가합니다.

```shell
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation
```

이 옵션을 추가하고 프로그램을 실행하면, 프로젝트 루트 디렉터리에 `hotspot_pid<PID>.log` 형태의 로그 파일이 생성됩니다. 아래 샘플 코드를 실행하여 실제로 최적화가 일어나는지 확인해 보겠습니다.

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    // findMax 메서드를 3000번 호출
    (1..3000).forEach { i ->
        findMax(arr)
        if (i % 100 == 0) {
            // C2 컴파일이 수행될 시간을 확보하기 위해 잠시 대기
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

생성된 로그 파일을 열어보면 다음과 유사한 내용을 찾을 수 있습니다.

```xml
<task_queued compile_id='205' method='com.kotlin.JitTestKt findMax ([I)I' bytes='39' count='228' backedge_count='2048' iicount='228' level='3' stamp='0.438' comment='tiered' hot_count='228'/>
```

위 로그는 `findMax` 메서드가 228번 호출(`count='228'`)되고 루프가 2048번 실행(`backedge_count='2048'`)되어, Level 3 컴파일을 위해 C1 컴파일러 큐에 등록되었음을 의미합니다.

조금 더 아래 로그를 살펴보면, 컴파일이 완료된 것을 확인할 수 있습니다.

```xml
<nmethod compile_id='205' compiler='c1' level='3' entry='0x...' ... method='com.kotlin.JitTestKt findMax ([I)I' ... />
```

## JIT 컴파일러 성능 비교

JIT 컴파일러가 실제로 성능에 얼마나 큰 영향을 미치는지 확인하기 위해, JIT 컴파일러를 활성화했을 때와 비활성화했을 때의 실행 시간을 비교해 보겠습니다. `-Xint` VM 옵션을 사용하면 JIT 컴파일러를 끄고 순수 인터프리터 모드로만 실행할 수 있습니다.

```kotlin
fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val startTimeMills = System.currentTimeMillis()
    // 1억 번 호출
    (1..1_0000_0000).forEach { findMax(arr) }
    println("실행시간: ${System.currentTimeMillis() - startTimeMills}ms")
}

fun findMax(arr: IntArray): Int { ... }
```

#### 실행 결과

-   **JIT 컴파일러 활성화 (기본 상태)**
    -   실행시간: 약 **875ms ~ 1,595ms**
-   **JIT 컴파일러 비활성화 (`-Xint` 옵션 사용)**
    -   실행시간: 약 **24,760ms ~ 32,401ms**

결과에서 볼 수 있듯이, JIT 컴파일러가 활성화되었을 때 **약 20배 이상** 성능이 향상되었습니다. 이는 JIT 컴파일러가 현대 자바 애플리케이션의 고성능을 뒷받침하는 핵심 기술임을 명확히 보여줍니다.

> **⚠️ 주의: IntelliJ 사용자를 위한 팁**
> Spring Boot 애플리케이션을 IntelliJ IDEA에서 기본 설정으로 실행하면, 빠른 시작을 위해 `-XX:TieredStopAtLevel=1` 옵션이 자동으로 추가됩니다. 이 경우 JIT 컴파일이 Level 1에서 멈추어 C2 컴파일러의 최적화가 이루어지지 않습니다. 정확한 성능 테스트나 JIT 동작을 확인하려면 **Run/Debug Configurations -> Modify options -> Do not build before run** 을 선택하여 이 최적화 옵션을 비활성화해야 합니다. (IntelliJ 버전에 따라 메뉴 이름은 다를 수 있습니다. 'launch optimization' 관련 옵션을 찾아 비활성화하세요.)

## 결론

JIT 컴파일러는 자바의 '느리다'는 초창기 인식을 완전히 뒤바꾼 혁신적인 기술입니다. 프로그램 실행 중에 실시간으로 코드를 분석하고 최적화함으로써, 자바가 네이티브 언어에 필적하는 높은 성능을 달성할 수 있도록 해줍니다. JIT 컴파일러의 기본 원리를 이해하는 것은 자바 애플리케이션의 성능을 분석하고 최적화하는 데 큰 도움이 될 것입니다.

---

**참고 자료:**
- [Baeldung - Tiered Compilation in JVM](https://www.baeldung.com/jvm-tiered-compilation)
- [Oracle - Java Performance: The Definite Guide](https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html)
- [YouTube - The Java JIT Compiler](https://www.youtube.com/watch?v=CQi3SS2YspY)