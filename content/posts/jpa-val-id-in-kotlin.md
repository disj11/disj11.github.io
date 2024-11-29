---
title: "코틀린에서 JPA 사용시 ID 에 val 사용하기"
description: "JPA 에서 ID 에 val 을 사용해도 save 이후 ID 가 잘 설정된다."
date: 2023-09-15T05:10:46+09:00
url: "/jpa-val-id-in-kotlin/"
tags: [JPA, TIL]
---

## TL;DR

JPA 에서 ID 에 `val` 을 사용해도 `save` 이후 ID 가 잘 설정된다.

## 코드 예제

*Member.kt*

```kotlin
@Entity
class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    @Column(
        name = "name",
        length = 30,
        nullable = false,
    )
    var name: String,
)
```

*MemberRepository.kt*

```kotlin
@Repository
interface MemberRepository : JpaRepository<Member, Long>
```

*사용처*

```kotlin
val member = Member(name = "홍길동") // member.id = null
val savedMember = memberRepository.save(member) // savedMember.id = 1
println(member.id) // member.id = 1
```

보는 바와 같이 `val` 로 id 를 설정하였어도 `save` 이후에 id 값이 잘 설정된다.  
하지만 일반적으로 `val` 은 값이 변한다고 생각하지 않기 때문에 값을 저장한 이후에는 `member` 대신 `savedMember` 를 사용하는 것이 좋아 보인다.
