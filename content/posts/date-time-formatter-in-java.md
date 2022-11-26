---
title: "Date Time Formatter in Java"
date: 2021-09-29T19:33:31+09:00
url: /date-time-formatter-in-java/
tags: ["java"]
---

## 개요

Java8 에서 추가된 `DateTimeFormatter` 클래스에 대해 알아보자.

## 미리 정의된 인스턴스

`DateTimeFormatter` 에는 ISO 및 RFC 표준을 따라 정의되어 있는 날짜/시간 포맷을 제공한다. 예를들어 `ISO_LOCAL_DATE` 인스턴스를 사용하여 다음과 같이 '2021-09-29' 와 같은 문자열을 얻을 수 있다.

```java
LocalDate date = LocalDate.of(2021, 9, 29);
DateTimeFormatter.ISO_LOCAL_DATE.format(date); // 2021-09-29
```

만약 '2021-09-29+09:00' 와 같이 오프셋을 포함한 문자열을 구하고 싶다면 `ISO_OFFSET_DATE` 를 사용한다.

```java
LocalDate date = LocalDate.of(2021, 9, 29);
DateTimeFormatter.ISO_OFFSET_DATE.format(date.atStartOfDay(ZoneId.of("UTC+9"))); // 2021-09-29+09:00
```

## FormatStyle의 사용

사람이 이해하기 쉽게 날짜를 보여주고 싶을때가 있다. 이럴 때에는 `java.time.format.FormatStyle` 을 사용할 수 있다. `FormatStyle` 은 enum 값으로 FULL, LONG, MEDIUM, SHORT가 정의 되어있다.

```java
LocalDate day = LocalDate.of(2021, 9, 29);
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.FULL).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.LONG).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.SHORT).format(day));
```

출력은 다음과 같다.

```
Wednesday, September 29, 2021
September 29, 2021
Sep 29, 2021
9/29/21
```

`ZonedDateTime` 인스턴스를 사용하여 날짜와 시간을 함께 표현할 수도 있다.

```java
LocalDate day = LocalDate.of(2021, 9, 29);
LocalTime time = LocalTime.of(13, 12, 45);
ZonedDateTime zonedDateTime = ZonedDateTime.of(day, time, ZoneId.of("Asia/Seoul"));
System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).format(zonedDateTime));
System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.LONG).format(zonedDateTime));
System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).format(zonedDateTime));
System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).format(zonedDateTime));
```

출력은 다음과 같다.

```
Wednesday, September 29, 2021 at 1:12:45 PM Korean Standard Time
September 29, 2021 at 1:12:45 PM KST
Sep 29, 2021, 1:12:45 PM
9/29/21, 1:12 PM
```

반대로 문자열을 `ZonedDateTime` 으로 변경하고 싶다면 `format()` 메서드 대신 `parse()` 메서드를 사용한다.

```java
ZonedDateTime dateTime = ZonedDateTime.from(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).parse("Wednesday, September 29, 2021 at 1:12:45 PM Korean Standard Time"));
```

## 사용자 정의 포맷

미리 정의된 포맷이 아닌 직접 포맷을 정의하여 사용하고 싶을 때가 있다. 이럴때에는 `ofPattern()` 메서드를 사용한다.

```java
String pattern = "yyyy-MM-dd'T'HH:mm:ss";
DateTimeFormatter formatter = DateTimeFormatter.ofPattern(pattern);
System.out.println(formatter.format(LocalDateTime.now())); // 2021-09-29T12:26:18
```

패턴 문자의 수는 중요하다. 예를들어 month에 `MM` 과 같은 패턴을 사용한다면 1월을 "01"로 표기하며, `M` 과 같이 표기할 경우 1월을 "1"로 표기한다.

자주 사용하는 패턴 문자는 다음과 같다.

| 문자 | 의미 | 표시 | 예시 |
| --- | --- | --- | --- |
| u | year | year | 2004; 04 |
| y | year-of-era | year | 2004; 04 |
| M/L | month-of-year | number/text | 7; 07; Jul; July; J |
| d | day-of-month | number | 10 |
| H | hour-of-day (0-23) | number | 0 |
| m | minute-of-hour | number | 30 |
| s | second-of-minute | number | 55 |
| S | fraction-of-second | number | 978 |
| n | nano-of-second | number | 987654321 |

추가적인 패턴 문자를 알고 싶다면 [Java documentation](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)의 Pattern Letters and Symbols 표를 확인한다.

## 마무리

이번 포스트에서는 `DateTimeFormatter` 에 대하여 알아보았다. `DateTimeFormatter` 은 `SimpleDateFormat` 과 달리 스레드에 안전하고 더 최적화 되어있으므로 만약 java8 이상 버전을 사용한다면 `SimpleDateFormat` 대신 `DateTimeFormatter` 을 사용하자.

참고 자료: [Guide to DateTimeFormatter](https://www.baeldung.com/java-datetimeformatter)
