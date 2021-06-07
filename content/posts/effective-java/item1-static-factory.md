---
title: 정적 팩터리 메서드를 고려하라
date: 2021-06-03T21:00:00+09:00
menu:
  sidebar:
    name: 정적 팩터리 메서드를 고려하라
    identifier: static-factory-method
    parent: effective-java
    weight: 10
---

생성자 대신 정적 팩터리 메서드를 사용할때의 장점과 단점을 알아보자

### **장점**

### 1. 이름을 갖을 수 있다.

반환 될 객체의 특성을 명확히 알 수 있는 이름을 지을 수 있다. 예를들어 생성자인 `BigInteger(int, int, Random)` 보다 정적 팩터리 메서드인 `BigInteger.probablePrime` 쪽이 '값이 소수인 BigInteger를 반환한다'는 의미를 더 잘 들어낼 수 있다. 또 생성자는 하나의 시그니처로 하나의 생성자만 만들 수 있지만, 정적 팩터리 메서드는 이러한 제약이 없다.

### 2. 인스턴스를 통제할 수 있다.

호출될 때마다 새로운 인스턴스를 생성하지 않아도 된다. 인스턴스를 미리 만들어 재활용하는 식으로 불필요한 객체 생성을 피할 수 있다. 대표적인 예로 `Boolean.valueOf(boolean)` 메서드가 있다. 특히 생성 비용이 큰 객체가 자주 요청되는 상황에 이 방식을 사용한다면 상당한 성능을 끌어올릴 수 있다.

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있다.

반환할 객체의 클래스를 자유롭게선택할 수 있는 유연성을 제공한다. 대표적인 예로 자바 컬렉션 프레임워크는 `java.util.Collections`에서 정적 팩터리 메서드를 통해 45개의 유틸리티 구현체를 제공한다. 이를 통해 API가 작아진 것은 물론 프로그래머가 API를 사용하기 위한 난이도도 낮추었다. 자바 8부터는 인터페이스가 정적 메서드를 가질 수 없다는 제한도 풀렸기 때문에 이를 활용할 수도 있다.

### 4. 입력 매개변수에 따라 다른 클래스의 객체를 반환 할 수 있다.

반환 타입의 하위 타입이기만 하면 어떤 클래스의 객체를 반환하든 상관없다. 예를 들어 `EnumSet` 클래스의 경우 원소가 64개 이하라면 `RegularEnumSet`을, 64개 이상이라면 `JumboEnumSet`을 반환하여 효율성을 높였다.

### 5. 정적 팩터리를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.

이러한 특성은 서비스 제공자 프레임워크(service provider framework)를 만드는 근간이 된다. 대표적으로 JDBC의 서비스 접근 API역할을 하는 `DriverManager.getConnection`이 정적 팩터리를 사용한 것이다.

### **단점**

### 1. 하위 클래스를 만들 수 없다.

상속을 위해서는 public이나 protected 생성자가 필요하기 때문에 정적 팩터리 메서드만 제공한다면 하위 클래스를 만들 수 없다. 하지만 이 제약은 컴포지션 사용을 유도하고 불변 타입으로 만들기 위해서는 이 제약을 지켜야 하기 때문에 오히려 장점으로 작용할 수 있다.

### 2. 프로그래머가 찾기 어렵다.

생성자처럼 API 설명에 명확히 드러나지 않기 때문에 사용자는 정적 팩터리 메서드 방식의 클래스를 인스턴스화 할 방법을 알아야한다. 이 문제를 완화하기 위해 알려진 규약을 따라 메서드 이름을 짓는 것이 좋다.

- from: 매개변수를 하나 받아 해당 타입의 인스턴스를 반환

  `Date d = Date.from(instant);`

- of: 여러 매개변수를 받아 적절한 타입의 인스턴스를 반환

  `Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);`

- valueOf: from과 of의 자세한 버전

  `BigInteger.valueOf(Integer.MAX_VALUE);`

- instance or getInstance: 매개변수로 명시한 인스턴스를 반환하지만 같은 인스턴스임을 보장하지 않음

  `StackWalker luke = StackWalker.getInstance(options);`

- create or newInstance: instance, getInstance와 같지만 매번 새로운 인스턴스를 생성해 반환

  `Object newArray = Array.newInstance(clasObject, arrayLen);`

- getType: getInstance와 같지만 생성할 클래스가 아닌 다른 클래스의 팩터리 메서드로 정의할 때 사용한다. "Type"은 메서드가 반환할 객체의 타입이다.

  `FileStore fs = Files.getFileStore(path);`

- newType: newInstance와 같지만 생성할 클래스가 아닌 다른 클래스의 팩터리 메서드로 정의할 때 사용한다.. "Type"은 메서드가 반환할 객체의 타입이다.

  `BufferedReader br = Files.newBufferedReader(path);`

- type: getType과 newType의 간결한 버전

  `List<Complaint> litany = Collections.list(legacyLitany);`

### **정리**

정적 팩터리 메서드를 사용하는 것이 유리한 경우가 많으므로, 무작정 public 생성자를 제공하던 습관을 고치고 정적 팩터리 메서드의 사용을 고려하자.