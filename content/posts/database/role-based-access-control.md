---
title: RBAC (Role Based Access Control - 역할 기반 접근 제어)
date: 2019-06-15T02:30:00+09:00
menu:
  sidebar:
    name: RBAC (Role Based Access Control - 역할 기반 접근 제어)
    identifier: role-based-access-control
    parent: database
    weight: 10
---

## 개요

역할 기반 접근 제어는 사용자의 역할에 따라 권한을 구분하고, 권한이 없는 사용자에게는 시스템 접근을 제한하는 방법이다. 사전에 역할을 미리 정의해두고, 사용자에게 역할을 부여함으로써 권한을 제어한다.

## 간단한 솔루션

Stackoverflow 같은 사이트를 만든다고 가정하자. 우리는 아래와 같은 두 가지 모듈이 필요한 상황이다.

* 질문 (questions)
* 채팅 (chat)

이 모듈의 권한을 관리하기 위해서는 아래와 같은 테이블이 필요하다.

```
# Modules
- id : 모듈의 고유 ID
- name : 모듈에 대한 설명
```

그리고 이 모듈들이 할 수 있는 액션을 정의한다. 예를 들어 질문 모듈의 경우

1. 질문 등록
2. 질문 삭제

등등 이 있을 수 있고, 채팅 모듈의 경우

1. 채팅방 열기
2. 강퇴

등등이 있을 수 있다.

```
# Actions
- module_id : 사전에 정의한 모듈의 ID
- action : 모듈의 기능
- name : 기능에 대한 설명

- Primary Key : [module_id, action]
```

이제 두 테이블을 정의한 모습을 보자.

```
Modules
+------------------------------------------+
| ID        | Name                         |
+------------------------------------------+
| questions | Question and Answer Module   |
| chat      | Chat Module                  |
+------------------------------------------+
```

```
Actions
+-----------------------------------------------+
| Module    | Action    |  Name                 |
+-----------------------------------------------+
| questions | read      | Read Questions        |
| questions | create    | Create Questions      |
| questions | edit      | Edit Questions        |
| questions | delete    | Delete Questions      |
|           |           |                       |
| chat      | join      | Join the Chat         |
| chat      | kick      | Kick users            |
| chat      | create    | Create Chatrooms      |
+-----------------------------------------------+  
```

여기서 중요한 점은 위 두 테이블은 *시스템 관리자가 직접 수정할 수 없는 데이터*이므로, 기능 추가 / 제거와 같은 GUI 가 있어서는 안된다는 점이다.

여기까지 만들어졌다면 이제는 역할을 만들어야 한다. 역할 테이블은 위 테이블들과는 달리 관리자에 의해 관리될 수 있으며, 기본적으로 아래와 같은 역할이 있을 수 있다.

```
Q&A User:
   - 질문을 읽을 수 있음
   - 질문을 등록 할 수 있음
Q&A Moderator:
   - 질문을 읽을 수 있음
   - 질문을 등록 할 수 있음
   - 질문을 수정할 수 있음
Q&A Admin:
   - 질문을 읽을 수 있음
   - 질문을 등록 할 수 있음
   - 질문을 수정할 수 있음
   - 질문을 삭제 할 수 있음

Chat User:
   - 채팅방에 참여할 수 있음

Chat Moderator:
   - 채팅방에 참여할 수 있음
   - 채팅방에서 사용자를 강퇴할 수 있음

Chat Admin:
   - 채팅방에 참여할 수 있음
   - 채팅방에서 사용자를 강퇴할 수 있음
   - 채팅방을 개설할 수 있음
```

그럼 위 내용을 바탕으로 역할 테이블을 만들어 보자

```
Roles
+-----------------------+
| ID | Name             |
+-----------------------+
| 1  | Q&A User         |
| 2  | Q&A Moderator    |
| 3  | Q&A Admin        |
| 4  | Chat User        |
| 5  | Chat Moderator   |
| 6  | Chat Admin       |
+-----------------------+
```

위와 같은 테이블이 탄생하였다. 이제 역할에서 허용할 기능을 알려줄 `roles_actions`을 정의하자.

```
# Roles_Actions
  - role_id
  - module_id 
  - action

  PK: [role_id, module_id, action]
```

위 테이블에는 아래와 같은 내용이 들어가게 될 것이다.

```
Roles_Actions
+--------------------------------+
| Role ID | Module ID | Action   |
+--------------------------------+
|    1    | questions |  read    |
|    1    | questions |  create  |
|    2    | questions |  read    |
|    2    | questions |  create  |
|    2    | questions |  edit    |
               ...  
|    6    |    chat   |  join    |
|    6    |    chat   |  kick    |
|    6    |    chat   |  create  |
+--------------------------------+
```

마지막으로 사용자에게 역할을 부여하기 위한 테이블 `users_roles`을 정의하자.

```
# User_Roles
   - user_id [FK:user_id, unsigned]
   - role_id [FK:roles_id, unsigned]
```

테이블의 내용은 아래와 같다.

```
Users_Roles
+---------------------------------+
| User ID | Role ID               |
+---------------------------------+
|    1    |  3 (= Q&A Admin)      |
|    1    |  6 (= Chat Admin)     |
|    2    |  2 (= Q&A Moderator)  |    
|    2    |  4 (= Chat User)      |
|    3    |  2 (= Q&A Moderator)  |  
|    3    |  5 (= Chat Moderator) | 
+---------------------------------+
```

## 참고

[https://stackoverflow.com/questions/28157798/is-my-role-based-access-control-a-feasible-solution/28159647#28159647](https://stackoverflow.com/questions/28157798/is-my-role-based-access-control-a-feasible-solution/28159647#28159647)