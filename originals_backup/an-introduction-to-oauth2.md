---
title: "Oauth2 에 대해서 알아보자"
description: "OAuth 2.0 의 클라이언트 구현에 도움이 될 만한 정보를 위주로 OAuth 2.0 에 대하여 알아보자."
date: 2021-10-22T10:08:30+09:00
tags: ["development"]
---

## 소개

OAuth는 오픈 API의 인증(authentication)과 권한 부여(authorization)를 제공하기 위해 만들어진 프로토콜이다. OAuth 1.0과 OAuth 2.0이 있는데, 현재는 [RFC 5849](https://datatracker.ietf.org/doc/html/rfc5849)에서 설명하는 OAuth 1.0을 폐기하고, [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)에 설명된 OAuth 2.0 방식을 사용한다. 이번 포스팅에서는 OAuth 2.0에 관하여 알아본다.

## 역할 (Rules)

OAuth 2.0을 이해하기 위해서는 먼저 OAuth 2.0에서 정의하는 4가지 역할에 관하여 알아야한다.

1. Resource Owner (리소스 소유자)
2. Resource Service (리소스 서버)
3. Client (클라이언트)
4. Authorization Server (인증 서버)

**리소스 소유자**는 보호된 리소스의 소유자를 말한다. 예를 들어 은행관련 서비스에서 계좌 잔액이라는 리소스가 있다면, 이 계좌의 소유주(예금주)가 리소스의 소유자가 된다.

**리소스 서버**는 보호된 리소스를 제공하는 서버를 말한다. 예를 들어 [오픈뱅킹](https://www.openbanking.or.kr/) 서비스는 각 은행들의 API를 연동하여 다양한 리소스(거래내역, 계좌실명 등)를 제공한다. 이런 경우 오픈뱅킹의 서버가 리소스 서버라고 할 수 있다.

**클라이언트**는 오픈 API를 호출하는 응용 프로그램을 말한다. 예를 들어 오픈뱅킹 API를 이용하여 모든 은행들의 잔액을 볼 수 있는 어플리케이션을 만들었다면, 이 어플리케이션이 클라이언트가 된다.

**인증 서버**는 리소스 소유자로부터 리소스에 접근 권한을 획득한 이후에 리소스에 접근하기 위한 엑세스 토큰(Access Token)을 발급해주는 서버를 말한다.

## OAuth 2.0의 흐름

다음 그림은 OAuth 2.0의 대략적인 흐름을 나타낸다.

```
+--------+                               +---------------+
|        |--(1)- Authorization Request ->|   Resource    |
|        |                               |     Owner     |
|        |<-(2)-- Authorization Grant ---|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(3)-- Authorization Grant -->| Authorization |
| Client |                               |     Server    |
|        |<-(4)----- Access Token -------|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(5)----- Access Token ------>|    Resource   |
|        |                               |     Server    |
|        |<-(6)--- Protected Resource ---|               |
+--------+                               +---------------+
```

그림을 보고 하나씩 짚어보자.

1. 리소스 서버에게 리소스를 요청하기 전에 먼저 인증을 요청한다. 인증 요청은 리소스 소유자에게 직접 할 수도 있지만, 중간에 인증 서버를 통해 간접적으로 하는 것이 좋다.
2. 클라이언트는 리소스 소유자의 인가를 나타내는 자격정보인 [인가 승인](#인가-승인-authorization-grant)을 받는다.
3. 클라이언트는 2번에서 받은 인가 승인을 사용하여 엑세스 토큰을 요청한다.
4. 인증 서버는 클라이언트를 인증하고, 제시된 인가 승인이 유효한지 확인한다. 유효한 경우 엑세스 토큰을 발급한다.
5. 클라이언트는 엑세스 토큰을 제시하여 리소스 서버에 보호된 리소스를 요청한다.
6. 리소스 서버는 엑세스 토큰이 유효한지 확인하고, 유효한 경우 요청을 받아들인다.

## 인가 승인 (Authorization Grant)

"인가 승인"은 리소스 소유자가 보호된 리소스에 대한 접근을 허용한다는 것을 나타내는 자격 정보(Credentials)이다. 이는 클라이언트가 엑세스 토큰을 얻기 위해 사용된다. 인가 승인에는 네 가지 유형이 있다. 이번 포스팅에서는 네 가지 유형 중 인가 코드(authorization code) 방식에 대해서만 설명하며, 추가적인 유형은 [RFC 6749 - Authorization Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-1.3)를 참고한다.

### 인가 코드 (Authorization Code)

클라이언트는 사용자 에이전트(User-Agent)를 통해 리소스 소유자를 인증 서버로 안내한다. 인증 서버는 리소스 소유자를 인증하고, 인증이 완료되면 클라이언트는 인증 코드를 획득한다. 네이버 로그인이나 카카오 로그인 API가 사용하는 인가 승인 유형이 바로 이 유형이다.

## 엑세스 토큰 (Access Token)

엑세스 토큰은 보호된 리소스에 접근할 수 있도록 하는 권한 증명이다. 엑세스 토큰으로 접근할 수 있는 리소스의 범위와 사용할 수 있는 기간이 정해져 있다.

## 리프레시 토큰 (Refresh Token)

리프레시 토큰은 엑세스 토큰을 얻는 데 사용할 수 있는 권한 증명이다. 엑세스 토큰이 유효하지 않거나, 만료된 경우 새로운 엑세스 토큰을 발급 받기 위해 사용된다. 더 좁은 범위로 추가적인 엑세스 토큰을 받기 위해 사용되기도 한다. 아래의  그림은 리프레시 토큰을 사용하여 엑세스 토큰을 갱신하는 흐름을 보여준다.

```
+--------+                                           +---------------+
|        |--(1)------- Authorization Grant --------->|               |
|        |                                           |               |
|        |<-(2)----------- Access Token -------------|               |
|        |               & Refresh Token             |               |
|        |                                           |               |
|        |                            +----------+   |               |
|        |--(3)---- Access Token ---->|          |   |               |
|        |                            |          |   |               |
|        |<-(4)- Protected Resource --| Resource |   | Authorization |
| Client |                            |  Server  |   |     Server    |
|        |--(5)---- Access Token ---->|          |   |               |
|        |                            |          |   |               |
|        |<-(6)- Invalid Token Error -|          |   |               |
|        |                            +----------+   |               |
|        |                                           |               |
|        |--(7)----------- Refresh Token ----------->|               |
|        |                                           |               |
|        |<-(8)----------- Access Token -------------|               |
```

## 인가 획득 (Obtaining Authorization)

클라이언트가 리소스 소유자로부터 인가를 획득하는 과정에 대해 알아보자. [인가 승인](#인가-승인-authorization-grant)에서 알아본 것처럼 인가 획득을 위한 유형에는 네 가지가 있지만, 이 포스팅에서는 [인가 코드](#인가-코드-authorization-code)를 사용하여 인가를 획득하는 과정에 대해서만 설명한다.

인가 코드 승인 유형은 리다이렉션이 기반이 된다. 때문에 클라이언트는 리소스 소유자의 유저 에이전트(일반적으로 웝 브라우저)와 상호작용 할 수 있어야하며, 리다이렉션을 통해 인증 서버로 오는 요청을 받을수 있어야 한다.

### 인가 코드 승인 유형 흐름

다음 그림은 인가 코드 방식의 흐름을 보여준다.

```
+----------+
| Resource |
|   Owner  |
|          |
+----------+
     ^
     |
    (2)
+----|-----+          Client Identifier      +---------------+
|         -+----(1)-- & Redirection URI ---->|               |
|  User-   |                                 | Authorization |
|  Agent  -+----(2)-- User authenticates --->|     Server    |
|          |                                 |               |
|         -+----(3)-- Authorization Code ---<|               |
+-|----|---+                                 +---------------+
  |    |                                         ^      v
 (1)  (3)                                        |      |
  |    |                                         |      |
  ^    v                                         |      |
+---------+                                      |      |
|         |>---(4)-- Authorization Code ---------'      |
|  Client |          & Redirection URI                  |
|         |                                             |
|         |<---(5)----- Access Token -------------------'
+---------+       (w/ Optional Refresh Token)
```

1,2,3번의 과정이 두 부분으로 나누어 지는데, 이는 유저 에이전트를 통해 전달되기 때문이다.

1번은 클라이언트가 리소스 소유자의 유저 에이전트를 인증 서버로 안내하며 흐름을 시작하는 과정이다. 클라이언트는 클라이언트의 식별자(client identifier), 요청 범위(requested scope), 로컬 상태(local state), 리다이렉션 URI를 포함해야한다. 네이버 아이디로 로그인 API를 연동한 어플리케이션에서 [네이버 아이디로 로그인] 버튼을 누르면 흐름이 시작되는데, 이 과정이라고 생각하면 된다.

2번은 인증 서버가 유저 에이전트를 통해 리소스 소유자를 인증하고, 리소스 소유자가 클라이언트의 접근 요청에 승인 혹은 거부할 지를 선택하는 단계이다.

![](/images/naver_login.png)

3번 과정은 인증 서버가 유저 에이전트를 제공된 리다이렉션 URI로 이동시키는 과정이다. 이때 리다이렉션 URI에는 인증 코드와 로컬 상태가 포함된다.

4번은 클라이언트가 이전 과정에서 받은 인증 코드를 포함하여 인증 서버에 엑세스 토큰을 요청하는 단계이다. 요청을 보낼 때에는 3번 과정에서 사용했던 리다이렉션 URI도 포함하여 전달한다.

5번 과정에서 인증서버는 인증 코드가 유효한지 확인하고, 3번 과정과 4번 과정의 리다이렉션 URI가 동일한지 확인한다. 유효하다는 게 획인되면 엑세스 토큰과 (선택적으로) 리프레시 토큰을 응답한다.

### 인가 요청 (Authorization Request)

클라이언트는 인증 코드 요청을 할 때 `application/x-www-form-urlencoded` 를 사용하여 다음 파라미터를 포함해야한다.

* `response_type`  
  **필수.** "code"로 고정
* `client_id`  
  **필수.** 클라이언트의 식별자
* `redirect_uri`  
  **선택사항.** 자세한 내용은 [RFC 6749 - Redirection Endpoint](https://datatracker.ietf.org/doc/html/rfc6749#section-3.1.2) 참고
* `scope`  
  **선택사항.** 자세한 내용은 [RFC 6749 - Access Token Scope](https://datatracker.ietf.org/doc/html/rfc6749#section-3.3) 참고
* `state`  
  **권장사항.** 인증 서버는 유저 에이전트를 클라이언트로 리다이렉트할 때 이 값을 포함한다. [RFC 6749 - Cross-Site Request Forgery](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12)에 기술된대로 사이트 간 요청 위조를 방지하는 데 사용하는 것이 좋다.

인가 요청의 예:

```
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com
```

### 인가 응답 (Authorization Response)

리소스 소유자가 접근 요청을 승인하면, 인증 서버는 다음의 파라미터를 `application/x-www-form-urlencoded` 를 사용하여 클라이언트에게 전달한다.

#### 성공 응답 (Successful Response)

* `code`  
  **필수.** 인증 서버에서 생성된 인증 코드이다. 유출 위험을 줄이기 위해 만료 시간이 짧아야 하며 최대 10분이 권장된다. 클라이언트는 인증 코드를 두 번 이상 사용하면 안된다. 만약 두번 이상 사용될 경우 인증 서버는 요청을 거부해야 하며, 해당 인증 코드 이전에 발급된 모든 토큰을 취소하는 것이 좋다.
* `state`  
  클라이언트가 인증 요청 시 `state` 파라미터를 포함했다면 **필수.** 클라이언트로부터 전달 받은 값과 동일해야한다.

성공 응답의 예:

```
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```

#### 오류 응답 (Error Response)

만약 요청이 누락되거나, 유효하지 않는 경우, 리다이렉션 URI가 일치하지 않는 경우, 클라이언트 식별자가 누락되거나 유효하지 않는 경우는 유저 에이전트를 유효하지 않은 URI로 리다이렉트 되게 해서는 안된다. 이 경우에 인증 서버는 다음 파라미터를 사용하여 리소스 소유자에게 오류를 알려주는 것이 좋다.

* `error`  
  **필수.** 다음의 에러 코드 사용
    * `invalid_request`  
      요청 시 필수 파라미터의 누락, 유효하지 않은 파라미터 포함, 파라미터를 두 번 이상 포함 등
    * `unauthorized_client`  
      클라이언트가 이 인가 승인 유형을 사용할 권한이 없음
    * `access_denied`  
      리소스 소유자 또는 인증 서버가 요청을 거부
    * `unsupported_response_type`  
      인증 서버가 Authorization Code Grant 유형을 지원하지 않음
    * `invalid_scope`  
      요청한 scope가 유효하지 않거나, 알 수 없거나, 손상된 경우
    * `server_error`  
      인증 서버에 예기치 못한 오류가 발생
    * `temporarily_unavailable`  
      인증 서버의 과부하 또는 유지보수로 인해 요청을 처리할 수 없음
* `error_description`  
  **선택사항.** 클라이언트 개발자가 발생한 오류를 이해하는 데 도움을 주는 정보를 제공한다.
* `error_uri`  
  **선택사항.** 클라이언트 개발자를 위해 발생한 오류와 관련된 추가 정보를 제공한다.
* `state`  
  클라이언트가 인증 요청 시 `state` 파라미터를 포함했다면 **필수.** 클라이언트로부터 전달 받은 값과 동일해야한다.

오류 응답의 예:

```
HTTP/1.1 302 Found
Location: https://client.example.com/cb?error=access_denied&state=xyz
```

### 엑세스 토큰 요청 (Access Token Request)

클라이언트는 토큰 요청시 UTF-8 인코딩을 사용하여 `application/x-www-form-urlencoded` 형식으로 된 다음과 같은 파라미터를 body에 담아야한다.

* `grant_type`  
  **필수.** 값은 "authorization_code"로 고정
* `code`  
  **필수.** 인증 서버로부터 받은 인증 코드
* `redirect_uri`  
  인증 요청 시 `redirect_uri`가 존재했다면 **필수**이며, 인증 요청 시 사용했던 값과 동일해야 한다.
* `client_id`  
  클라이언트가 인증 서버와 인증하지 않는 경우 **필수.** 클라이언트 인증에 관한 자세한 내용은 [RFC 6749 - Client Authentication](https://datatracker.ietf.org/doc/html/rfc6749#section-3.2.1)를 참고한다.

토큰 요청의 예:

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencodedgrant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

### 엑세스 토큰 응답 (Access Token Response)

Access Token 요청이 유효하고 인증되었다면, 인증 서버는 Access Token과 선택적으로 갱신 토큰을 발급한다.

#### 성공 응답 (Successful Response)

인증 서버는 Access Token과 선택적으로 갱신 토큰을 발급하고, 다음 파라미터를 200 (OK) 상태 코드로 응답한다. 파라미터는 `application/json` 유형을 사용하여 HTTP Response Body에 포함한다.

* `access_token`  
  **필수.** 인증 서버가 발급한 엑세스 토큰
* `token_type`  
  **필수.** 토큰의 타입으로 보통 `baerer` 타입을 많이 사용한다. 자세한 내용은  [RFC 6749 - Access Token Types](https://datatracker.ietf.org/doc/html/rfc6749#section-7.1)를 참고한다.
* `expires_in`  
  **권장사항.** 초 단위의 Access Token 수명. 예를 들어 값이 3600이라면 토큰이 생성된 시간으로부터 3600초(1 시간) 뒤에 만료된다는 의미이다.
* `refresh_token`  
  **선택사항.** 인증 서버가 발급한 리프레시 토큰
* `scope`  
  클라이언트가 요청한 범위와 동일하다면 **선택사항.** 그렇지 않은 경우 **필수.** 자세한 내용은 [RFC 6749 - Access Token Scope](https://datatracker.ietf.org/doc/html/rfc6749#section-3.3)를 참고한다.

성공 응답의 예:

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
    "example_parameter":"example_value"
}
```

#### 오류 응답 (Error Response)

인증 서버는 다음 파라미터를 400 (Bad Request) 상태 코드로 응답한다. 파라미터는 `application/json` 유형을 사용하여 HTTP Response Body에 포함한다.

* `error`  
  **필수.** 다음의 에러 코드 사용
    * `invalid_request`  
      요청 시 필수 파라미터의 누락, 유효하지 않은 파라미터 포함, 파라미터를 두 번 이상 포함 등
    * `invalid_client`  
      클라이언트가 인증에 실패한 경우 (e.g. 알 수 없는 클라이언트, 클라이언트 인증이 포함되지 않음, 지원되지 않는 인증 방법).
    * `invalid_grant`  
      인가 승인 유형 또는 Refresh Token이 유효하지 않거나, 만료, 취소된 경우 또는 인증 요청에 사용된 리다이렉션 URI가 일치하지 않거나 다른 클라이언트에게 발급된 경우
    * `unauthorized_client`  
      클라이언트가 이 인가 승인 유형을 사용할 권한이 없음
    * `access_denied`  
      리소스 소유자 또는 인증 서버가 요청을 거부
    * `unsupported_grant_type`  
      서버가 지원하지 않는 인가 승인 유형인 경우
    * `temporarily_unavailable`  
      인증 서버의 과부하 또는 유지보수로 인해 요청을 처리할 수 없음 (503 Service Unavailable 상태 코드는 HTTP redirect를 통해 클라이언트에게 전달 될 수 없기 때문에 이 오류 코드가 필요)
    * `invalid_scope`  
      요청한 scope가 유효하지 않거나, 알 수 없거나, 손상된 경우
* `error_description`  
  **선택사항.** 클라이언트 개발자가 발생한 오류를 이해하는 데 도움을 주는 정보를 제공한다.
* `error_uri`  
  **선택사항.** 클라이언트 개발자를 위해 발생한 오류와 관련된 추가 정보를 제공한다.

오류 응답의 예:

```
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "error":"invalid_request"
}
```

## 사용 예

카카오와 네이버에서도 OAuth 2.0 방식의 로그인 API를 제공한다.

### 카카오

[카카오 로그인](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api) 문서를 확인해보면, 카카오 로그인을 위해서 인가 코드 받기, 토큰 받기 두 과정을 거친다. 이 과정이 [인가 획득](#인가-획득-obtaining-authorization)과 동일하다. 이렇게 발급받은 토큰은 [카카오스토리 API](https://developers.kakao.com/docs/latest/ko/kakaostory/rest-api) 등을 사용할 때 필요하다. 예를 들어 카카오스토리의 프로필 가져오기 API를 사용하려면 이 발급받은 토큰이 필요하다. 이 과정이 [OAuth 2.0의 흐름](#oauth-20의-흐름)의 5,6번 과정에 속한다.

### 네이버

네이버도 OAuth를 이용하며, [네이버 로그인 API 명세](https://developers.naver.com/docs/login/api/api.md)에서 확인할 수 있다. 카카오와 마찬가지로 인가 코드 받기, 토큰 받기 두 과정을 거친다. 이렇게 발급받은 토큰을 사용하여 회원 프로필 조회 API, 카페 API 등을 사용할 수 있다.

## 마무리

이 포스팅은 OAuth 2.0의 클라이언트 구현에 도움이 되는 내용에 초점을 맞춰 생략된 부분이 많다. 만약 OAuth 2.0에 대해 더 자세히 알고 싶다면, [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)을 참고하는 것이 좋다.
