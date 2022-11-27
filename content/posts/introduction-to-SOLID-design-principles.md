---
title: "Introduction to SOLID Design Principles"
date: 2021-11-03T21:21:49+09:00
tag: ["development"]
---

## 개요

SOLID 란 Robert C.Martin 이 명명한 객체 지향 프로그래밍의 다섯 가지 설계 원칙이다. SOLID 는 다음을 의미한다.

* **S** - Single-responsibility Principle (SRP: 단일 책임 원칙)
* **O** - Open-closed Principle (OCP: 개방-폐쇄 원칙)
* **L** - Liskov Substitution Principle (LSP: 리스코프 치환 원칙)
* **I** - Interface Segregation Principle (ISP: 인터페이스 분리 원칙)
* **D** - Dependency Inversion Principle (DIP: 의존 관계 역전 원칙)

이번 포스팅에서는 이 다섯 가지 원칙에 대해서 알아본다.

## 단일 책임 원칙

하나의 클래스는 한 가지 책임만 가져야 하며, 클래스를 변경해야 하는 이유는 단 하나여야 한다는 원칙이다. 이 원칙을 지키면 어떤 점이 좋을까?

1. **테스팅** - 하나의 책임만을 가졌으므로 훨씬 더 적은 테스트 케이스로 테스트를 만들 수 있다.
2. **낮은 결합도** - 하나의 클래스가 갖는 기능이 적을수록 종속성이 줄어 든다. 이는 클래스가 변경되었을 때 외부의 영향을 신경쓸 필요가 줄어든다는 것을 의미한다.

예제를 통해 단일 책임 원칙을 지키는 코드를 알아보자.

```java
public class Book {

    private String name;
    private String author;
    private String text;

    //constructor, getters and setters
}
```

이름과 저자, 텍스트를 갖는 간단한 Book 클래스이다. 여기에 텍스트 속성과 관련된 몇 가지 메서드를 추가해 보자.

```java
public class Book {

    private String name;
    private String author;
    private String text;

    //constructor, getters and setters
    
    // Book 클래스의 속성과 직접적인 연관을 갖는 메서드
    public String replaceWordInText(String word) {
        return text.replaceAll(word, text)
    }
    
    public boolean isWordInText(String word) {
        return text.contains(word);
    }
}
```

이제 책을 읽기 위해 `printTextToConsole()` 메서드를 추가하려고 한다. 다음과 같이 `Book` 클래스에 메서드를 추가하면 될까?

**Bad:**

```java
public class Book {
    //...

    void printTextToConsole() {
        // 텍스트 서식 지정 및 출력
    }
}
```

이는 앞에서 설명한 단일 책임 원칙을 위반한다. 단일 책임 원칙을 지키기 위해서는 텍스트 출력에 대한 별도 클래스를 생성해야 한다.

**Good:**

```java
public class BookPrinter {
    void printTextToConsole(String text) {
        // 텍스트 서식 지정 및 출력
    }

    void printTextToAnotherMedium(String text) {
        // 다른 미디어로 텍스트를 보내기 위한 코드...
    }
}
```

콘솔로 텍스트를 출력하는 기능을 만들었다. 콘솔이 아닌 다른 미디어로 텍스트를 출력하는 기능을 추가할 수도 있을 것이다. 미디어가 이메일이 됐든, 로그가 됐든, 어느 것이든 상관없이 이제 텍스트 출력이라는 문제에만 전념할 수 있는 클래스가 만들어졌다.

## 개방-폐쇄 원칙

클래스는 확장에는 열려있지만, 수정에는 닫혀있어야 한다는 원칙이다. 조금 더 쉽게 풀면, 기존의 코드를 수정하지 않고도 새로운 기능을 추가할 수 있어야 한다는 것이다. 이 원칙을 지키면 잠재적인 새로운 버그를 방지하는 데 도움이 된다.

**Bad:**

```java
public class Vehicle {
    private Type type;
    
    public void move() {
        if (Type.AIR_PLANE.equals(type)) {
            System.out.println("하늘을 날다");
        } else if (Type.CAR.equals(type)) {
            System.out.println("도로를 달린다");
        } else if (Type.SUBWAY) {
            System.out.println("철도를 달린다");
        }
    }
}
```

이 코드는 탈 것의 종류가 추가될 때마다 기존의 코드를 수정해야 한다. 개방-폐쇄 원칙을 지킨 코드는 다음과 같다.

```java
public interface Vehicle {
    void move();
}

public class AirPlane implements Vehicle {
    @Override
    public void move() {
        System.out.println("하늘을 날다");
    }
}

public class Car implements Vehicle {
    @Override
    public void move() {
        System.out.println("도로를 달린다");
    }
}

public class Subway implements Vehicle {
    @Override
    public void move() {
        System.out.println("철도를 달린다");
    }
}
```

이 코드는 새로운 탈 것이 추가되더라도 기존의 코드 변경 없이 새로운 클래스를 생성하면 된다.

## 리스코프 치환 원칙

S가 T의 하위형 타입이라면 T 타입의 객체는 어떠한 속성의 수정 없이 타입 S로 교체할 수 있어야 한다는 원칙이다. 리스코프 치환 원칙 위반의 예로 가장 많이 사용하는 것은 직사각형 클래스로부터 정사각형 클래스를 파생하는 것이다.

**Bad:**

```java
public class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getArea() {
        return width * heght;
    }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.hegiht = height;
    }
    
    @Override
    public void setHeight(int height) {
        this.height = height;
        this.width = height;
    }
}
```

정사각형은 가로와 세로가 항상 같아야 하기 때문에 가로와 세로를 독립적으로 변경할 수 없다. 그렇다고 해서 이 코드의 `Square` 와 같이 메서드를 오버라이딩 하게 되면, 가로와 세로를 독립적으로 변경할 수 있는 직사각형 할당자의 조건을 위반하게 된다. 이 코드를 클라이언트에서 다음과 같이 사용했다고 해보자.

```java
public void printArea(List<Rectangle> rectangles) {
    for (Rectangle rectangle : rectangles) {
        rectangle.setWidth(4);
        rectangle.setHeight(5);
        System.out.println(rectangle.getArea());    
    }
}
```

이 메서드는 20 이 출력 될까? 만약 매개 변수로 받은 클래스의 타입이 `Square` 라면 25 가 출력될 것이다. 리스코프 치환 원칙을 지키는 코드는 다음과 같다.

**Good:**

```java
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private int width;
    private int height;
    
    // setter
    
    @Override
    public int getArea() {
        return width * height;
    }
}

public class Square implements Shape {
    private int length;
    
    // setter
    
    public getArea() {
        return length * length;
    }
}
```

## 인터페이스 분리 원칙

많은 기능을 가진 인터페이스를 더 작게 분할해야 한다는 원칙이다. 다음의 예시를 보자.

**Bad:**

```java
public interface Printer {
    void print();
    void fax();
    void scan();
}

public class AllInOnePrinter implements Printer {
    public void print() {
        // ...
    }
    
    public void fax() {
        // ...
    }
    
    public void scan() {
        // ...
    }
}

public class EconomicPrinter implements Printer {
    public void print() {
        // ...
    }

    public void fax() {
        throw new UnsupportedOperationException("Fax not supported.");
    }

    public void scan() {
        throw new UnsupportedOperationException("Scan not supported.");
    }
}
```

인터페이스 분리 원칙을 따른 코드는 다음과 같다.

**Good:**

```java
public interface Printer {
  void print();
}

public interface Fax {
  void fax();
}

public interface Scanner {
  void scan();
}

public class AllInOnePrinter implements Printer, Fax, Scanner {
  print() {
    // ...
  }  
  
  fax() {
    // ...
  }

  scan() {
    // ...
  }
}

public class EconomicPrinter implements Printer {
  print() {
    // ...
  }
}
```

## 의존 관계 역전 원칙

이 원칙은 소프트웨어 모듈을 분리 시 지켜야 하는 두 가지 사항을 명시한다.

1. 상위 모듈은 하위 모듈에 의존해서는 안된다. 상위 모듈과 하위 모듈 모두 추상화에 의존해야 한다.
2. 추상화는 세부 사항에 의존해서는 안된다. 세부 사항은 추상화에 의존해야 한다.

설명 만으로는 이해하기가 힘들 수 있지만, 우리가 자주 사용하는 의존성 주입도 이 원칙을 따르는 방법 중 하나이다. 다음 예시를 보자.

**Bad:**

```java
public class MailService {
    private final RealMailSender mailSender;
    
    public MailService() {
        this.mailSender = new RealMailSender();
    }
    
    public void send() {
        mailSender.send();
    }
}
```

이 코드는 `new` 키워드로 생성된 `RealMailSender` 로 인해 `MailService` 와 `RealMailSender` 간의 강한 결합을 발생시킨다. 이는 테스트를 어렵게 만들 뿐 아니라 `RealMailSender` 클래스를 다른 메일 발송 클래스로 전환하는 것도 어렵게 만든다. 의존 관계 역전 원칙을 지키는 코드를 살펴보자.

**Good:**

```java
public interface MailSender {
    void send();
}

public class RealMailSender implements MailSender {
    @Override
    public void send() {
        // 실제 메일을 발송한다.
    }
}

public class ConsoleMailSender implements MailSender {
    @Override
    public void send() {
        // 메일 내용을 콘솔에만 출력한다.
    }
}

public class MailService {
    private final MailSender mailSender;
    
    public MailService(MailSender mailSender) {
        this.mailSender = mailSender;
    }
}
```

의존 관계 역전 원칙을 잘 지키는 코드이다. 의존성 주입을 통해 실제 구현체와의 의존성을 분리하였다. 만약 테스트 환경이라면 실제로 메일이 발송되어서는 안되기 때문에 `ConsoleMailSender` 를 사용하면 될 것이다. 운영 환경이라면 `RealMailSender` 를 사용하면 된다.

## 마무리

객체 지향 설계의 SOLID 원칙에 대해 살펴보았다. 객체 지향 설계의 '원칙' 이라는 말이 붙었을 정도이다. 간단한 프로그램을 만들더라도 항상 SOLID 원칙을 준수할 수 있도록 습관을 들이는 것이 좋을 것이다.