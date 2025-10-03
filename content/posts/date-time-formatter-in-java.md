---
title: "Java DateTimeFormatter 완벽 가이드: 날짜와 시간 포매팅의 모든 것"
description: "Java 8에 도입된 java.time.format.DateTimeFormatter의 사용법을 알아봅니다. 기존 SimpleDateFormat의 문제점을 해결하고, 스레드에 안전하며 강력한 날짜/시간 포매팅 및 파싱 기능을 제공하는 방법을 예제와 함께 설명합니다."
date: 2021-09-29T19:33:31+09:00
tags: ["java", "datetime", "java8"]
---

## 들어가며: SimpleDateFormat을 넘어서

Java 8 이전에는 날짜와 시간을 포매팅하기 위해 주로 `java.text.SimpleDateFormat` 클래스를 사용했습니다. 하지만 `SimpleDateFormat`은 **스레드에 안전하지 않고(not thread-safe)**, 객체가 **변경 가능(mutable)**하여 멀티스레드 환경에서 예기치 않은 문제를 일으킬 수 있었습니다.

이러한 문제들을 해결하기 위해 Java 8에서는 `java.time` 패키지와 함께 스레드에 안전하고 불변(immutable)인 **`java.time.format.DateTimeFormatter`** 클래스를 도입했습니다. 이 글에서는 `DateTimeFormatter`를 사용하여 날짜와 시간을 원하는 형식의 문자열로 변환하거나, 문자열을 날짜 객체로 파싱하는 다양한 방법을 알아보겠습니다.

## 1. 미리 정의된 표준 포맷 사용하기

`DateTimeFormatter` 클래스는 ISO 표준 및 RFC 표준을 따르는 여러 가지 기본 포맷터를 상수로 제공합니다. 이를 통해 별도의 패턴 지정 없이도 표준 형식의 날짜/시간 문자열을 쉽게 생성할 수 있습니다.

예를 들어, `ISO_LOCAL_DATE`를 사용하면 'YYYY-MM-DD' 형식의 문자열을 얻을 수 있습니다.

```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

LocalDate date = LocalDate.of(2021, 9, 29);
String formattedDate = DateTimeFormatter.ISO_LOCAL_DATE.format(date);

System.out.println(formattedDate); // 출력: 2021-09-29
```

만약 시간대 오프셋 정보가 포함된 '2021-09-29+09:00'과 같은 문자열을 원한다면 `ISO_OFFSET_DATE`를 사용할 수 있습니다.

```java
import java.time.LocalDate;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;

LocalDate date = LocalDate.of(2021, 9, 29);
String formattedDate = DateTimeFormatter.ISO_OFFSET_DATE.format(date.atStartOfDay(ZoneId.of("Asia/Seoul")));

System.out.println(formattedDate); // 출력: 2021-09-29+09:00
```

## 2. 지역화된 스타일 사용하기 (FormatStyle)

사용자가 거주하는 국가나 문화권에 맞춰 자연스러운 형식으로 날짜와 시간을 보여주고 싶을 때가 있습니다. 이럴 때는 `java.time.format.FormatStyle` 열거형을 사용할 수 있습니다. `FormatStyle`은 `FULL`, `LONG`, `MEDIUM`, `SHORT` 네 가지 스타일을 제공하며, JVM의 기본 로케일(Locale)에 따라 다른 형식의 문자열을 생성합니다.

```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.time.format.FormatStyle;

LocalDate day = LocalDate.of(2021, 9, 29);

// 기본 로케일이 US라고 가정
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.FULL).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.LONG).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM).format(day));
System.out.println(DateTimeFormatter.ofLocalizedDate(FormatStyle.SHORT).format(day));
```

**출력 결과 (Locale: US):**
```
Wednesday, September 29, 2021
September 29, 2021
Sep 29, 2021
9/29/21
```

`ZonedDateTime` 객체를 사용하면 날짜와 시간을 함께 표현할 수도 있습니다.

```java
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.ZonedDateTime;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.time.format.FormatStyle;

LocalDate day = LocalDate.of(2021, 9, 29);
LocalTime time = LocalTime.of(13, 12, 45);
ZonedDateTime zonedDateTime = ZonedDateTime.of(day, time, ZoneId.of("Asia/Seoul"));

System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).format(zonedDateTime));
System.out.println(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).format(zonedDateTime));
```

**출력 결과 (Locale: US):**
```
Wednesday, September 29, 2021 at 1:12:45 PM Korean Standard Time
Sep 29, 2021, 1:12:45 PM
```

## 3. 사용자 정의 포맷 패턴 사용하기

가장 일반적으로 사용되는 방법으로, `ofPattern()` 메서드를 통해 원하는 형식을 직접 정의할 수 있습니다.

```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

String pattern = "yyyy-MM-dd'T'HH:mm:ss";
DateTimeFormatter formatter = DateTimeFormatter.ofPattern(pattern);

System.out.println(formatter.format(LocalDateTime.now())); // 예: 2021-09-29T12:26:18
```

패턴에서 사용되는 문자의 개수는 출력 형식에 영향을 줍니다. 예를 들어, 월(month)을 표기할 때 `M`은 '9'로, `MM`은 '09'로, `MMM`은 'Sep'으로, `MMMM`은 'September'로 표기됩니다.

### 자주 사용하는 패턴 문자

| 문자 | 의미 | 예시 | 설명 |
| :--- | :--- | :--- | :--- |
| `y` | 연도(Year) | `2024`, `24` | `yy`는 두 자리, `yyyy`는 네 자리로 표기됩니다. |
| `M` | 월(Month) | `7`, `07`, `Jul`, `July` | 개수에 따라 숫자, 텍스트, 축약형으로 표기됩니다. |
| `d` | 일(Day of month) | `1`, `10` | `d`는 한 자리, `dd`는 두 자리로 표기됩니다. |
| `E` | 요일(Day of week) | `Tue`, `Tuesday` | `E`는 축약형, `EEEE`는 전체 이름으로 표기됩니다. |
| `a` | 오전/오후(AM/PM) | `AM`, `PM` | |
| `H` | 시(Hour, 0-23) | `0`, `23` | `HH`는 두 자리로 표기됩니다. |
| `h` | 시(Hour, 1-12) | `1`, `12` | `hh`는 두 자리로 표기됩니다. |
| `m` | 분(Minute) | `5`, `30` | `mm`은 두 자리로 표기됩니다. |
| `s` | 초(Second) | `5`, `55` | `ss`는 두 자리로 표기됩니다. |
| `S` | 밀리초(Fraction of second) | `978` | 소수점 이하 초를 나타냅니다. |

더 많은 패턴 문자는 [공식 Java 문서](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)의 'Patterns for Formatting and Parsing' 섹션에서 확인할 수 있습니다.

## 4. 문자열을 날짜 객체로 파싱하기

`DateTimeFormatter`는 포매팅뿐만 아니라 문자열을 `LocalDate`나 `LocalDateTime` 같은 날짜/시간 객체로 변환(파싱)하는 데에도 사용됩니다. `parse()` 메서드를 사용하면 됩니다.

```java
String dateString = "2024-01-05";
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

LocalDate parsedDate = LocalDate.parse(dateString, formatter);
System.out.println(parsedDate.getYear()); // 2024
```

만약 `FormatStyle`을 사용해 생성된 지역화된 문자열을 파싱하고 싶다면 다음과 같이 할 수 있습니다.

```java
String localizedDateTimeString = "Wednesday, September 29, 2021 at 1:12:45 PM Korean Standard Time";
DateTimeFormatter fullFormatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).withLocale(Locale.US);

ZonedDateTime parsedZonedDateTime = ZonedDateTime.parse(localizedDateTimeString, fullFormatter);
System.out.println(parsedZonedDateTime);
```

## 5. 스레드 안전성: DateTimeFormatter가 더 나은 이유

앞서 언급했듯이, `SimpleDateFormat`의 가장 큰 문제점 중 하나는 스레드에 안전하지 않다는 것입니다. 여러 스레드가 동일한 `SimpleDateFormat` 인스턴스에 동시에 접근하면 날짜가 잘못 파싱되거나 예외가 발생할 수 있습니다.

반면, **`DateTimeFormatter`는 불변(immutable) 객체**이므로 여러 스레드에서 동시에 사용해도 아무런 문제가 없습니다. 따라서 `DateTimeFormatter` 인스턴스는 한 번 생성하여 static 변수로 공유하거나 여러 스레드에 안전하게 전달할 수 있어, 성능과 안정성 면에서 `SimpleDateFormat`보다 훨씬 뛰어납니다.

## 마무리

이번 포스트에서는 `DateTimeFormatter`의 기본적인 사용법부터 사용자 정의 패턴, 파싱, 그리고 스레드 안전성까지 알아보았습니다. Java 8 이상을 사용한다면, 더 안전하고 강력하며 사용하기 편리한 `DateTimeFormatter`를 사용하는 것이 좋습니다. `SimpleDateFormat`은 레거시 코드와의 호환성을 제외하고는 더 이상 사용할 이유가 없습니다.

---

**참고 자료:**
- [Baeldung - A Guide to java.time.format.DateTimeFormatter](https://www.baeldung.com/java-datetimeformatter)
- [Oracle Java 11 Documentation - DateTimeFormatter](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)
