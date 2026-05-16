---
title: "Java/Spring URI 처리: RFC 표준 차이와 실무 함정"
description: "URI 표준(RFC 2396 vs RFC 3986)의 차이, Java URLEncoder와 JavaScript encodeURIComponent의 인코딩 방식, Spring UriComponentsBuilder 사용 시 생기는 문제를 정리합니다."
date: 2026-01-15T19:41:00+09:00
tags: ["java", "javascript", "url-encoding", "uri", "spring"]
---

웹 개발에서 URI를 다루다 보면 생각보다 낯선 버그를 만납니다. 프론트엔드에서 보낸 `"hello world"`가 백엔드에서는 `"hello+world"`로 찍히거나, `user_profile`이라는 host가 어느 순간 사라지는 식입니다. 이런 문제는 대개 **URI 표준이 바뀌어 온 과정**과 맞닿아 있습니다.

이 글에서는 Java와 Spring에서 URI를 다룰 때 자주 밟는 함정을 정리하고, 실무에서 어떤 방식을 쓰는 편이 안전한지 살펴봅니다.

## 📝 목차

1. [URI 표준의 역사: RFC 2396 vs RFC 3986](#-uri-표준의-역사-rfc-2396-vs-rfc-3986)
2. [함정 1: Java vs JavaScript 인코딩 차이](#-함정-1-java-vs-javascript-인코딩-차이)
3. [함정 2: java.net.URI의 hostname 제약](#-함정-2-javaneturi의-hostname-제약)
4. [함정 3: UriComponentsBuilder.fromUri()의 정보 누락](#-함정-3-uricomponentsbuilderfromuri의-정보-누락)
5. [실무에서 쓸 만한 해결 방법](#-실무에서-쓸-만한-해결-방법)
6. [인코딩/디코딩 시 주의사항](#-인코딩디코딩-시-주의사항)

---

## 📜 URI 표준의 역사: RFC 2396 vs RFC 3986

URI 처리에서 발생하는 대부분의 혼란은 **두 개의 RFC 표준** 사이의 차이에서 비롯됩니다.

| 표준 | 연도 | 상태 | 특징 |
|:---|:---:|:---:|:---|
| **RFC 2396** | 1998 | ❌ 폐지됨 | 최초의 URI 표준. hostname에 `_` 불허, 공백을 `+`로 인코딩 |
| **RFC 3986** | 2005 | ✅ 현재 표준 | RFC 2396을 대체. hostname 규칙 완화, 공백을 `%20`으로 인코딩 |

문제는 **Java의 핵심 클래스 중 일부가 여전히 1998년 표준을 기준으로 동작**한다는 점입니다.

| 클래스/함수 | 따르는 표준 | 비고 |
|:---|:---:|:---|
| `java.net.URLEncoder` | RFC 2396 + Form 스펙 | 공백 → `+` |
| `java.net.URI` | RFC 2396 | hostname에 `_` 불허 |
| `encodeURIComponent` (JS) | RFC 3986 | 공백 → `%20` |
| `UriComponentsBuilder.fromUriString()` | RFC 3986 | ✅ 권장 |
| `UriComponentsBuilder.fromUri(URI)` | 혼합 | ⚠️ 주의 필요 |

이 차이가 실제 코드에서 어떤 문제로 이어지는지 하나씩 보겠습니다.

---

## ⚠️ 함정 1: Java vs JavaScript 인코딩 차이

### 문제 상황

```java
// Java
String encoded = URLEncoder.encode("hello world", StandardCharsets.UTF_8);
System.out.println(encoded); // 출력: hello+world
```

```javascript
// JavaScript
const encoded = encodeURIComponent("hello world");
console.log(encoded); // 출력: hello%20world
```

같은 문자열인데 결과가 다릅니다:
- **Java**: `hello+world` (공백 → `+`)
- **JavaScript**: `hello%20world` (공백 → `%20`)

### 원인: 설계 목적의 차이

**Java URLEncoder**는 **HTML Form 전송용** (`application/x-www-form-urlencoded`)으로 설계되었습니다:
- 공백을 **`+`** 로 인코딩
- `*`, `-`, `_`, `.`는 인코딩하지 않음
- 그 외 모든 문자는 `%XX` 형태로 인코딩

**JavaScript encodeURIComponent**는 **URI 컴포넌트용** (RFC 3986)으로 설계되었습니다:
- 공백을 **`%20`** 으로 인코딩
- `*`, `-`, `_`, `.`, `!`, `~`, `'`, `(`, `)`는 인코딩하지 않음

### 동작이 달라지는 문자들

| 문자 | Java URLEncoder | JavaScript encodeURIComponent |
|:---:|:---:|:---:|
| ` ` (공백) | `+` | `%20` |
| `~` | `%7E` | `~` |
| `!` | `%21` | `!` |
| `'` | `%27` | `'` |
| `(` | `%28` | `(` |
| `)` | `%29` | `)` |

### 실무 문제: 캐시 키 불일치

```java
// Backend - 캐시 키 생성
String cacheKey = "search:" + URLEncoder.encode("안녕 하세요", UTF_8);
// 결과: search:안녕+하세요
```

```javascript
// Frontend - 캐시 조회 요청
const cacheKey = "search:" + encodeURIComponent("안녕 하세요");
// 결과: search:안녕%20하세요
```

같은 검색어인데 캐시 키가 달라져 **캐시 미스**가 발생합니다.

---

## ⚠️ 함정 2: java.net.URI의 hostname 제약

### 문제 상황

```java
URI uri = new URI("myapp://user_profile?version=1.0");
System.out.println(uri.getHost()); // 출력: null ❌
```

`user_profile`이 host인데 왜 `null`이 반환될까요?

### 원인: RFC 2396의 hostname 규칙

`java.net.URI`는 [RFC 2396 Section 3.2.2](https://www.rfc-editor.org/rfc/rfc2396#section-3.2.2)를 따릅니다. 이 규칙에 따르면 hostname은:

> a sequence of domain labels separated by ".", each domain label starting and ending with an **alphanumeric character** and possibly also containing **"-"** characters.

즉, hostname에는 언더스코어(`_`)가 허용되지 않습니다.

```java
// ✅ 정상: 언더스코어 없음
new URI("myapp://userprofile?version=1.0").getHost();   // "userprofile"

// ❌ 문제: 언더스코어 포함
new URI("myapp://user_profile?version=1.0").getHost();  // null
```

> [!NOTE]
> **RFC 3986에서는 언더스코어가 허용됩니다.**
> 
> RFC 3986은 `hostname`, `domainlabel` 등의 제약 규칙을 **삭제**했습니다. 따라서 `user_profile`은 현재 표준에서는 완전히 유효한 host입니다. `java.net.URI`가 폐지된 1998년 표준을 따르고 있는 것이 문제입니다.

### java.net.URI의 authority vs host

`java.net.URI`는 정보를 버리지 않습니다. 단지 다른 변수에 저장할 뿐입니다:

```java
URI uri = new URI("myapp://user_profile?version=1.0");
uri.getHost();       // null
uri.getAuthority();  // "user_profile" ✅
uri.toString();      // "myapp://user_profile?version=1.0" ✅
```

`user_profile` 정보 자체가 사라진 것은 아닙니다. `authority`에 들어가 있습니다. 문제는 이 값을 다른 클래스와 함께 사용할 때 생깁니다.

---

## ⚠️ 함정 3: UriComponentsBuilder.fromUri()의 정보 누락

### 문제 상황

`java.net.URI`와 `UriComponentsBuilder`를 함께 사용할 때 발생하는 문제입니다:

```kotlin
// URL에서 특정 파라미터를 제거하는 확장함수
private fun String.removeDebugParameter(): String = 
    UriComponentsBuilder.fromUri(URI(this))  // ⚠️ 여기가 문제!
        .replaceQueryParam("debug", null)
        .build()
        .toUriString()
```

```kotlin
// 테스트
"myapp://home?debug=true&version=1.0".removeDebugParameter()
// 기대: myapp://home?version=1.0
// 실제: myapp://home?version=1.0 ✅

"myapp://user_profile?debug=true&version=1.0".removeDebugParameter()
// 기대: myapp://user_profile?version=1.0
// 실제: myapp://?version=1.0 ❌ (host가 사라짐!)
```

### 원인 분석

1. **`URI("myapp://user_profile?...")`** → `host`가 `null`, `authority`에 `user_profile` 저장
2. **`UriComponentsBuilder.fromUri(uri)`** → `uri.getHost()`만 참조, `authority`는 무시
3. **결과** → host 정보 누락!

```java
// UriComponentsBuilder.uri() 메서드 내부
public UriComponentsBuilder uri(URI uri) {
    this.host = uri.getHost();  // null이 반환됨!
    // uri.getAuthority()는 참조하지 않음
    ...
}
```

`UriComponentsBuilder`는 RFC 3986을 따르지만, `authority`를 별도로 복원해주지는 않습니다. 그래서 `java.net.URI`를 거쳐 들어오면 일부 정보가 빠질 수 있습니다.

---

## ✅ 실무에서 쓸 만한 해결 방법

### 1. JavaScript와 동일한 인코딩 (RFC 3986)

```java
public class URIEncoder {
    
    /**
     * JavaScript의 encodeURIComponent와 동일하게 동작하는 인코딩.
     */
    public static String encodeURIComponent(String value) {
        if (value == null) return null;
        
        return URLEncoder.encode(value, StandardCharsets.UTF_8)
                .replace("+", "%20")   // 공백
                .replace("%7E", "~")   // 틸드
                .replace("%21", "!")   // 느낌표
                .replace("%27", "'")   // 작은따옴표
                .replace("%28", "(")   // 여는 괄호
                .replace("%29", ")");  // 닫는 괄호
    }
}
```

> [!TIP]
> 공백 처리만 맞추면 되는 경우라면 간단하게:
> ```java
> URLEncoder.encode(value, UTF_8).replace("+", "%20");
> ```

### 2. Spring UriComponentsBuilder 올바른 사용법

```java
// ❌ 위험: java.net.URI를 경유하면 정보 누락 가능
UriComponentsBuilder.fromUri(new URI(urlString))  

// ✅ 권장: 문자열을 직접 파싱 (RFC 3986 정규식 사용)
UriComponentsBuilder.fromUriString(urlString)
```

`fromUriString()`은 문자열을 RFC 3986 기반 정규식으로 직접 파싱하므로, `java.net.URI`의 hostname 제약을 피할 수 있습니다.

```java
// 쿼리 파라미터 추가
String url = UriComponentsBuilder.fromUriString("https://api.example.com/search")
        .queryParam("q", "hello world")        // 자동으로 %20 인코딩
        .queryParam("name", "홍길동")           // 한글 자동 인코딩
        .build()
        .toUriString();
// 결과: https://api.example.com/search?q=hello%20world&name=%ED%99%8D%EA%B8%B8%EB%8F%99
```

### 3. java.net.URI 사용 시 주의사항

`java.net.URI`를 반드시 사용해야 한다면, `getHost()` 대신 `getAuthority()`를 확인하세요:

```java
URI uri = new URI("myapp://user_profile?version=1.0");

// 방어적 코딩
String host = uri.getHost();
if (host == null) {
    host = uri.getAuthority();  // fallback
}
```

### 4. JavaScript에서 Form 인코딩이 필요한 경우

```javascript
// 방법 1: 수동 치환
function encodeFormData(value) {
    return encodeURIComponent(value).replace(/%20/g, '+');
}

// 방법 2: URLSearchParams 사용
const params = new URLSearchParams();
params.append('message', 'hello world');
console.log(params.toString()); // message=hello+world
```

### 5. 선택 가이드

| 상황 | 권장 방법 |
|:---|:---|
| **쿼리 파라미터 구성** | `UriComponentsBuilder.fromUriString()` + `queryParam()` |
| **URL 문자열 직접 구성** | `URIEncoder.encodeURIComponent()` 유틸리티 |
| **Form 데이터 전송** | `URLEncoder.encode()` 그대로 사용 |
| **프론트-백엔드 일관성** | 양쪽 모두 RFC 3986 방식으로 통일 |

---

## ⚠️ 인코딩/디코딩 시 주의사항

### 더블 인코딩 방지

이미 인코딩된 문자열을 다시 인코딩하면 `%`가 `%25`로 변환되어 문제가 발생합니다:

```java
// ❌ 잘못된 예: 이미 인코딩된 문자열을 다시 인코딩
String alreadyEncoded = "hello%20world";
URLEncoder.encode(alreadyEncoded, UTF_8); // "hello%2520world" (% → %25)
```

### 예외: 의도된 더블 인코딩 (Nested URL)

URL 안에 URL을 파라미터로 넣는 경우에는 **의도적인 더블 인코딩이 필수**입니다:

```java
// 딥링크 생성 예시
String innerUrl = "https://site.com/search?q=" + URIEncoder.encodeURIComponent("hello world");
// 결과: "https://site.com/search?q=hello%20world"

String deepLink = "myapp://webview?link=" + URIEncoder.encodeURIComponent(innerUrl);
// 결과: "myapp://webview?link=https%3A%2F%2Fsite.com%2Fsearch%3Fq%3Dhello%2520world"
//                                                               ↑ %20 → %2520
```

| 구분 | 설명 |
|:---|:---|
| ❌ 실수로 인한 중복 인코딩 | 같은 레벨에서 모르고 다시 인코딩 → **버그** |
| ✅ 구조적 중첩 인코딩 | inner/outer 각 레벨에서 의도적으로 인코딩 → **정상** |

### 디코딩 동작 차이

인코딩과 마찬가지로 디코딩도 Java와 JavaScript에서 다르게 동작합니다:

```java
// Java URLDecoder: '+'를 공백으로 해석
URLDecoder.decode("hello+world", UTF_8);   // "hello world" ✅
URLDecoder.decode("hello%20world", UTF_8); // "hello world" ✅
```

```javascript
// JavaScript decodeURIComponent: '+'를 그대로 유지
decodeURIComponent("hello+world");   // "hello+world" ⚠️
decodeURIComponent("hello%20world"); // "hello world" ✅
```

> [!WARNING]
> JavaScript에서 Form 인코딩된 데이터(`+`가 공백인 경우)를 디코딩하려면:
> ```javascript
> function decodeFormData(value) {
>     return decodeURIComponent(value.replace(/\+/g, '%20'));
> }
> ```

---

## 🎯 요약

| 함정 | 원인 | 해결책 |
|:---|:---|:---|
| Java/JS 인코딩 불일치 | URLEncoder는 RFC 2396 + Form 스펙 | `URIEncoder.encodeURIComponent()` 유틸리티 |
| `URI.getHost()` null 반환 | RFC 2396은 hostname에 `_` 불허 | `getAuthority()` fallback 또는 회피 |
| `fromUri()` 정보 누락 | `UriComponentsBuilder`가 `authority` 미참조 | `fromUriString()` 사용 |

정리하면, Java의 URI 관련 클래스 중 일부는 1998년 표준(RFC 2396)의 영향을 아직 받습니다. 현대 웹 개발에서는 가능한 한 RFC 3986 기준으로 파싱하고 인코딩하는 도구를 쓰는 편이 안전합니다.

```java
// java.net.URI를 거치지 않는다
UriComponentsBuilder.fromUriString(url)
        .queryParam("key", "value")
        .build()
        .toUriString();
```

---

*참고 자료:*
- [RFC 3986 - URI 문법](https://www.rfc-editor.org/rfc/rfc3986) (현재 표준)
- [RFC 2396 - URI 문법](https://www.rfc-editor.org/rfc/rfc2396) (구버전, Obsolete)
- [Java URLEncoder 문서](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/URLEncoder.html)
- [Spring UriComponentsBuilder 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/util/UriComponentsBuilder.html)
- [MDN encodeURIComponent](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent)
- [카카오페이 기술 블로그 - URL이 이상해요!](https://tech.kakaopay.com/post/url-is-strange/)

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
