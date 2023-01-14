---
title: "Number Formatter in Java"
description: "간단한 예제를 통해 DecimalFormat 의 사용법을 알아보자."
date: 2021-11-01T20:00:12+09:00
url: /number-formatter-in-java/
tags: ["java"]
---

## 개요

[DecimalFormat](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/text/DecimalFormat.html)은 미리 정의된 포맷을 사용하여 10진수 문자열 표현을 형식화 할 수 있는 `NumberFormat` 의 하위 클래스이다. 역으로 문자열을 숫자로 구문 분석하는 데 사용할 수도 있다. 이번 포스팅에서는 `DecimalFormat` 의 사용법을 알아본다.

## 패턴 문자

숫자를 어떤 형식으로 나타낼 지 지정하기 위해서는 먼저 패턴 문자를 알아야한다. 총 11가지 문자가 있지만, 네 가지만 알고 있으면 대부분의 상황에서 문제 없이 사용할 수 있다.

* `0` : 값이 제공되면 숫자를, 그렇지 않다면 0을 출력
* `#` : 값이 제공되면 숫자를, 그렇지 않다면 아무것도 출력하지 않음
* `.` : 소수점 구분 기호를 넣을 위치를 지정
* `,` : 그룹화 구분 기호를 넣을 위치를 지정

`DecimalFormat` 사용 시 패턴이 지정될 경우 지정된 규칙이 실행되고, 그렇지 않은 경우는 JVM `Locale` 의 `DecimalFormatSymbol` 에 따라 규칙이 실행된다.

## 기본 포맷팅

실제 코드를 통해 포맷을 적용해보자.

### Simple Decimal

정수 부분은 패턴 문자가 입력되는 문자보다 개수가 더 적더라도 잘리지 않는다. 다음의 테스트 코드를 통해 확인할 수 있다.

```java
double d = 123.45;
Assertions.assertEquals(new DecimalFormat("#.##").format(d), "123.45");
Assertions.assertEquals(new DecimalFormat("0.00").format(d), "123.45");
```

정수와 소수 부분의 패턴 문자가 입력되는 문자보다 길 경우는 `#` 인 경우 삭제되고, `0` 인 경우는 0 이 채워지는 것을 볼 수 있다.

```java
double d = 123.45;
assertEquals(new DecimalFormat("####.###").format(d), "123.45");
assertEquals(new DecimalFormat("0000.000").format(d), "0123.450");
```

### Rounding (반올림)

앞에서 정수 부분은 패턴 문자가 입력되는 문자보다 개수가 더 적더라도 잘리지 않는다고 하였다. 하지만 소수 부분은 패턴 문자가 더 적은 경우, 패턴 문자의 길이에 맞게 반올림 된다.

```java
double d = 123.45;
assertEquals(new DecimalFormat("#.#").format(d), "123.5");
assertEquals(new DecimalFormat("#").format(d), "123");
```

### Grouping (그룹핑)

`,` 패턴 문자는 다음과 같이 사용한다.

```java
double d = 1234567.89;
assertEquals(new DecimalFormat("#,###.#").format(d), "1,234,567.9");
assertEquals(new DecimalFormat("#,###").format(d), "1,234,568");
```

### Mixing String Literals (문자열 리터럴 혼용)

문자열 리터럴과 패턴을 혼용하여 사용할 수 있다.

```java
double d = 1234567.89;
assertEquals(new DecimalFormat("The # number").format(d), "The 1234568 number");
```

다음과 같은 방법을 통해 문자열 리터럴에 특수 문자를 사용 수도 있다.

```java
double d = 1234567.89;
assertEquals(new DecimalFormat("The '#' # number").format(d), "The # 1234568 number");
```

## Localized Formatting

이탈리아 같은 몇몇의 나라에서는 그룹핑 문자로 `.` 를 사용하고 소수 구분 기호로 `,` 를 사용한다. 이런 나라에서는 `#,###.##` 패턴을 이용할 시 `1.234.567,89` 로 포맷팅 된다. 경우에 따라 이는 유용한 i18n 기능이 될 수도 있지만, 그렇지 않은 경우도 있을 것이다. 이럴 때에는 `DecimalFormatSymbols` 을 사용한다.

```java
double d = 1234567.89;
assertEquals(new DecimalFormat("#,###.##", new DecimalFormatSymbols(Locale.ENGLISH)).format(d), "1,234,567.89");
assertEquals(new DecimalFormat("#,###.##", new DecimalFormatSymbols(Locale.ITALIAN)).format(d), "1.234.567,89");
```

## Parsing

다음과 같이 문자열을 숫자로 파싱이 가능하다.

```java
assertEquals(new DecimalFormat("", new DecimalFormatSymbols(Locale.ENGLISH)).parse("1234567.89"), 1234567.89);
```

## Thread-Safety

`DecimalFormat` 은 스레드에 안전하지 않기 때문에 스레드 간 동일 인스턴스를 공유할 때 주의하여야 한다.

## 참조

[A Practical Guide to DecimalFormat](https://www.baeldung.com/java-decimalformat)

