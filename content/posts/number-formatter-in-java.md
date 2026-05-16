---
title: "Java 숫자 포매팅: DecimalFormat으로 원하는 모든 형식 만들기"
description: "Java의 DecimalFormat 클래스로 숫자를 원하는 형식의 문자열로 만드는 방법을 정리합니다. 주요 패턴 문자, 소수점 처리, 그룹핑, 지역화, 스레드 안전성 문제를 함께 다룹니다."
date: 2021-11-01T20:00:12+09:00
tags: ["java", "formatting", "decimalformat"]
---

## 들어가며: 숫자를 보기 좋게 표현하는 방법

애플리케이션에서 숫자를 다룰 때, 단순히 값을 보여주는 것을 넘어 사용자가 읽기 편한 형태로 가공해야 하는 경우가 많습니다. 예를 들어, `1234567.89`라는 숫자를 "1,234,567.89"나 "$1,234,567.89"와 같이 표현하는 것입니다. 

Java에서는 `java.text.DecimalFormat` 클래스로 이런 숫자 포매팅을 처리할 수 있습니다. `DecimalFormat`은 `NumberFormat`의 하위 클래스로, 특정 패턴을 사용해 숫자를 문자열로 변환하거나 형식화된 문자열을 다시 숫자로 파싱합니다.

이 글에서는 `DecimalFormat`의 기본 사용법을 예제와 함께 정리합니다.

## 핵심 패턴 문자 이해하기

`DecimalFormat`은 패턴 문자를 조합해 출력 형식을 정합니다. 자주 쓰는 패턴 문자는 다음과 같습니다.

| 문자 | 의미 | 설명 |
| :--- | :--- | :--- |
| `0` | 숫자 (Digit) | 해당 자리에 숫자가 있으면 표시하고, **없으면 0으로 채웁니다.** (Zero Padding) |
| `#` | 숫자 (Digit) | 해당 자리에 숫자가 있으면 표시하고, **없으면 아무것도 표시하지 않습니다.** |
| `.` | 소수점 구분 기호 | 소수점의 위치를 지정합니다. |
| `,` | 그룹핑 구분 기호 | 천 단위와 같이 숫자를 그룹으로 묶는 구분 기호의 위치를 지정합니다. |
| `¤` | 통화 기호 | 로케일에 맞는 통화 기호(예: $, \)로 치환됩니다. |
| `%` | 백분율 기호 | 숫자에 100을 곱하고 뒤에 % 기호를 붙입니다. |

`0`과 `#`의 차이를 잘 구분해야 합니다. `0`은 자리 수를 강제로 맞추고 싶을 때, `#`은 불필요한 0을 제거하고 싶을 때 사용합니다.

## 주요 포매팅 예제

몇 가지 포매팅 예제를 보겠습니다.

### 1. 소수점 자릿수 제어 및 반올림

패턴에서 소수점 이하 자릿수를 지정하면, 그에 맞게 숫자가 **반올림** 됩니다.

```java
double num = 123.456;

// 소수점 둘째 자리까지 표시 (셋째 자리에서 반올림)
Assertions.assertEquals("123.46", new DecimalFormat("#.##").format(num));

// 소수점 첫째 자리까지 표시 (둘째 자리에서 반올림)
Assertions.assertEquals("123.5", new DecimalFormat("0.0").format(num));

// 정수 부분만 표시 (소수점 첫째 자리에서 반올림)
Assertions.assertEquals("123", new DecimalFormat("#").format(num));
```

### 2. 0으로 자릿수 채우기 (Zero Padding)

`0` 패턴을 사용하면 원하는 자릿수를 0으로 채울 수 있습니다. 고정 길이의 숫자 문자열을 만들 때 유용합니다.

```java
double num = 12.3;

// 정수 4자리, 소수 3자리로 강제. 부족한 부분은 0으로 채움.
Assertions.assertEquals("0012.300", new DecimalFormat("0000.000").format(num));

// # 패턴은 0을 채우지 않음.
Assertions.assertEquals("12.3", new DecimalFormat("####.###").format(num));
```

### 3. 천 단위 구분 기호 (Grouping)

`,` 패턴을 사용하여 큰 숫자의 가독성을 높일 수 있습니다. `#,##0` 패턴이 가장 일반적으로 사용됩니다.

```java
double largeNum = 1234567.89;

Assertions.assertEquals("1,234,567.89", new DecimalFormat("#,##0.00").format(largeNum));
Assertions.assertEquals("1,234,568", new DecimalFormat("#,##0").format(largeNum)); // 반올림됨
```

### 4. 통화 및 백분율 포매팅

`¤`와 `%` 패턴을 사용하여 통화 및 백분율을 쉽게 표현할 수 있습니다.

```java
double money = 5678.9;
double rate = 0.75;

// 로케일이 US라고 가정
Assertions.assertEquals("$5,678.90", new DecimalFormat("¤#,##0.00").format(money));

// 숫자에 100을 곱한 후 % 기호를 붙임
Assertions.assertEquals("75%", new DecimalFormat("##%").format(rate));
```

### 5. 문자열 리터럴 혼합

패턴 안에 일반 문자열을 포함하여 형식을 꾸밀 수 있습니다. 특수 문자를 리터럴로 사용하고 싶을 때는 작은따옴표(`'`)로 감싸줍니다.

```java
long userId = 8827L;

Assertions.assertEquals("User ID: 8827", new DecimalFormat("'User ID:' #").format(userId));
```

## 지역화(Localization) 포매팅

국가별로 숫자 표기법이 다릅니다. 예를 들어, 미국에서는 `1,234.56`으로 표기하지만, 독일에서는 `1.234,56`으로 표기합니다. `DecimalFormat`은 `Locale` 정보를 이용하여 이러한 지역화된 포매팅을 자동으로 처리할 수 있습니다.

```java
double num = 1234567.89;
String pattern = "#,###.##";

// 영어권 로케일
DecimalFormatSymbols symbolsEn = new DecimalFormatSymbols(Locale.ENGLISH);
Assertions.assertEquals("1,234,567.89", new DecimalFormat(pattern, symbolsEn).format(num));

// 이탈리아 로케일
DecimalFormatSymbols symbolsIt = new DecimalFormatSymbols(Locale.ITALIAN);
Assertions.assertEquals("1.234.567,89", new DecimalFormat(pattern, symbolsIt).format(num));
```

## 문자열을 숫자로 파싱하기

`parse()` 메서드를 사용하면 형식화된 문자열을 다시 `Number` 객체로 변환할 수 있습니다. 파싱할 문자열이 지정된 패턴과 일치하지 않으면 `ParseException`이 발생할 수 있으므로 예외 처리가 필요합니다.

```java
String numStr = "1,234.56";
DecimalFormat format = new DecimalFormat("#,##0.00");

try {
    Number parsedNum = format.parse(numStr);
    Assertions.assertEquals(1234.56, parsedNum.doubleValue());
} catch (ParseException e) {
    e.printStackTrace();
}
```

## 스레드 안전성 문제

`DecimalFormat` 클래스는 **스레드에 안전하지 않습니다(not thread-safe)**. 이는 여러 스레드에서 **동일한 `DecimalFormat` 인스턴스를 공유하며 동시에 사용하면** 예기치 않은 결과가 나오거나 오류가 발생할 수 있음을 의미합니다.

웹 애플리케이션 같은 멀티스레드 환경에서는 이 문제를 피해야 합니다. 선택지는 다음과 같습니다.

1.  **매번 새로 생성하기**: 메서드 내에서 매번 `new DecimalFormat(...)`을 호출하여 지역 변수로 사용합니다. 간단하지만, 객체 생성 비용이 발생할 수 있습니다.
2.  **`ThreadLocal` 사용하기**: 각 스레드마다 별도의 `DecimalFormat` 인스턴스를 갖도록 `ThreadLocal`로 감싸서 사용합니다. 객체를 재사용하면서 스레드 간 공유를 피할 수 있습니다.

    ```java
    private static final ThreadLocal<DecimalFormat> myFormatter = 
        ThreadLocal.withInitial(() -> new DecimalFormat("#,##0.00"));
    
    public String formatNumber(double num) {
        return myFormatter.get().format(num);
    }
    ```

## 결론

`DecimalFormat`은 다양한 숫자 형식을 만들 수 있는 편리한 도구입니다. 다만 **스레드에 안전하지 않다** 는 점을 잊으면 안 됩니다. 멀티스레드 환경에서는 매번 새로 만들거나 `ThreadLocal`을 사용하는 방식으로 인스턴스 공유를 피해야 합니다.

---

**참고 자료:**
- [Baeldung - A Practical Guide to DecimalFormat](https://www.baeldung.com/java-decimalformat)
- [Oracle Java 11 Documentation - DecimalFormat](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/text/DecimalFormat.html)

---
*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
