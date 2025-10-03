---
title: "Java 11의 새로운 표준 HTTP Client 완벽 정복하기"
description: "Java 11부터 표준으로 채택된 새로운 HTTP Client API(JEP 321)의 모든 것을 알아봅니다. 기존 HttpURLConnection의 한계를 극복하고, HTTP/2와 WebSocket을 지원하며, 동기 및 비동기 방식을 모두 제공하는 현대적인 API 사용법을 상세한 예제와 함께 설명합니다."
date: 2021-11-02T18:11:57+09:00
tags: ["java", "http", "java11", "concurrency"]
---

## 들어가며: 왜 새로운 HTTP Client가 필요했을까요?

오랫동안 자바에서 HTTP 통신을 하기 위해서는 `HttpURLConnection`을 사용해야 했습니다. 하지만 이 API는 다음과 같은 여러 한계를 가지고 있었습니다.

-   **오래된 프로토콜 기반 설계**: FTP, Gopher 등 현재는 거의 사용되지 않는 프로토콜까지 고려하여 너무 추상적으로 설계되었습니다.
-   **블로킹(Blocking) 방식**: 요청(Request)과 응답(Response)이 하나의 스레드를 완전히 점유하는 동기 방식으로만 동작하여 성능에 한계가 있었습니다.
-   **불편한 API**: API 사용법이 직관적이지 않고 장황하여, 대부분의 개발자들은 Apache HttpClient, OkHttp, Spring RestTemplate과 같은 서드파티 라이브러리를 사용하는 것을 선호했습니다.

이러한 문제를 해결하기 위해, Java 9에서 인큐베이터 모듈로 처음 소개되었던 새로운 HTTP Client API가 **Java 11에서 정식 표준(`java.net.http` 패키지)으로 채택** 되었습니다. 이 새로운 API는 HTTP/2와 WebSocket을 기본적으로 지원하며, 동기 및 비동기 프로그래밍 모델을 모두 제공하는 현대적인 클라이언트입니다. 이 글에서는 새로운 HTTP Client API의 핵심 구성요소와 사용법을 자세히 살펴보겠습니다.

## HTTP Client API의 핵심 구성요소

새로운 API는 크게 세 가지 핵심 클래스로 구성됩니다.

1.  **`HttpRequest`**: 보낼 요청에 대한 모든 정보(URI, 헤더, 본문 등)를 담고 있는 불변(immutable) 객체입니다.
2.  **`HttpClient`**: 요청을 보내는 주체로, 요청 전반에 걸친 설정(프로토콜 버전, 프록시, 리다이렉션 정책 등)을 관리하는 컨테이너 역할을 합니다.
3.  **`HttpResponse<T>`**: 요청을 보낸 후 서버로부터 받은 응답(상태 코드, 헤더, 본문 등)을 담고 있는 객체입니다.

## 1. 요청 만들기 (HttpRequest)

`HttpRequest` 객체는 `HttpRequest.Builder`를 사용하여 생성합니다. 빌더 패턴을 사용하므로 메서드 체이닝을 통해 직관적으로 요청을 구성할 수 있습니다.

### URI 및 HTTP 메서드 지정

가장 기본적인 요청은 URI를 지정하고 HTTP 메서드를 호출하는 것입니다.

```java
import java.net.URI;
import java.net.http.HttpRequest;

HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://postman-echo.com/get"))
    .GET() // GET 요청
    .build();
```

### 헤더, 타임아웃, 프로토콜 버전 설정

추가적인 정보가 필요하다면 빌더에 관련 메서드를 호출하여 설정할 수 있습니다.

```java
import java.time.Duration;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;

HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://postman-echo.com/get"))
    .version(HttpClient.Version.HTTP_2) // HTTP/2 사용 명시 (서버가 지원하지 않으면 HTTP/1.1로 자동 전환)
    .header("Accept", "application/json") // 단일 헤더 추가
    .headers("X-Custom-Header1", "Value1", "X-Custom-Header2", "Value2") // 여러 헤더 추가
    .timeout(Duration.ofSeconds(10)) // 타임아웃 설정 (기본값: 무한대)
    .GET()
    .build();
```

### 요청 본문(Body) 설정하기

`POST`나 `PUT` 요청 시에는 본문을 전송해야 합니다. `HttpRequest.BodyPublishers` 팩토리 클래스를 사용하면 다양한 타입의 데이터를 본문으로 쉽게 변환할 수 있습니다.

-   **문자열(String) 본문**: `BodyPublishers.ofString()`
    ```java
    HttpRequest request = HttpRequest.newBuilder()
      .uri(new URI("https://postman-echo.com/post"))
      .headers("Content-Type", "application/json")
      .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"John Doe\"}"))
      .build();
    ```

-   **파일(File) 본문**: `BodyPublishers.ofFile()`
    ```java
    HttpRequest request = HttpRequest.newBuilder()
      .uri(new URI("https://postman-echo.com/post"))
      .headers("Content-Type", "text/plain;charset=UTF-8")
      .POST(HttpRequest.BodyPublishers.ofFile(Paths.get("path/to/file.txt")))
      .build();
    ```

-   **바이트 배열(Byte Array) 본문**: `BodyPublishers.ofByteArray()`
    ```java
    byte[] sampleData = "Sample request body".getBytes();
    HttpRequest request = HttpRequest.newBuilder()
      .uri(new URI("https://postman-echo.com/post"))
      .POST(HttpRequest.BodyPublishers.ofByteArray(sampleData))
      .build();
    ```

이 외에도 `ofInputStream`, `noBody` 등 다양한 `BodyPublisher`가 제공됩니다.

## 2. 요청 보내기 (HttpClient)

`HttpClient`는 `HttpRequest`를 서버로 전송하는 역할을 합니다. `HttpClient` 인스턴스는 한 번 생성하면 여러 요청에 재사용할 수 있습니다.

### 동기(Synchronous) 요청: `send()`

`send()` 메서드는 응답이 올 때까지 현재 스레드를 블로킹(blocking)하고, 작업이 완료되면 `HttpResponse` 객체를 반환합니다. 간단한 요청에 적합합니다.

```java
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;

// 1. HttpClient 생성 (재사용 가능)
HttpClient client = HttpClient.newHttpClient();

// 2. HttpRequest 생성
HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://postman-echo.com/get"))
    .build();

// 3. 동기 방식으로 요청 전송
// BodyHandlers는 응답 본문을 어떻게 처리할지 결정합니다.
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// 4. 응답 처리
System.out.println("Status Code: " + response.statusCode());
System.out.println("Response Body: " + response.body());
```

### 비동기(Asynchronous) 요청: `sendAsync()`

`sendAsync()` 메서드는 요청을 보낸 후 즉시 `CompletableFuture<HttpResponse>`를 반환하고, 실제 HTTP 요청 및 응답 처리는 별도의 스레드에서 수행됩니다. 이를 통해 현재 스레드를 블로킹하지 않고 다른 작업을 계속할 수 있어 높은 처리량이 요구되는 애플리케이션에 매우 유용합니다.

```java
import java.util.concurrent.CompletableFuture;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;

HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://postman-echo.com/get"))
    .build();

// 비동기 방식으로 요청 전송
CompletableFuture<HttpResponse<String>> futureResponse = client.sendAsync(request, HttpResponse.BodyHandlers.ofString());

// 응답이 오면 thenAccept 콜백이 실행됨
futureResponse.thenAccept(response -> {
    System.out.println("Status Code: " + response.statusCode());
    System.out.println("Response Body: " + response.body());
});

// 다른 작업 수행...
futureResponse.join(); // 비동기 작업이 완료될 때까지 대기 (예제이므로 추가)
```

여러 요청을 동시에 보내고 결과를 조합하는 것도 가능합니다.

```java
import java.util.List;
import java.util.stream.Collectors;
import java.util.concurrent.CompletableFuture;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;

List<URI> targets = List.of(
    new URI("https://postman-echo.com/get?foo1=bar1"),
    new URI("https://postman-echo.com/get?foo2=bar2")
);
HttpClient client = HttpClient.newHttpClient();

List<CompletableFuture<String>> futures = targets.stream()
    .map(target -> client.sendAsync(
            HttpRequest.newBuilder(target).GET().build(),
            HttpResponse.BodyHandlers.ofString())
        .thenApply(HttpResponse::body))
    .collect(Collectors.toList());

// 모든 비동기 작업이 완료될 때까지 기다린 후 결과 처리
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

### HttpClient 커스터마이징

`HttpClient.newBuilder()`를 사용하면 리다이렉션 정책, 프록시, 인증자 등 다양한 설정을 커스터마이징할 수 있습니다.

```java
import java.net.ProxySelector;
import java.net.InetSocketAddress;
import java.util.concurrent.Executors;
import java.net.http.HttpClient;

HttpClient customClient = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .followRedirects(HttpClient.Redirect.NORMAL) // 리다이렉션 정책 설정
    .proxy(ProxySelector.of(new InetSocketAddress("proxy.example.com", 80))) // 프록시 설정
    .executor(Executors.newFixedThreadPool(4)) // 비동기 작업을 위한 스레드 풀 지정
    .build();
```

## 3. 응답 다루기 (HttpResponse)

`HttpResponse` 객체를 통해 서버의 응답 정보를 얻을 수 있습니다.

-   `statusCode()`: `200`, `404`와 같은 HTTP 상태 코드를 반환합니다.
-   `body()`: 응답 본문을 반환합니다. 이 반환 타입은 `send()` 또는 `sendAsync()`에 전달한 `HttpResponse.BodyHandlers`에 의해 결정됩니다. (예: `BodyHandlers.ofString()` -> `String`, `BodyHandlers.ofByteArray()` -> `byte[]`)
-   `headers()`: 응답 헤더 정보를 `HttpHeaders` 객체로 반환합니다.
-   `version()`: 실제 통신에 사용된 HTTP 프로토콜 버전을 반환합니다.

```java
import java.net.http.HttpResponse;
import java.net.http.HttpHeaders;
import java.util.Optional;

// 이전에 client.send()로 얻은 response 객체가 있다고 가정
HttpResponse<String> response = ...;

int statusCode = response.statusCode();
String body = response.body();
HttpHeaders headers = response.headers();
Optional<String> contentType = headers.firstValue("Content-Type");
```

## 마무리

Java 11의 새로운 HTTP Client는 최신 웹 환경에 필수적인 HTTP/2와 논블로킹 I/O를 지원하는 강력하고 현대적인 API입니다. 더 이상 무거운 외부 라이브러리에 의존하지 않고도, 자바 표준 라이브러리만으로 효율적이고 안정적인 HTTP 통신을 구현할 수 있게 되었습니다. 특히 `CompletableFuture`와 결합된 비동기 API는 높은 동시성 처리가 필요한 애플리케이션의 성능을 극대화하는 데 큰 도움이 될 것입니다.

---

**참고 자료:**
- [JEP 321: HTTP Client (Standard)](https://openjdk.java.net/jeps/321)
- [Baeldung - Exploring the New HTTP Client in Java](https://www.baeldung.com/java-9-http-client)