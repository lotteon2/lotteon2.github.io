---
title: 5주차. Resilience4j에 대해
description: Resilience4j 개념 정리
author: changhyo
date: 2023-12-15 09:12 +0900
lastmod: 2023-12-15 09:12 +0900
categories: techoTalk
tags: resilience4j circuitbreaker
---

오늘의 주제는 Resilience4j입니다. 이 글은 [Resilience4j 공식문서](https://resilience4j.readme.io/)를 학습하면서 정리한 내용입니다.

# Resilience4j

Resilience4j는 Netflix의 Hystrix와 유사한 fault tolerance 라이브러리입니다. fault tolerance란 시스템의 일부가 장애가 나더라도 전체 시스템은 중단 없이 잘 동작할 수 있는 능력을 의미합니다. 이러한 기능은 각각의 시스템을 독립적으로 운용하는 MSA환경에서 한 시스템의 장애가 전체 시스템의 장애로 전파되는 걸 방지하기 위한 용도로 많은 곳에서 사용되고 있습니다.

사실 Resilience4j보다 CircuitBreaker라는 용어가 조금 더 익숙할 거라 생각됩니다. Resilience4j와 CircuitBreaker의 관계는 간단합니다. Resilience4j의 핵심 모듈 중 하나로 CircuitBreaker가 포함되어 있습니다. Resilience4j는 CircuitBreaker를 포함해 Bulkhead, RateLimiter, Retry, TimeLimiter, Cache라는 6개의 핵심 모듈을 가지고 있습니다. 

> CircuitBreaker는 다양한 분야에서 사용되는 범용적인 용어입니다. 저 같은 경우에는 이전에 금융을 공부할 때 서킷브레이커라는 용어를 먼저 들어봤었죠. Resilience4j의 CircuitBreaker역시 다양한 분야에서 활용되는 CircuitBreaker의 의미를 크게 벗어나지 않으니 배경 지식이 있으시다면 그것과 연관해 생각하셔도 좋을 거 같습니다.
> 

---

## 데코레이터 패턴

Resilience4j의 핵심 모듈은 대부분 유사한 형태를 가지고 있습니다. Registry를 등록해 사용하며, 방출된 Event를 Consumer할 수 있으며, Functional Interface를 Decorate할 수 있습니다.

이러한 공통점들 중 Decorate에 대해 간단히 살펴 보겠습니다. 데코레이터 패턴은 Resilience4j만의 특징이 아니라 개발에서 널리 사용되고 있는 디자인 패턴의 한 종류입니다.

데코레이터 패턴은 기존 클래스에 영향을 주지 않으면서 새로운 동작을 추가하고 싶을 때 사용하는 패턴입니다. 

간단한 예시를 살펴보겠습니다. 

원본객체 A가 존재합니다.

```java
public class A {
    public void sayHello() {};
}
```

이때 원본객체 A에 영향을 주지 않고 새로운 기능을 추가하기 위해 다음과 같은 데코레이터를 이용합니다.

```java
public class Decorator {
    public A a; // has a 관계
		
    public Decorator(A a) {
        this.a = a;
    }
    // 기존 기능은 원본객체에게 위임
    public void sayHello() {
        a.sayHello();
    }
    // 새로운 기능 정의
    public void sayWorld() {};
}
```

원본객체와 데코레이터 객체 모두 같은 인터페이스를 구현하도록 만들면 원본객체 대신 데코레이터를 사용할 수 있게 됩니다.

```java
public class A implements Inter{
    public void sayHello() {};
}
```

```java
public class Decorator implements Inter{
    public Inter a;
		
    public Decorator(Inter a) {
        this.a = a;
    }

    @Override
    public void sayHello() {
        a.sayHello();
    }
		
    public void sayWorld() {};
}
```

데코레이터 패턴의 장점은 상속 없이 클래스의 기능을 확장할 수 있다는 점입니다.

---

# 핵심 모듈

## CircuitBreaker

서킷브레이커란 호출의 성공 및 실패를 계산하며 실패율이 임계치를 초과하면 자동으로 `호출을 거부`하는 기술입니다.

서킷브레이커는 CLOSED, OPEN, HALF_OPEN의 세 가지 일반 상태와 DISABLED, FORCED_OPEN이라는 두 가지 특수 상태를 갖는 유한 상태 기계로 구현됩니다.

> 유한 상태 기계란?
> 한 번에 오로지 하나의 상태만 가질 수 있는 형태를 말합니다

서킷브레이커는 슬라이딩 윈도우를 사용해 호출 결과를 저장하고 집계합니다. 이때 개수 기반 슬라이딩 윈도우와 시간 기반 슬라이딩 윈도우를 선택할 수 있습니다. 

**개수 기반 슬라이딩 윈도우**

- N번의 요청 중 실패한 요청 또는 느린 호출의 횟수가 임계값을 넘으면 서킷브레이커가 OPEN됩니다.

**시간 기반 슬라이딩 윈도우**

- N초 동안의 요청 중 실패한 요청 또는 느린 호출의 횟수가 임계값을 넘으면 서킷브레이커가 OPEN됩니다.

기본적으로 모든 예외는 다 실패로 간주됩니다. 실패로 간주할 예외를 리스트로 정의하거나, 무시할 예외(무시한 예외는 실패, 성공 둘 중 무엇으로도 계산되지 않는다)를 리스트로 정의할 수도 있습니다. 또한 서킷 브레이커는 최소한의 측정치가 필요합니다. 최소한으로 필요한 호출 횟수가 10번이고 호출을 9번밖에 측정하지 않았다면, 9번 모두 실패했더라도 `서킷브레이커는 열리지 않습니다.`

서킷브레이커는 OPEN상태일 땐 `CallNotPermittedException`을 던져 호출을 거절합니다. 대기시간이 경과하면 서킷브레이커는 OPEN에서 HALF_OPEN으로 상태가 변경되고 설정한 횟수만큼의 호출만 받아 서비스가 정상적으로 복구되었는지 확인합니다. 실패 비율 또는 느린 호출 비율이 임계값 이상이면 OPEN으로, 실패 비율과 느린 호출 비율이 모두 임계치 미만이면 CLOSED로 상태를 변경합니다.

서킷브레이커는 DISABLED, FORCED_OPEN라는 특별한 상태를 가집니다. DISABLED 또는 FORCED_OPEN 상태일 때 서킷브레이커는 이벤트를 생성하지도 메트릭을 기록하지도 않습니다. 이 상태에서 벗어나려면 상태 전환을 트리거하거나 서킷브레이커를 리셋해야 합니다.

- DISABLED는 항상 접근을 허용합니다.
- FORCED_OPEN는 항상 접근을 거부합니다.

서킷브레이커의 특징 중 하나는 thread-safe하다는 점입니다. 원자성이 보장되며 특정 시점에는 오직 하나의 스레드만 상태나 슬라이딩 윈도우를 업데이트할 수 있습니다.

## Bulkhead

Bulkhead는 `동시 실행 횟수를 제한`합니다.

Resilience4j는 Bulkhead 패턴을 구현할 수 있는 두 가지 구현체를 제공합니다.

- semaphoreBulkhead : 세마포어를 사용합니다.
    - Hystrix와 달리 shadow(이름 뿐인) 쓰레드 풀 옵션을 제공하지 않습니다.
- FixedThreadPoolBulkhead : 유한 큐와 고정 쓰레드 풀을 사용합니다.

## RateLimiter

RateLimiter는 `단위 시간동안의 실행 횟수`를 제한합니다. 한마디로 호출 속도를 제한하는 것이죠.

제한된 요청은 단순히 거부하거나 또는 큐에 담아두었다가 추후에 실행하게 할 수도 있습니다.

## Retry

Retry는 실패시 재시도에 대한 구체적인 로직을 정의할 수 있도록 도와줍니다. 

최대 시도 횟수, 재시도할 때마다 기다리는 시간, 결과를 통한 재시도 여부 등을 설정할 수 있습니다.

## TimeLimiter

TimeLimiter는 요청에 대한 `타임아웃 설정`을 제한합니다. 이때 주어진 시간이 지났을 때 해당 Future를 취소시킬지 여부를 함께 설정할 수 있습니다.

## Cache

javax.cache.Cache를 이용한 캐싱 기능을 제공합니다.

---

# 활용 예시

Resilience4j의 여러 모듈은 사용 방법이 모두 비슷합니다. 따라서 CircuitBreaker의 활용 예시만 대표로 한번 살펴보도록 하겠습니다.

서킷브레이커 객체를 얻기 위해서는 기본적으로 `CircuitBreakerConfig > CircuitBreakerRegistry > CircuitBreaker`순으로 객체를 정의해야 합니다.

CircuitBreakerConfig는 서킷브레이커의 다양한 파라미터를 설정할 수 있습니다. 만약 Resilience4j가 기본적으로 제공하는 파라미터 설정으로 사용하려면 `CircuitBreakerConfig.ofDefaults()`라는 static메서드로 생성하면 됩니다.

```java
// 기본설정
CircuitBreakerConfig config = CircuitBreakerConfig.ofDefaults();

// 직접 설정하기
CircuitBreakerConfig customConfig = CircuitBreakerConfig.custom()
                                          ... // 다양한 설정
                                          .build();
```

다음으로 CircuitBreakerRegistry에 우리가 만든 CircuitBreakerConfig를 등록해 줍니다.

```java
// 빈 CircuitBreakerRegistry 만들기
CircuitBreakerRegistry registry = CircuitBreakerRegistry.ofDefaults();

// 특정 config를 넣으면서 CircuitBreakerRegistry만들기
CircuitBreakerRegistry registry2 = CircuitBreakerRegistry.of(config);

// CircuitBreakerRegistry에 새로운 config 추가
registry2.addConfiguration("name",config);
```

CircuitBreakerRegistry에 저장된 config설정으로 서킷브레이커 객체를 만들 수 있습니다.

```java
CircuitBreaker circuitBreaker = circuitBreakerRegistry
		.circuitBreaker("circuitBreakerName","configName");
```

CircuitBreakerRegistry에 등록하지 않고 CircuitBreakerConfig만으로 곧바로 CircuitBreaker객체를 만들 수도 있습니다.

```java
CircuitBreaker circuitBreaker = CircuitBreaker.of("circuitBreakerName",config);
```

획득한 CircuitBreaker객체의 executeRunnable을 통해 로직을 실행하면 해당 로직에 서킷브레이커를 적용시킬 수 있습니다.

```java
circuitBreaker.executeRunnable(() -> myMethod);
```

하지만 이러한 방식보다는 조금 더 간편한 AOP를 활용한 어노테이션 방식을 많이 이용합니다.

```java
@Configuration
public class Resilience4jConfig {
    @Bean
    public CircuitBreakerConfig circuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
                .slidingWindowSize(20)
                .failureRateThreshold(80)
                .build();
    }

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry(CircuitBreakerConfig circuitBreakerConfig) {
        return CircuitBreakerRegistry.of(circuitBreakerConfig);
    }

    @Bean
    public CircuitBreaker myCircuit(CircuitBreakerRegistry circuitBreakerRegistry) {
        return circuitBreakerRegistry.circuitBreaker("myCircuit");
    }

}
```

사용하려는 메서드에 @CircuitBreaker어노테이션을 적용하면 됩니다.

```java
@Component
public class myClass{
    @CircuitBreaker(
        name = "myCircuit",
        fallbackMethod = "myFallback"
    )
    public void myLogic() {}

    public void myFallback(CallNotPermittedException e) {}
}
```

fallbackMethod는 에러가 발생했을 때 해당 에러를 처리할 수 있게 도와줍니다. 꼭 서킷브레이커에서 발생하는 CallNotPermittedException이 아닌 메서드에서 발생하는 다른 예외를 처리하는 것도 가능합니다. 

fallbackMethod는 이를 호출하는 메서드와 비슷한 형태를 가져야 합니다.

1. 반환타입이 동일해야 합니다.
2. Throwable 혹은 Exception타입의 매개변수를 받아야 합니다.
3. 호출하는 메서드가 가지는 매개변수를 모두 가지거나 하나도 가지지 않아야 합니다. 

---

# References

- https://resilience4j.readme.io/
- https://onlyfor-me-blog.tistory.com/434
- [https://inpa.tistory.com/entry/GOF-💠-데코레이터Decorator-패턴-제대로-배워보자](https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-%EB%8D%B0%EC%BD%94%EB%A0%88%EC%9D%B4%ED%84%B0Decorator-%ED%8C%A8%ED%84%B4-%EC%A0%9C%EB%8C%80%EB%A1%9C-%EB%B0%B0%EC%9B%8C%EB%B3%B4%EC%9E%90)
- https://yeoonjae.tistory.com/entry/Project-Resilience4j-%EB%A1%9C-Circuit-Breaker-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0
- https://mein-figur.tistory.com/entry/resilience4j-circuit-breaker-example
