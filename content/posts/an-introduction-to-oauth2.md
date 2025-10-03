---
title: "OAuth 2.0의 핵심 개념과 동작 원리 이해하기"
description: "안전한 API 인증 및 권한 부여를 위한 표준 프로토콜, OAuth 2.0의 핵심 개념과 가장 널리 사용되는 인가 코드 승인 흐름(Authorization Code Grant Flow)에 대해 자세히 알아봅니다."
date: 2021-10-22T10:08:30+09:00
tags: ["development", "oauth", "security"]
---

## 들어가며

최신 웹 서비스들은 다른 서비스의 기능을 안전하게 연동하여 더 풍부한 사용자 경험을 제공합니다. 예를 들어, 어떤 사진 인화 서비스에서 사용자의 구글 포토에 있는 사진을 직접 불러와 인화할 수 있게 하는 기능을 생각해볼 수 있습니다. 이때 사용자가 자신의 구글 계정 아이디와 비밀번호를 사진 인화 서비스에 직접 제공하는 것은 매우 위험합니다.

**OAuth (Open Authorization)**는 바로 이런 문제를 해결하기 위해 등장한 **인증(Authentication)과 권한 부여(Authorization)를 위한 개방형 표준 프로토콜**입니다. OAuth를 사용하면, 사용자는 자신의 민감한 자격 증명(비밀번호 등)을 제3자 애플리케이션에 노출하지 않고도, 특정 서비스(예: 구글 포토)의 리소스(예: 사진)에 대한 접근 권한을 안전하게 부여할 수 있습니다.

현재는 OAuth 1.0([RFC 5849](https://datatracker.ietf.org/doc/html/rfc5849))을 개선한 OAuth 2.0([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749))이 널리 사용되고 있으며, 이 글에서는 OAuth 2.0을 중심으로 핵심 개념과 동작 방식을 자세히 살펴보겠습니다.

## OAuth 2.0의 주요 역할 (Roles)

OAuth 2.0의 흐름을 이해하려면 먼저 4가지 주요 역할을 알아야 합니다.

1.  **Resource Owner (리소스 소유자)**
    보호된 리소스의 최종 소유자, 즉 사용자입니다. 예를 들어, 은행 서비스의 계좌 잔액이라는 리소스가 있다면, 그 계좌의 주인인 예금주가 리소스 소유자입니다.

2.  **Resource Server (리소스 서버)**
    보호된 리소스를 호스팅하는 서버입니다. 예를 들어, 구글 포토의 사진 데이터, 페이스북의 친구 목록 등을 저장하고 API를 통해 제공하는 서버가 여기에 해당합니다.

3.  **Client (클라이언트)**
    리소스 소유자를 대신하여 리소스 서버의 보호된 리소스에 접근을 요청하는 애플리케이션입니다. 위에서 예로 든 사진 인화 서비스, 소셜 로그인 기능을 사용하는 웹사이트 등이 클라이언트가 됩니다.

4.  **Authorization Server (인증 서버)**
    리소스 소유자의 인증을 처리하고, 클라이언트에게 리소스 접근 권한을 증명하는 **엑세스 토큰(Access Token)**을 발급해주는 서버입니다. 때로는 리소스 서버와 동일한 서버일 수도 있지만, 대규모 서비스에서는 별도로 분리되는 경우가 많습니다.

## OAuth 2.0의 기본 흐름

다음 그림은 OAuth 2.0의 전체적인 상호작용 흐름을 보여줍니다.

```
+--------+                               +---------------+
|        |--(A)- Authorization Request ->|   Resource    |
|        |                               |     Owner     |
|        |<-(B)-- Authorization Grant ---|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(C)-- Authorization Grant -->| Authorization |
| Client |                               |     Server    |
|        |<-(D)----- Access Token -------|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(E)----- Access Token ------>|    Resource   |
|        |                               |     Server    |
|        |<-(F)--- Protected Resource ---|               |
+--------+                               +---------------+
```

각 단계를 쉽게 풀어보면 다음과 같습니다.

1.  **(A) 권한 부여 요청**: 클라이언트(예: 사진 인화 서비스)가 리소스 소유자(사용자)에게 "당신의 구글 포토 사진에 접근하고 싶습니다"라고 요청합니다. 이 과정은 보통 인증 서버를 통해 이루어집니다.
2.  **(B) 권한 부여 승인**: 리소스 소유자가 인증 서버를 통해 본인임을 인증(로그인)하고, 클라이언트의 요청을 검토한 후 접근을 허용합니다. 이 결과로 클라이언트는 **인가 승인(Authorization Grant)**이라는 임시 자격 증명을 얻게 됩니다.
3.  **(C) 엑세스 토큰 요청**: 클라이언트는 (B)에서 받은 인가 승인을 인증 서버에 제출하며 "리소스 소유자가 접근을 허락했으니, 이제 리소스 서버에 접근할 수 있는 엑세스 토큰을 발급해주세요"라고 요청합니다.
4.  **(D) 엑세스 토큰 발급**: 인증 서버는 클라이언트의 신원과 인가 승인을 검증한 후, 유효하다면 **엑세스 토큰**을 발급합니다.
5.  **(E) 리소스 접근**: 클라이언트는 발급받은 엑세스 토큰을 가지고 리소스 서버에 가서 "이 토큰을 가진 저에게 보호된 리소스(사진)를 주세요"라고 요청합니다.
6.  **(F) 리소스 제공**: 리소스 서버는 엑세스 토큰이 유효한지 확인하고, 유효하다면 요청받은 리소스를 클라이언트에게 제공합니다.

## 인가 승인 (Authorization Grant)

**인가 승인**은 리소스 소유자가 클라이언트에게 접근 권한을 부여했음을 나타내는 자격 증명으로, 클라이언트가 엑세스 토큰을 얻기 위해 사용됩니다. OAuth 2.0은 시나리오에 따라 네 가지 표준 승인 유형을 정의합니다.

1.  **인가 코드 (Authorization Code)**: 가장 일반적이고 안전한 방식으로, 클라이언트가 사용자를 인증 서버로 리디렉션하여 인증 코드를 받아오고, 그 코드를 사용해 엑세스 토큰을 발급받습니다. (이 글에서 자세히 다룹니다.)
2.  **암시적 승인 (Implicit)**: 주로 자바스크립트 기반의 단일 페이지 애플리케이션(SPA)과 같이 서버 측 코드가 없는 클라이언트를 위해 설계되었습니다. 보안상 취약점이 있어 현재는 권장되지 않으며, 대신 인가 코드 방식에 PKCE(Proof Key for Code Exchange)를 추가한 방식이 사용됩니다.
3.  **리소스 소유자 암호 자격증명 (Resource Owner Password Credentials)**: 사용자의 아이디와 비밀번호를 클라이언트가 직접 받아 인증하는 방식으로, 클라이언트를 완전히 신뢰할 수 있는 경우(예: 서비스에서 직접 만든 공식 앱)에만 제한적으로 사용해야 합니다.
4.  **클라이언트 자격증명 (Client Credentials)**: 사용자의 개입 없이 클라이언트가 자신의 자격증명만으로 인증하는 방식입니다. 클라이언트가 관리하는 리소스에 접근할 때 사용됩니다.

이 글에서는 가장 대표적인 **인가 코드(Authorization Code)** 방식에 대해 집중적으로 설명합니다.

## 엑세스 토큰과 리프레시 토큰

### 엑세스 토큰 (Access Token)
보호된 리소스에 접근할 수 있는 열쇠와 같은 역할을 하는 권한 증명입니다. 보통 만료 시간이 짧게 설정되어 있어(예: 1시간), 토큰이 탈취되더라도 피해를 최소화할 수 있습니다.

### 리프레시 토큰 (Refresh Token)
엑세스 토큰이 만료되었을 때, 사용자의 재인증 없이 새로운 엑세스 토큰을 발급받기 위해 사용하는 특별한 토큰입니다. 엑세스 토큰보다 훨씬 긴 유효 기간을 가지며, 안전하게 저장하고 관리해야 합니다. 리프레시 토큰을 사용하면 사용자가 매번 로그인해야 하는 불편함을 줄일 수 있습니다.

다음은 리프레시 토큰을 사용한 엑세스 토큰 갱신 흐름입니다.
```
+--------+                                           +---------------+
|        |--(1)------- Authorization Grant --------->|               |
|        |                                           |               |
|        |<-(2)----------- Access Token -------------|               |
|        |               & Refresh Token             |               |
...
|        |--(5)---- Access Token ---->| Resource |   | Authorization |
|        |                            |  Server  |   |     Server    |
|        |<-(6)- Invalid Token Error -|          |   |               |
...
|        |--(7)----------- Refresh Token ----------->|               |
|        |                                           |               |
|        |<-(8)------- New Access Token -------------|               |
+--------+                                           +---------------+
```
(6)번 단계에서 엑세스 토큰이 만료되어 오류가 발생하면, 클라이언트는 (7)번 단계에서 저장해 둔 리프레시 토큰을 인증 서버로 보내 (8)번 단계에서 새로운 엑세스 토큰을 발급받습니다.

## 인가 코드 승인 유형 상세 흐름

네이버나 카카오 로그인과 같은 소셜 로그인 기능은 대부분 이 방식을 사용합니다. 이 방식은 리디렉션을 기반으로 동작하므로, 클라이언트는 사용자의 웹 브라우저와 상호작용할 수 있어야 합니다.

```
+----------+
| Resource |
|   Owner  |
|          |
+----------+
     ^
     |
    (B)
+----|-----+          Client Identifier      +---------------+
|         -+----(A)-- & Redirection URI ---->|               |
|  User-   |                                 | Authorization |
|  Agent  -+----(B)-- User authenticates --->|     Server    |
| (Browser)|                                 |               |
|         -+----(C)-- Authorization Code ---<|               |
+-|----|---+                                 +---------------+
  |    |                                         ^      v
 (A)  (C)                                        |      |
  |    |                                         |      |
  ^    v                                         |      |
+---------+                                      |      |
|         |>---(D)-- Authorization Code ---------'      |
|  Client |          & Redirection URI                  |
|         |                                             |
|         |<---(E)----- Access Token -------------------'
+---------+       (w/ Optional Refresh Token)
```

1.  **(A) 인가 요청**: 클라이언트가 사용자의 브라우저를 인증 서버의 로그인 페이지로 안내하며 흐름이 시작됩니다. 이때 클라이언트 식별자(`client_id`), 요청 범위(`scope`), 리디렉션 URI(`redirect_uri`) 등의 정보를 함께 전달합니다. 사용자가 [네이버 아이디로 로그인] 버튼을 클릭하는 순간이 바로 이 과정입니다.

2.  **(B) 사용자 인증 및 동의**: 인증 서버는 사용자에게 로그인 및 접근 권한 동의를 요청합니다. 사용자는 아이디/비밀번호를 입력하여 본인임을 인증하고, "프로필 정보 제공"과 같은 권한 부여에 동의하거나 거부합니다.
    ![](/images/naver_login.png)

3.  **(C) 인가 코드 발급**: 사용자가 동의하면, 인증 서버는 생성된 **인가 코드(Authorization Code)**를 쿼리 파라미터에 담아 (A)에서 지정된 `redirect_uri`로 사용자의 브라우저를 이동시킵니다.

4.  **(D) 엑세스 토큰 요청**: 클라이언트는 백엔드에서 (C)를 통해 전달받은 인가 코드를 사용하여 인증 서버에 엑세스 토큰을 직접 요청합니다. 이 요청은 사용자의 브라우저를 거치지 않으므로 안전합니다.

5.  **(E) 엑세스 토큰 발급**: 인증 서버는 인가 코드가 유효한지, 리디렉션 URI가 일치하는지 등을 검증한 후, 최종적으로 **엑세스 토큰**과 선택적으로 **리프레시 토큰**을 클라이언트에 응답합니다.

### 인가 요청 (Authorization Request)

클라이언트는 `application/x-www-form-urlencoded` 형식으로 다음 파라미터를 포함하여 인증 서버에 요청해야 합니다.

-   `response_type`: **필수.** `"code"`로 고정해야 합니다.
-   `client_id`: **필수.** 클라이언트를 식별하는 ID입니다.
-   `redirect_uri`: **선택사항.** 인가 코드를 전달받을 URI입니다. 보안을 위해 사전에 등록된 URI와 일치하는지 검증하는 것이 중요합니다.
-   `scope`: **선택사항.** 클라이언트가 요청하는 접근 권한의 범위입니다. (예: `profile`, `email`)
-   `state`: **권장사항.** CSRF(Cross-Site Request Forgery) 공격을 방지하기 위한 임의의 문자열입니다. 인증 서버는 응답 시 이 값을 그대로 반환해야 합니다.

**요청 예시:**
```http
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com
```

### 인가 응답 (Authorization Response)

#### 성공 응답
사용자가 접근을 승인하면, 인증 서버는 `redirect_uri`로 다음 파라미터를 전달합니다.

-   `code`: **필수.** 인증 서버가 생성한 인가 코드입니다. 보안을 위해 보통 10분 이내의 짧은 만료 시간을 가지며, 한 번만 사용할 수 있습니다.
-   `state`: 요청 시 `state` 파라미터를 포함했다면 **필수**로 포함되어야 하며, 요청 시의 값과 동일해야 합니다.

**응답 예시 (리디렉션):**
```http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```

#### 오류 응답
요청이 유효하지 않거나 사용자가 거부한 경우, 다음과 같은 오류 코드를 `redirect_uri`로 전달합니다.

-   `error`: **필수.** 오류 유형을 나타내는 코드입니다. (예: `access_denied`, `invalid_request`)
-   `error_description`: **선택사항.** 오류에 대한 상세 설명입니다.
-   `state`: 요청 시 `state` 파라미터를 포함했다면 **필수**입니다.

**오류 응답 예시 (리디렉션):**
```http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?error=access_denied&state=xyz
```

### 엑세스 토큰 요청 (Access Token Request)

클라이언트는 다음 파라미터를 `application/x-www-form-urlencoded` 형식의 HTTP Body에 담아 인증 서버에 토큰을 요청합니다.

-   `grant_type`: **필수.** `"authorization_code"`로 고정해야 합니다.
-   `code`: **필수.** 인가 응답으로 받은 인가 코드입니다.
-   `redirect_uri`: 인가 요청 시 `redirect_uri`를 사용했다면 **필수**이며, 동일한 값을 보내야 합니다.
-   `client_id`: **필수.** 클라이언트 ID입니다. (클라이언트 인증 방식에 따라 `Authorization` 헤더에 `client_id`와 `client_secret`을 포함할 수도 있습니다.)

**요청 예시:**
```http
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

### 엑세스 토큰 응답 (Access Token Response)

#### 성공 응답
요청이 유효하면 인증 서버는 `application/json` 형식으로 다음 정보를 포함한 응답을 반환합니다.

-   `access_token`: **필수.** 발급된 엑세스 토큰입니다.
-   `token_type`: **필수.** 토큰 유형으로, 보통 `Bearer` 또는 `MAC` 타입을 사용합니다.
-   `expires_in`: **권장사항.** 엑세스 토큰의 유효 시간(초)입니다.
-   `refresh_token`: **선택사항.** 새로운 엑세스 토큰을 발급받기 위한 리프레시 토큰입니다.
-   `scope`: **선택사항.** 최종적으로 부여된 권한의 범위입니다.

**성공 응답 예시:**
```http
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "access_token": "2YotnFZFEjr1zCsicMWpAA",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
    "scope": "profile"
}
```

#### 오류 응답
요청이 실패하면 인증 서버는 400 (Bad Request) 상태 코드와 함께 `application/json` 형식으로 오류 정보를 반환합니다.

-   `error`: **필수.** 오류 유형을 나타내는 코드입니다. (예: `invalid_grant`, `invalid_client`)
-   `error_description`: **선택사항.** 오류에 대한 상세 설명입니다.

**오류 응답 예시:**
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "error": "invalid_grant"
}
```

## 마무리

이 글에서는 OAuth 2.0의 클라이언트 구현에 도움이 되는 내용을 중심으로, 특히 가장 널리 사용되는 '인가 코드 승인' 방식에 대해 자세히 알아보았습니다. OAuth 2.0은 안전한 API 생태계를 구축하는 데 필수적인 기술이며, 각 승인 유형의 특징과 보안 고려사항을 정확히 이해하고 적용하는 것이 중요합니다.

더 깊이 있는 정보나 다른 승인 유형에 대해 알고 싶다면, 공식 명세인 [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)를 참고하시는 것이 가장 좋습니다.
