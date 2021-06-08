---
title: 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라
date: 2021-06-08T22:29:00+09:00
menu:
  sidebar:
    name: 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라
    identifier: dependency-injection
    parent: effective-java
    weight: 10
---

맞춤법 검사기를 개선하며 의존 객체 주입의 장점을 알아보자.

우선 맞춤법 검사기를 정적 유틸리티 클래스를 통하여 구현한 모습을 살펴보자.

## **정적 유틸리티 클래스 사용**

```java
public class SpellChecker {
    private static final Lexicon dictionary = ...;

    private SpellChecker() {}

    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```

이와 비슷하게 싱글턴으로 구현하는 방법도 있다.

## **싱글턴 사용**

```java
public class SpellChecker {
    private final Lexicon dictionary = ...;

    private SpellChecker(...) {}
    public static SpellChecker INSTANCE = new SpellChecker(...);

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

두 방식의 문제점은 사전을 하나만 사용한다고 가정했다는 것이다. 실무에서는 사전이 언어 별로 따로 존재하고, 특수 어휘용 사전이 별도로 필요할 수도 있다. 이 문제점을 어떻게 고칠 수 있을까?

가장 간단히 고치는 방법은 final을 지우고 사전을 교체하는 메서드를 추가할 수 있을 것이다. 하지만 이 방식은 오류를 내기 쉬우며, 멀티 스레드 환경에서는 사용할 수 없다는 문제가 있다. 이처럼 **사용하는 자원에 따라 동작이 달라지는 클래스에서는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않다.**

이런 상황에서는 **의존 객체 주입**을 사용할 수 있다. 의존 객체 주입의 한 형태로 **인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방법**이 있다. 변경된 코드는 아래와 같다.

## **의존 객체 주입 사용**

```java
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

위 코드는 dictionary라는 하나의 자원을 사용하지만, 자원이 몇 개가 되든, 의존 관계가 어떻든 상관없이 잘 작동한다. 또 불변을 보장하여 여러 클라이언트가 안심하고 의존 객체들을 공유하여 사용할 수 있다.

## **정리**

클래스가 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스 동작에 영향을 준다면, 정적 유틸리티 클래스와 싱글턴은 사용하지 않는 것이 좋다. 대신 필요한 자원 혹은 자원을 만들어 주는 팩터리를 생성자, 정적 팩터리, 빌더에 넘겨주자. 의존 객체 주입은 클래스의 유연성, 재사용성, 테스트 용이성을 개선해준다.