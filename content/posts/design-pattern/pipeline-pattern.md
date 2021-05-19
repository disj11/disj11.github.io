---
title: Pipeline Pattern (파이프라인 패턴)
date: 2019-07-09T23:33:00+09:00
menu:
  sidebar:
    name: Pipeline Pattern (파이프라인 패턴)
    identifier: pipeline-pattern
    parent: design-pattern
    weight: 10
---

## 개요

이니시스 빌링 결제 구현 중 `본인 인증 -> 빌링 키 발급 -> 빌링 키로 결제`
이렇게 세 가지 단계로 진행되는 파이프라인을 구성해야 했다. 본인 인증 결과가 빌링 키 발급의 input 이 되고, 빌링 키 발급의 결과가 빌링 키 결제의 input 이 되는 식이었다. 또 해당 파이프라인 전후로
다른 작업(결제 후 포인트 추가, 이메일 발송 등)의 추가 / 삭제가 간편해야 했다.

## Step Interface

파이프라인 각각의 단계에 해당하는 인터페이스

```java
public interface Step<I, O> {
    O process(I input);
}
```

## Pipeline Class

각각의 단계를 순차적으로 실행하는 클래스

```java
public class Pipeline<I, O> {
    private final Step<I, O> current;

    public Pipeline(Step<I, O> current) {
        this.current = current;
    }

    public <O2> Pipeline<I, O2> pipe(Step<O, O2> next) {
        return new Pipeline<>(input -> next.process(current.process(input)));
    }

    public O execute(I input) {
        return current.process(input);
    }
}
```

## Example

문자열 형태의 숫자를 int 형으로 변경 후 2를 곱하는 과정을 파이프로 구성해보자.

```java
public class ExamplePipeline {
    static class StringToIntStep implements Step<String, Integer> {
        public Integer process(String input) {
            return Integer.parseInt(input);
        }
    }

    static class CalculationStep implements Step<Integer, Integer> {
        public Integer process(Integer input) {
            return input * 2;
        }
    }

    public static void main(String[] args) {
        Pipeline<String, Integer> pipeline = new Pipeline<>(new StringToIntStep())
                .pipe(new CalculationStep());
        System.out.println(pipeline.execute("3")); // 6 출력
    }
}
```

## 참고

[The Pipeline design pattern (in java)](https://medium.com/@deepakbapat/the-pipeline-design-pattern-in-java-831d9ce2fe21)