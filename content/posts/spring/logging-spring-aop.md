---
title: "Logging - Spring AOP"
date: 2020-08-06T18:40:00+09:00
menu:
  sidebar:
    name: Logging - Spring AOP
    identifier: logging-spring-aop
    parent: spring
    weight: 10
---


## 개요

[관점 지향 프로그래밍과 프록시](/posts/spring/spring-aop-proxy/)를 통하여 AOP가 무엇인지에 대해서 알아보았다. 이해를 돕기 위해 AOP를 사용하여 로그를 남기는 방법을 알아보자.

## AspectJ 기능 활성화

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
```

## Dependency 추가

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
    <version>2.3.2.RELEASE</version>
</dependency>
```

## Loggable 어노테이션을 생성

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

## Aspect 생성

```java
@Aspect
@Component
@Slf4j
public class LoggingAdvice {
    @Pointcut("@annotation(com.tutorial.logging.Loggable)")
    public void pointcut() {
    }

    @Around("pointcut()")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        ObjectMapper mapper = new ObjectMapper();

        String methodName = pjp.getSignature().getName();
        String className = pjp.getTarget().getClass().toString();

        Object[] args = pjp.getArgs();
        log.info("method invoked {} : {}() arguments : {}", className, methodName, mapper.writeValueAsString(args));

        Object rtnObject = pjp.proceed();
        log.info("{} : {}() Response : {}", className, methodName, mapper.writeValueAsString(rtnObject));
        return rtnObject;
    }
}
```

로그를 남기고 싶은 메서드에 `@Loggable` 어노테이션을 달면 끝.

## Pointcut, Advice

Pointcut 사용법이 궁금하다면?
[Pointcut tutorial](https://www.baeldung.com/spring-aop-pointcut-tutorial)

Advice 사용법이 궁금하다면?
[Advice tutorial](https://www.baeldung.com/spring-aop-advice-tutorial)