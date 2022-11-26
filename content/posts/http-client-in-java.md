---
title: "Http Client in Java"
date: 2021-11-02T18:11:57+09:00
url: /http-client-in-java/
tags: ["java"]
---

## 개요

이전까지 자바에서 사용하던 `HttpURLConnection` 는 지원 수준이 너무 낮아 서드 파티 라이브러리인 [Apache HttpClient](https://hc.apache.org/httpcomponents-client-5.1.x/), [Jetty](https://www.eclipse.org/jetty/documentation.php), 스프링의 [RestTemplate](https://www.baeldung.com/rest-template) 을 많이 사용하였다. 하지만 Java 11 에서 HTTP/2와 Web Socket 을 구현하는 HTTP Client API 의 표준화가 정식으로 도입되었다. 이번 포스팅에서는 Java 11 에서 채택된 HTTP Client API 표준화에 대해 알아본다.

## 변경점 (JEP 321)

1. Java 9 에서 도입되었던 HTTP API가 이제 공식적으로 Java SE API에 통합 되었다. 새로운 [HTTP APIs](https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/package-summary.html) 는 `java.net.HTTP.*` 패키지에서 확인할 수 있다.

2. 최신 버전의 HTTP 프로토콜은 stream multiplexing, header compression 와 push promises 등을 통해 전반벅인 성능이 향상되었다.

3. Java 11 부터 API는 비동기로 동작합니다. 비동기는 `CompletableFuture` 를 사용하여 구현되었다.

4. 새로운 HTTP 클라이언트 API는 외부 종속성 없이 HTTP/2와 같은 최신 웹 기능을 지원한다.

5. 새로운 API는 HTTP 1.1/2 WebSocket에 대한 기본 지원을 제공한다. 핵심 기능을 제공하는 클래스와 인터페이스는 다음과 같다.

   * HttpClient class, java.net.http.HttpClient
   * HttpRequest class, java.net.http.HttpRequest
   * HttpResponse<T> interface, java.net.http.HttpResponse
   * WebSocket interface, java.net.http.WebSocket

## 이전 버전의 문제점

기존에 사용하던 HttpURLConnection API 는 다음과 같은 문제가 존재했다:

* 더 이상 작동하지 않는 프로토콜을 사용하도록 디자인 되었다. (FTP, gopher, etc.)
* HTTP/1.1 이전 버전이며 너무 추상적이다.
* blocking mode 로만 동작한다. (하나의 스레드당 하나의 request/response)

## HTTP Client API 개요

`HttpURLConnection` 과 달리 HTTP Client 는 동기화 비동기 모두를 제공한다. API는 다음의 3가지 핵심 클래스로 이루어 진다.

* `HttpRequest` - `HttpClient` 를 통해 보낼 요청
* `HttpClient` - 여러 요청에 대한 공통 구성 정보를 담는 컨테이너 역할
* `HttpResponse` - `HttpRequest` 호출의 결과

다음 섹션에서 각각에 대해 더 자세히 알아보자.

## HttpRequest

이름에서 알 수 있듯 보내려는 요청을 나타내는 객체이다. `HttpRequest.Builder` 를 사용하여 새 인스턴스를 만들 수 있다. `Builder` 클래스는 `HttpRequest.newBuilder()` 를 통해 얻을 수 있다.

### URI 설정

요청을 생성할 때 가장 먼저 해야 할 일은 URL을 제공하는 것이다. 다음과 같이 두 가지 방법을 통해 URL 을 제공 할 수 있다.

```java
HttpRequest.newBuilder(new URI("https://postman-echo.com/get"))
 
HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
```

### HTTP Method 지정

`Builder` 에서 다음 메서드 중 하나를 호출하여 사용할 HTTP Method 를 정의 할 수 있다.

* GET()
* POST(BodyPublisher body)
* PUT(BodyPublisher body)
* DELETE()

`BodyPublisher` 에 대해서는 이후에 다루기로 하고, 먼저 간단한 GET 요청 예제를 살펴보자.

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .GET()
  .build();
```

 이 요청에는 요청에 필요한 모든 정보가 있다. 하지만 때때로 요청에 추가적인 파라미터가 필요할 수도 있다. 몇 가지 중요한 파라미터에는 다음과 같은 것들이 있다.

* HTTP protocol 의 버전
* headers
* timeout

### HTTP protocol 버전

API 는 기본적으로 HTTP/2 프로토콜을 사용하지만 사용하려는 프로토콜의 버전을 명시할 수 있다.

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .version(HttpClient.Version.HTTP_2)
  .GET()
  .build();
```

중요한 점은 HTTP/2 가 지원되지 않는 경우 클라이언트는 HTTP/1.1 로 대체한다는 점이다.

### Header 설정

header 에 추가적인 정보가 필요한 경우 builder 메서드를 사용할 수 있다. 여기에는 두 가지 방법이 있다.

* `headers()` 메서드를 통해 모든 헤더를 키와 값의 쌍으로 제공
* `header()` 메서드를 통해 하나의 키와 값을 제공

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .headers("key1", "value1", "key2", "value2")
  .GET()
  .build();

HttpRequest request2 = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .header("key1", "value1")
  .header("key2", "value2")
  .GET()
  .build();
```

### Timeout 설정

Builder 인스턴스의 `timeout()` 메서드를 사용하여 응답 시간을 설정할 수 있다. 만약 응답 시간을 초과한다면 `HttpTimeoutException` 이 발생한다. 기본 값은 infinity 이다.

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .timeout(Duration.of(10, SECONDS))
  .GET()
  .build();
```

## Request Body 설정

Request Method 가 `POST`, `PUT`, `DELETE` 인 경우 요청 본문을 설정할 수 있다. 새로운 API 는 요청 본문을 작성할 수 있도록 여러가지의 `BodyPublisher` 구현체를 제공한다.

* StringProcessor ( `HttpRequest.BodyPublishers.ofString` 를 사용하여 생성함)
* InputStreamProcessor (`HttpRequest.BodyPublishers.ofInputStream` 를 사용하여 생성함)
* ByteArrayProcessor (`HttpRequest.BodyPublishers.ofByteArray` 를 사용하여 생성함)
* FileProcessor (`HttpRequest.BodyPublishers.ofFile` 를 사용하여 생성함)

request body 가 필요 없는 경우는 `HttpRequest.BodyPublishers.noBody()` 를 사용한다.

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/post"))
  .POST(HttpRequest.BodyPublishers.noBody())
  .build();
```

### StringBodyPublisher

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/post"))
  .headers("Content-Type", "text/plain;charset=UTF-8")
  .POST(HttpRequest.BodyPublishers.ofString("Sample request body"))
  .build();
```

### InputStreamBodyPublisher

```java
byte[] sampleData = "Sample request body".getBytes();
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/post"))
  .headers("Content-Type", "text/plain;charset=UTF-8")
  .POST(HttpRequest.BodyPublishers.ofInputStream(() -> new ByteArrayInputStream(sampleData)))
  .build();
```

### ByteArrayProcessor

```java
byte[] sampleData = "Sample request body".getBytes();
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/post"))
  .headers("Content-Type", "text/plain;charset=UTF-8")
  .POST(HttpRequest.BodyPublishers.ofByteArray(sampleData))
  .build();
```

### FileProcessor

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/post"))
  .headers("Content-Type", "text/plain;charset=UTF-8")
  .POST(HttpRequest.BodyPublishers.fromFile(
    Paths.get("src/test/resources/sample.txt")))
  .build();
```

## HttpClient

모든 요청은 `HttpClient` 를 통해 전송한다. `HttpClient` 는 `HttpClient.newBuilder()` 메서드 또는 `HttpClient.newHttpClient()` 를 통해 얻을 수 있다.

### Response Body 핸들링

Publisher 와 비슷하게 Response Body handler 생성을 위한 메서드가 있다.

```
BodyHandlers.ofByteArray
BodyHandlers.ofString
BodyHandlers.ofFile
BodyHandlers.discarding
BodyHandlers.replacing
BodyHandlers.ofLines
BodyHandlers.fromLineSubscriber
```

`BodyHandlers` 팩토리 클래스의 사용에 주의한다. java 11 이전 버전에서는 다음과 같이 사용했다.

```java
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandler.asString());
```

이제는 다음과 같이 사용할 수 있다.

```java
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
```

### Proxy 설정

Builder 인스턴스에서 `proxy()` 메서드를 사용하여 간단하게 프록시를 추가할 수 있다. 다음은 시스템 기본 프록시를 사용하게 하는 예제이다.

```java
HttpResponse<String> response = HttpClient
  .newBuilder()
  .proxy(ProxySelector.getDefault())
  .build()
  .send(request, BodyHandlers.ofString());
```

### Redirect Polcy 설정

접근하려는 페이지가 다른 주소로 이동하는 경우가 있다. 이 경우 일반적으로 변경된 URI 와 함께 HTTP 상태코드 3xx 를 받게 된다. 적절한 리다이렉션 정책을 설정하면 `HttpClient` 가 자동으로 요청을 새 URI 로 리다이렉션 한다. 리다이렉션 정책 설정은 Builder 인스턴스의 `followRedirects()` 메서드를 사용한다.

```java
HttpResponse<String> response = HttpClient.newBuilder()
  .followRedirects(HttpClient.Redirect.ALWAYS)
  .build()
  .send(request, BodyHandlers.ofString());
```

리다이렉션 정책은 `HttpClient.Redirect` 에 정의 되어 있다.

### Authenticator 설정

`Authenticator` 는 연결을 위한 자격증명을 나타낸다. 예를 들어 연결 하려는 서버가 username, password 를 요구한다면 `PasswordAuthentication` 클래스를 사용할 수 있다.

```java
HttpResponse<String> response = HttpClient.newBuilder()
  .authenticator(new Authenticator() {
    @Override
    protected PasswordAuthentication getPasswordAuthentication() {
      return new PasswordAuthentication(
        "username", 
        "password".toCharArray());
    }
}).build()
  .send(request, BodyHandlers.ofString());
```

### Send Requests - Sync vs. Async

`HttpClient` 는 동기와 비동기 요청 모두 제공한다.

* `send()` - 동기
* `sendAsync()` - 비동기

`send` 메서드는 응답이 올 때까지 기다리고, `HttpResponse` 객체를 리턴한다.

```java
HttpResponse<String> response = HttpClient.newBuilder()
  .build()
  .send(request, BodyHandlers.ofString());
```

응답이 올 때까지 기다리기 때문에 많은 양의 데이터를 처리해야 할 때 단점이 있다. 반면에 `sendAsync` 메서드는 비동기로 작동하며 `CompletableFeature<HttpResponse>` 를 리턴한다.

```java
CompletableFuture<HttpResponse<String>> response = HttpClient.newBuilder()
  .build()
  .sendAsync(request, HttpResponse.BodyHandlers.ofString());
```

다음과 같은 사용도 가능하다.

```java
List<URI> targets = Arrays.asList(
  new URI("https://postman-echo.com/get?foo1=bar1"),
  new URI("https://postman-echo.com/get?foo2=bar2"));
HttpClient client = HttpClient.newHttpClient();
List<CompletableFuture<String>> futures = targets.stream()
  .map(target -> client
    .sendAsync(
      HttpRequest.newBuilder(target).GET().build(),
      HttpResponse.BodyHandlers.ofString())
    .thenApply(response -> response.body()))
  .collect(Collectors.toList());
```

### 비동기를 위한 Executor 설정

비동기 호출 시 사용할 스레드를 제공하는 `Executor` 을 정의할 수도 있다. 이를 사용하여 요청 처리에 사용되는 스레드 수를 제한할 수 있다.

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);

CompletableFuture<HttpResponse<String>> response1 = HttpClient.newBuilder()
  .executor(executorService)
  .build()
  .sendAsync(request, HttpResponse.BodyHandlers.ofString());

CompletableFuture<HttpResponse<String>> response2 = HttpClient.newBuilder()
  .executor(executorService)
  .build()
  .sendAsync(request, HttpResponse.BodyHandlers.ofString());
```

`HttpClient` 는 기본적으로 `java.util.concurrent.Executors.newCachedThreadPool()` 를 사용한다.

### CookieHandler 설정

Builder 의 `cookieHandler` 메서드를 사용하여 클라이언트별 `CookieHandler` 를 손쉽게 설정할 수 있다. 예를 들어 모든 쿠키를 허용하지 않으려면 다음과 같이 사용할 수 있다.

```java
HttpClient.newBuilder()
  .cookieHandler(new CookieManager(null, CookiePolicy.ACCEPT_NONE))
  .build();
```

만약 `CookieManager` 가 쿠키 저장을 허용했다면 `HttpClient` 에서 `CookieHandler` 를 통해 쿠키에 액세스 할 수 있다.

```java
((CookieManager) httpClient.cookieHanlder().get()).getCookieStore()
```

## HttpResponse

`HttpResponse` 클래스는 서버의 응답을 나타낸다. 여러가지 유용한 메서드가 있지만 가장 중요한 것은 두 가지 이다.

* `statusCode()` - HTTP 상태 코드를 반환한다.
* `body()` - 응답에 대한 본문을 반환하며 반환 유형은 `send()` 메서드에 전달된 `BodyHandler` 에 따라 다르다.

이 외에도 `uri()`, `headers()`, `trailers()`, `version()` 등과 같은 유용한 메서드가 있다.

### Response 객체의 URI

Response 객체의 `uri()` 메서드는 응답 된 URI 를 반환한다. 리다이렉션이 일어났을 수도 있기 때문에 request 객체의 URI 와 다른 경우도 있다.

```java
assertThat(request.uri().toString(), equalTo("http://stackoverflow.com"));
assertThat(response.uri().toString(), equalTo("https://stackoverflow.com/"));
```

### Response Headers

Response 객체의 `headers()` 메서드를 통해 응답 헤더를 확인할 수 있다. 응답 헤더는 `HttpHeaders` 이며 read-only 이다.	

```java
HttpResponse<String> response = HttpClient.newHttpClient()
  .send(request, HttpResponse.BodyHandlers.ofString());
HttpHeaders responseHeaders = response.headers();
```

### Response Version

`version()` 메서드를 통해 서버와 통신하는 데 사용된 HTTP 프로토콜의 버전을 알 수 있다. 요청 시 HTTP/2 버전을 사용했더라도 HTTP/1.1 을 통해 응답이 올 수도 있음에 주의한다.

```java
HttpRequest request = HttpRequest.newBuilder()
  .uri(new URI("https://postman-echo.com/get"))
  .version(HttpClient.Version.HTTP_2)
  .GET()
  .build();
HttpResponse<String> response = HttpClient.newHttpClient()
  .send(request, HttpResponse.BodyHandlers.ofString());
assertThat(response.version(), equalTo(HttpClient.Version.HTTP_1_1));
```

 ## 참조

[Exploring the New HTTP Client in Java](https://www.baeldung.com/java-9-http-client)

