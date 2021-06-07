---
title: 생성자에 매개변수가 많다면 빌더를 고려하라
date: 2021-06-07T22:59:00+09:00
menu:
  sidebar:
    name: 생성자에 매개변수가 많다면 빌더를 고려하라
    identifier: builder
    parent: effective-java
    weight: 10
---

아래와 같이 식품 영양 정보를 표현하는 클래스가 있다. 이런 클래스에 생성자를 사용하여 값을 입력받으려면 어떻게 될까? 필수 매개변수만 받는 생성자, 필수 매개변수와 선택 매개변수 1개를 받는 생성자, 필수 매개변수와 선택 매개변수 2개를 받는 생성자... 이런 형태로 생성자를 늘려나가야 할 것이다. 이러한 방식을 점층적 생성자 패턴(telescoping constructor pattern)이라고 한다.

```java
public class NutritionFacts {
	private final int servingSize; // 필수
	private final int servings; // 필수
	private int calories; // 선택
	private int fat; // 선택
	private int sodium; // 선택
	private int carbohydrate; // 선택
}
```

## 1. 점층적 생성자 패턴 - 확장하기 어려움

```java
public class NutritionFacts {
	private final int servingSize; // 필수
	private final int servings; // 필수
	private final int calories; // 선택
	private final int fat; // 선택
	private final int sodium; // 선택
	private final int carbohydrate; // 선택
	
	public NutritionFacts(int servingSize, int servings) {
		this(servingSize, serving, 0);
  }

  public NutritionFacts(int servingSize, int servings, int calories) {
		this(servingSize, serving, calories, 0);
  }

	public NutritionFacts(int servingSize, int servings, int calories, int fat) {
		this(servingSize, serving, calories, fat, 0);
  }

  // ...생략
}
```

이런 점층적 생성자 패턴은 매개변수가 많아지면 클라이언트 코드를 작성하거나 읽기 어렵다. 타입이 같은 매개변수가 연달아 있는 경우, 매개변수의 순서를 바꿔 건네줘도 컴파일러는 알아채지 못하기 때문에 찾기 어려운 버그가 발생하기도 한다. 그렇다면 setter를 사용하는 자바빈즈 패턴(JavaBeans pattern)을 사용하면 어떨까?

## 2. 자바빈즈 패턴

```java
public class NutritionFacts {
	private int servingSize; // 필수
	private int servings; // 필수
	private int calories; // 선택
	private int fat; // 선택
	private int sodium; // 선택
	private int carbohydrate; // 선택
	
	public setServingSize(int val) { servingSize = val; }
	public setServings(int val) { servings = val; }
	public setCalories(int val) { calories= val; }
	public setFat(int val) { fat = val; }
	public setSodium(int val) { sodium = val; }
	public setcarbohydrate(int val) { carbohydrate = val; }
}
```

자바빈즈 패턴을 사용할 경우 점층적 생성자 패턴의 단점은 보이지 않는다. 하지만 객체 하나를 만들기 위해 메서드를 여러번 호출해야 하고, 객체가 생성되기 전까지는 일관성(consistency)이 무서진 상태가 될 수 있다.

```
NutritionFactors cocaCola = new NutritionFacts(); // 객체가 생성되었지만 필수 값이 설정되지 않음
cocaCola.setServingSize(240); // 필수값 설정
cocaCola.setServings(8); // 필수값 설정 (여기까지 설정되어야 정상적인 상태)
cocaCola.setCalories(100);
```

이로 인해 자바빈즈 패턴에서는 클래스를 불변으로 만들 수 없으며 스레드 안전성을 얻기 위해서는 추가 작업이 필요하다. 이러한 문제를 해결하기 위해 점층적 생성자 패턴의 안전성과 자바빈즈 패턴의 기독성을 겸비한 빌더 패턴(Builder Pattern)을 사용할 수 있다.

## 3. 빌터 패턴

```java
public class NutritionFacts {
	private int servingSize; // 필수
	private int servings; // 필수
	private int calories; // 선택
	private int fat; // 선택
	private int sodium; // 선택
	private int carbohydrate; // 선택

	public NutritionFacts(Builder builder) {
		servingSize = builder.servingSize;
		servings = builder.servings;
		calories = builder.calories;
		fat = builder.fat;
		sodium = builder.sodium;
		carbohydrate = builder.carbohydrate;
	}
	
	public static class Builder {
		// 필수 매개변수
		private final int servingSize;
    private final int servings;

		// 선택 매개변수
		private int calories;
		private int fat;
		private int sodium;
		private int carbohydrate;

		public Bilder(int servingSize, int servings) {
			this.servingSize = servingSize;
			this.servings = servings;
		}

		public Builder calories(int val) { calories = val; return this; }
		public Builder fat(int val) { fat = val; return this; }
		public Builder sodium(int val) { sodium = val; return this; }
		public Builder carbohydrate(int val) { carbohydrate = val; return this; }

		public NutritionFactos build() {
			return new NutritionFacts(this);
		}
  }
}
```

이런 방식을 메서드 호출이 흐르듯 연결된다는 뜻으로 플루언트 API(fluent API) 혹은 메서드 연쇄(method chaning)라 한다. 빌더 패턴을 사용한 클라이언트 코드는 읽고 쓰기 쉽고, NutritionFacts 클래스는 불변이 된다. 아래 코드는 이 코드를 사용하는 클라이언트 코드이다.

```
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
  .calories(100)
  .sodium(35)
  .carbohydrate(27)
  .build();
```

## 정리

생성자나 정적 팩터리가 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하는 게 낫다. 빌더는 점층적 생성자보다 클라이언트 코드를 읽고 쓰기가 쉽고, 자바 빈즈보다 훨씬 안전하다.
