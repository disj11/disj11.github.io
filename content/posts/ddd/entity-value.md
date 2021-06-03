---
title: 엔티티와 밸류
date: 2021-06-03T20:00:00+09:00
menu:
  sidebar:
    name: 엔티티와 밸류
    identifier: ddd-entity-value
    parent: ddd
    weight: 10
---

## 엔티티와 밸류

도메인 모델은 Entity와 Value로 구분할 수 있다. 엔티티의 가장 큰 특징은 식별자를 갖는다는 것이다. 밸류는 엔티티와는 다르게 식별자를 갖지 않으며, 개념적으로 같은 데이터를 표현 할 때 사용한다. 예를 들어 `Order`라는 주문 도메인과 받는 사람을 표현하는 `Receiver`가 존재한다고 하자. `Order`라는 주문 도메인은 주문 번호라는 식별자가 있다. 이와 달리 `Receiver`는 식별자는 없지만, 받는 사람의 이름과 받는 사람의 전화번호 등 '받는 사람'이라는 개념적으로 동일한 필드를 표현하기 위해 사용할 수 있다.

```java
public class Order {
    private String orderNumber;
}

public class Receiver {
    private String name;
    private String phoneNumber;
}
```

밸류 타입은 꼭 두 개 이상의 데이터를 가져야 하는 것은 아니다. 아래와 같이 데이터의 의미를 명확하게 하기 위해 사용되는 경우도 있다.

```java
public class Money {
    private int value;
    
    public Money(int value) {
        this.value = value;
    }
    
    public int getValue() {
        return this.value;
    }
    
    public Money add(Money money) {
        return new Money(this.value + money.value);
    }
}
```

이와 같이 `Money` 타입을 만들어 사용할 경우 `Money` 타입을 사용하는 코드는 이제 단순히 '정수 타입 연산'이 아니라 '돈 계산' 이라는 의미를 명확하게 해준다.

한 가지 참고 사항으로, 밸류 객체는 불변으로 만드는 것이 좋다. `Money` 클래스의 `add()` 메서드를 보면 기존 값에 value를 더하는 것이 아닌 Money를 새로 생성한다. 이처럼 데이터 변경 기능을 제공하지 않는 타입을 불변(immutable)이라고 표현한다. 이렇게 불변 타입을 사용하면 보다 안전한 코드를 작성할 수 있다. 예를 들어 주문 항목을 표현하는 `OrderLine`이 있다고 하자.

```java
public class OrderLine {
    public OrderLine(Product product, Money price, int quantity) {
        this.price = price;
        this.quantity = quantity;
        this.amounts = calculateAmounts(); // price.value * quantity
    }
}
```

만약 Money가 setValue() 메서드를 제공한다면 다음과 같은 상황이 벌어질 수 있다.

```java
Money price = new Money(1000);
OrderLine line = new OrderLine(product, price, quantity); // 1
price.setValue(2000); // 2

// 1: [price=1000, quantity=2, amounts=2000]
// 2: [price=2000, quantity=2, amounts=2000]
```

이런 문제(참조 투명성)를 회피하기 위해서는 다음과 같은 코드가 필요하다.

```java
public class OrderLine {
    public OrderLine(Product product, Money price, int quantity) {
        this.price = new Money(price.value);
        this.quantity = quantity;
        this.amounts = calculateAmounts(); // price.value * quantity
    }
}
```

하지만 Money를 불변으로 만든다면, Money의 데이터를 바꿀 수 없기 때문에 이런 코드를 작성하지 않아도 파라미터로 전달받은 price를 안전하게 사용할 수 있다. 이처럼 [불변 객체](https://goo.gl/2Lo4pU)는 참조투명성과 스레드에 안전한 특징을 갖고 있다.