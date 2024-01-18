---
title: 9주차. 락을 통한 동시성 제어 - 쿠폰편
description: 프로젝트 내에서 락을 통해 동시성을 제어한 경험에 대해
author: changhyo
date: 2023-01-19 13:00
lastmod: 2024-01-19 13:00
categories: techoTalk
tags: lock concurrency multiThread coupon
---

락을 통한 동시성 제어의 세 번째 이야기, **쿠폰편**입니다. 이번에는 레디스의 Set자료구조를 활용해 쿠폰의 동시성 문제를 해결한 경험을 공유드려 보겠습니다.

BB의 쿠폰은 다음과 같은 요구사항을 가지고 있습니다.

> '쿠폰은 한정 수량으로 발급해 선착순으로 지급되며, 같은 종류의 쿠폰은 1인당 1장만 발급 가능하다'
> 
- 100장의 쿠폰을 발급하기로 했다면 99장이 발급되어도 안되고, 101장이 발급되어도 안됩니다.
- 쿠폰은 먼저 신청한 100명의 고객에게 발급되어야 합니다. 99번째 순서로 신청한 고객이 쿠폰을 받지 못하고 101번째 순서로 신청한 고객이 쿠폰을 받는 경우가 발생하면 안됩니다.
- 유저1이 A쿠폰을 이미 발급받았다면 유저1은 더 이상 동일한 A쿠폰을 추가로 발급받을 수 없습니다.

'먼저 신청한 100명의 고객에게 발급되어야 한다'는 조건은 다음과 같은 제약사항이 존재합니다. 

- 발급대기, 발급취소 기능은 존재하지 않습니다. 100장의 쿠폰을 발급했을 때 95번 고객이 발급을 포기하고, 이로 인해 101번 고객이 쿠폰을 발급 받게 되는 경우는 존재하지 않습니다.
- 쿠폰을 발급하는 도중에 서버가 다운됐을 때 다시 실행한 서버는 발급 대상을 모릅니다. 100명의 선착순 고객 중 만약 51번 고객의 쿠폰을 발급하다가 서버가 다운됐다면, 서버는 기존에 51번부터 100번 고객을 기억하지 못하기 때문에 새롭게 50명의 고객의 신청을 받고 처리합니다.

첫 번째 제약사항은 BB쿠폰의 정책이기 때문에 큰 문제가 되지 않지만, 두 번째 제약사항은 `선착순`이라는 의미를 훼손하는 심각한 걸림돌일 수 있습니다. 이를 방지하기 위해 DLQ를 활용하는 등 별도의 failover전략이 필요하지만 현재 BB에는 해당 내용이 구현되어 있지 않습니다. 그러므로 아래 설명할 내용들 역시 중간에 서버가 다운되는 상황은 **고려되어 있지 않음**을 말씀드리며 다음 내용으로 넘어가도록 하겠습니다.

쿠폰의 정합성에 Set자료구조를 선택한 이유는 `같은 종류의 쿠폰에 대해 1인당 1장만 발급 가능하다`는 조건 때문입니다. 레디스의 Set 역시 Java의 Set과 동일하게 중복을 허용하지 않아 Set 자료구조에 발급받은 유저의 정보를 넣어두면 '1인 1발급' 조건을 손쉽게 확인할 수 있습니다.

쿠폰을 발급하는 로직은 생각보다 간단합니다. 

```java
issueCoupon() {
    if(duplicateIssue) throw Exception // (1)
    if(currentCnt < limitCnt) { // (2)
        issue coupon // (3)
        incr currentCnt // (4)
    }
}
```
1. 해당 쿠폰을 유저가 이미 발급 받았는지 확인합니다. 중복 발급이라면 발급 요청을 거부합니다.
2. 지금까지 발행된 쿠폰의 개수(currentCnt)와 발급하기로 한 쿠폰의 개수(limitCnt)를 비교합니다. 발행된 쿠폰의 개수가 발급하기로 한 쿠폰의 개수보다 적을 때만 쿠폰 발급이 가능합니다.
3. 쿠폰을 실제로 발급합니다.
4. 구현 방법에 따라 발행된 쿠폰의 개수를 직접 증가시켜야 할 수도 있습니다.

# 예제 코드

## 기본 코드

Spring은 RedisTemplate을 제공합니다. [RedisTemplate](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisTemplate.html)은 Redis의 데이터 접근을 손쉽게 할 수 있게 도와주는 클래스입니다.

![](https://velog.velcdn.com/images/qwerty1434/post/750cd32c-dde2-4b65-8cf5-c06c003df98b/image.png)

앞선 설명에서 Set자료구조를 이용한다 했으니 우리는 RedisTemplate의 opsForSet메서드를 활용하겠습니다.

![](https://velog.velcdn.com/images/qwerty1434/post/7f177574-13f4-4d45-8e5f-dadda6812131/image.png)

opsForSet중에서도 우리가 사용할 메서드는 다음과 같습니다.

- [isMember](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/SetOperations.html#isMember(K,java.lang.Object))(key, value) : key에 해당하는 set이 value를 포함하고 있는지 확인합니다.
- [add](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/SetOperations.html#add(K,V...))(key, value) : key에 해당하는 set에 value를 추가합니다.
- [size](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/SetOperations.html#size(K))(key) : key에 해당하는 set의 크기를 반환합니다.

RedisTemplate을 이용해 쿠폰 발급 로직을 구현해 봅시다.

```java
@Service
class CouponService {
    private final RedisTemplate<String,String> redisTemplate;
		
    void issueCoupon(String key, String userId, long limitCnt) {
        if(redisTemplate.opsForSet().isMember(key, userId)) throw new RuntimeException(); // (1)
        long currentCnt = redisTemplate.opsForSet().size(key); // (2)
        if(currentCnt < limitCnt) {
            redisTemplate.opsForSet().add(key, userId); // (3)
            saveToRDB; // (4)
        }
    }
}
```

1. isMember()로 해당 유저가 이미 쿠폰을 발급 받았는지 확인합니다.
2. size()로 현재 쿠폰을 발급 받은 인원이 몇 명인지 확인합니다.
3. 해당 유저를 발급자 명단(set)에 추가합니다. set의 add()는 set의 크기를 증가시키고, 다음 size()는 증가된 크기의 size를 반환하기 때문에 직접 currentCnt를 증가시키지 않아도 됩니다.
4. 발급 후 필요한 로직을 실행합니다. BB는 유저의 쿠폰 발급 정보를 Mysql DB에 저장하고 있습니다. 

이렇게 하면 모든게 다 해결될까요? 아쉽지만 이 코드는 동시성 문제를 전혀 **해결하지 못합니다**. 그 이유는 개수를 확인하고 값을 추가하는 연산이 `원자적`이지 못하기 때문입니다. 간단한 예시로 살펴보겠습니다.
![](https://velog.velcdn.com/images/qwerty1434/post/f66204b4-c101-497a-9425-6c3e3eaaaae3/image.png)

1. 발급 가능 개수가 4개인 쿠폰이 있고 3명(A,B,C)이 해당 쿠폰을 발급 받은 상태입니다.
2. 유저D와 유저E가 쿠폰 발급을 요청합니다. 두 유저 모두 지금까지 발행된 쿠폰의 개수가 3개라는 응답을 받습니다.
3. 결과적으로 두 유저 모두 쿠폰 발급에 성공하며 최종적으로 5명의 유저에게 쿠폰이 지급됩니다.

레디스가 Single Thread로 동작해 동시성 문제를 해결하기 좋은 건 맞지만, 단순히 레디스를 사용하기만 해서 동시성 문제를 해결할 수 있는 건 아닙니다. 위 예시처럼 두 연산 중간에 다른 연산이 실행될 수 있기 때문에 이를 방지하려면 Atomic한 연산을 실행해야 합니다. 레디스가 제공하는 [INCR](https://redis.io/commands/incr/), [SETNX](https://redis.io/commands/setnx/)등의 명령어가 바로 원자적인 연산입니다. 또 다른 방법으로 레디스의 트랜잭션을 이용하는 방법도 있습니다.

## Redis Transaction

레디스 트랜잭션은 MULTI와 EXEC 명령어를 통해 실행됩니다. 만약 RDB의 트랜잭션과 동일하게 생각해 'MULTI와 EXEC사이의 명령어가 트랜잭션으로 묶이게 된다'고 단순히 생각하면 다음과 같이 코드를 **잘못 설계**할 수 있습니다.

```java
void issueCoupon(String key, String userId, long limitCnt) {
    try{
        operations.multi(); // 트랜잭션 시작
        long currentCnt = redisTemplate.opsForSet().size(key); // (1)
        if(currentCnt < limitCnt) {
            redisTemplate.opsForSet().add(key,userId); // (2)
            saveToRDB;
        }
        operations.exec(); // 트랜잭션 커밋
    } catch (Exception e) {
        operations.discard(); // 트랜잭션 롤백
    }
}
```

이 코드가 실패하는 이유는 레디스 트랜잭션이 MULTI가 시작된 뒤부터 EXEC를 실행하기 전까지 발생한 연산을 `단순히 쌓아두기만` 하는 방식으로 동작하기 때문입니다. 즉, 위 코드에서 (1)과 (2)연산은 아직 실행되지 않았고, EXEC를 만나는 시점에 비로소 두 연산이 `함께 실행`됩니다. 결과적으로 위 코드의 currentCnt는 원하는 값을 전달받을 수 없습니다.  또한 MULTI로 묶인 연산 중 일부가 실패했다고 해서 앞서 실행한 연산이 롤백되지 않습니다. 이처럼 레디스의 트랜잭션은 RDB의 트랜잭션과 유사하면서도 다른 부분이 존재하기 때문에 [공식문서](https://redis.io/docs/interact/transactions/)를 읽어보시는 걸 추천드립니다!

저는 레디스의 트랜잭션을 사용하기 위해 다음과 같이 코드를 설계했습니다.

```java
@Component
@RequiredArgsConstructor
public class RedisOperation {
    private final RedisTemplate<String, String> redisTemplate;
		
    public Object countAndSet(String key, String value) {
        return redisTemplate.execute(new SessionCallback<>() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                // (1)
                operations.multi();
                redisTemplate.opsForSet().add(key, value);
                redisTemplate.opsForSet().size(key);
                return operations.exec();
            }
        });
    }
}
```

```java
void issueCoupon(String key, String userId, long limitCnt) {
    List<Long> result = (List) redisOperation.countAndSet(redisKey, redisValue); // (1)

    Long currentCnt = result.get(1);
    if(issueCount > limitCnt) {
        redisTemplate.opsForSet().remove(key,userId); // (2)
        return;
    }
    saveToRDB;
}
```

1. 트랜잭션으로 개수를 확인하는 연산과 set에 값을 추가하는 연산을 함께 실행합니다. 이 경우 쿠폰의 발급 수량과 상관없이 `일단 유저를 set에 추가`합니다.
2. 방금 트랜잭션에서 진행한 발급이 만약 초과발급이었다면 `해당 유저를 set에서 제거`하고 RDB에도 insert하지 않습니다. 그렇지 않다면 RDB에 insert를 실행합니다.

위와 같은 형태로 코드를 짜고 테스트 코드를 실행했을 때 무사히 통과해서 동시성 이슈를 해결한 줄 알았지만, 다시 코드를 살펴보던 중 문제가 있음을 알게 되었습니다. 바로 `countAndSet연산과 if조건의 remove연산이 Atomic하지 않다`는 점이었습니다. 두 연산 사이에 다른 연산이 개입하거나, 또는 서버가 다운되어 remove연산을 수행하지 못하게 되면 쿠폰을 정확한 수량만큼 발급하지 못하는 문제가 발생할 수 있습니다.

> 싱글 쓰레드인 레디스를 사용한다고 해서 반드시 동시성 문제를 해결할 수 있는게 아닌 것처럼, 단순히 트랜잭션을 사용했다고 해서 반드시 동시성 문제를 해결할 수 있는게 아닌 것이죠.
> 

위와 같은 문제점을 발견하고 난 뒤 Set을 사용하는 게 정말 맞는지에 대해 처음부터 다시 고민해봤습니다. Set을 사용한 이유는 중복 발급 여부를 확인하기 위해 RDB를 매번 조회하는 걸 원치 않았기 때문인데, 사실 [이러한 조회가 성능에 미치는 영향은 미미하다](https://www.inflearn.com/questions/59250/%EC%95%88%EB%85%95%ED%95%98%EC%84%B8%EC%9A%94-unique-index-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%80%EC%9E%A5%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C-%EC%A7%88%EB%AC%B8%EB%93%9C%EB%A6%BD%EB%8B%88%EB%8B%A4)고 말하기도 합니다. 하지만 저는 Set을 활용할 수 있는 방법을 조금 더 찾아보기로 했습니다.

## Lua Script

최종적으로 제가 선택한 방법은 Lua Script를 작성하는 방법입니다. 사용자가 정의한 스크립트를 레디스에 전달하면 이를 실행해주는 방식인데, 이때 `레디스는 Lua Script에 정의된 명령들을 원자적으로 실행됨을 보장`해 줍니다. - [레디스 공식문서](https://redis.io/docs/interact/programmability/eval-intro/)

![](https://velog.velcdn.com/images/qwerty1434/post/7b133ab5-9dfe-4252-8f7e-e0eb4b82b3d8/image.png)

스크립트는 다음과 같습니다.

```lua
local key = KEYS[1]
local value = ARGV[1]
local limitCnt = tonumber(ARGV[2])
local currentCnt = redis.call('SCARD', key)

if currentCnt <= limitCnt then
    redis.call('SADD', key, value)
    return true
else
    return false
end
```

문법이 다를 뿐 초반에 설계했던 코드와 동일합니다. 처음에는 SCARD(size명령어)와 SADD(add명령어)가 원자성을 보장하지 못했었지만 지금은 Lua Script로 작성해 전달하기 때문에 두 명령어는 원자성을 보장받을 수 있습니다. 

# 최종 코드

Lua Script를 활용해 최종적으로 BB프로젝트에 사용한 코드는 다음과 같습니다. 실제 프로젝트 코드를 그대로 가져오다 보니 앞에서 설명하지 않은 코드들이 일부 함께 등장하는 점 양해 부탁드립니다.

**RedisLuaScriptExecutor.interface**

```java
public interface RedisLuaScriptExecutor {
    Object execute(String script, String key, Object... args);
}
```

**CouponLockExecutor.class**

```java
@Component
@RequiredArgsConstructor
public class CouponLockExecutor implements RedisLuaScriptExecutor{

    private final RedisTemplate<String,String> redisTemplate;

    @Override
    public Boolean execute(String script, String key, Object... args) {
        RedisScript<Boolean> redisScript = new DefaultRedisScript<>(script, Boolean.class);
        return redisTemplate.execute(redisScript, Collections.singletonList(key), args[0], String.valueOf(args[1]));
    }

}
```

**LockScript.class**

```java
public class LockScript {
    public static final String script = "local key = KEYS[1]\n" +
            "local value = ARGV[1]\n" +
            "local limitCnt = tonumber(ARGV[2])\n" +
            "local currentCnt = redis.call('SCARD', key)\n" +
            "if currentCnt <= limitCnt then\n" +
            "    redis.call('SADD', key, value)\n" +
            "    return true\n" +
            "else\n" +
            "    return false\n" +
            "end";
}
```

**CouponIssuer.class**

```java
@Component
public class CouponIssuer {
    ... 
    public IssuedCoupon issueCoupon(Coupon coupon, Long userId, String nickname, String phoneNumber, LocalDate issueDate) {
        if(coupon.getIsDeleted()) throw new DeletedCouponException();
        if(coupon.isExpired(issueDate)) throw new ExpiredCouponException();
		
        String redisKey = makeRedisKey(coupon);
        String redisValue = userId.toString();
        Integer limitCnt = coupon.getLimitCount();
        if(isDuplicated(redisKey, redisValue)) throw new AlreadyIssuedCouponException();
		
        boolean issuable = (Boolean)redisLuaScriptExecutor.execute(LockScript.script, redisKey, redisValue, limitCnt);
        if(issuable) {
            return issuedCouponRepository.save(makeIssuedCoupon(coupon,userId,nickname,phoneNumber));
        }
        throw new CouponOutOfStockException();
    }
    ...
}
```

# 기타

동시성 문제와 직접적인 관련은 없지만 쿠폰 시스템을 설계하며 고민했던 부분에 대해 추가로 소개드리려 합니다.

## 쿠폰 ttl 설정과 Dummy 데이터

쿠폰에는 유효 기간이 존재합니다. 그리고 쿠폰의 발급자를 담고 있는 Set데이터 역시 쿠폰의 유효기간이 만료됨에 따라 함께 레디스에서 사라져야 합니다. 레디스는 [TTL](https://redis.io/commands/ttl/)이라는 설정을 통해 일정 시간이 지난 데이터를 삭제해 줍니다. 

당연하지만 TTL설정을 위해서는 Set데이터가 레디스에 존재해야 합니다. 하지만 레디스의 Set은 자바처럼 데이터가 없는 `빈 자료구조를 선언하는 게 불가능`합니다. 가게에서 쿠폰을 생성하는 시점에 TTL을 설정하길 원했지만, 이 시점에는 Set데이터가 레디스에 존재하지 않습니다. Set데이터가 레디스에 저장되는 시점은 바로 최초의 1인이 쿠폰을 발급 받는 시점이기 때문입니다.

유저가 쿠폰을 발급받을 때 TTL을 설정한다는 얘기는 매번 새롭게 TTL을 갱신해줘야 함을 의미합니다.

```java
issueCoupon(String key, String userId, LocalDate expirationDate) {
    redisTemplate.opsForSet().add(key,value)
    redisTemplate.expireAt(key, Date.valueOf(expirationDate))
}
```

물론 최초 발급을 확인해 TTL을 한번만 설정하는 것도 가능하긴 합니다.

```java
issueCoupon(String key, String userId, LocalDate expirationDate) {
    redisTemplate.opsForSet().add(key,value)
    int size = redisTemplate.opsForSet().size(key)
    // 최초 한번만 ttl 설정
    if(size <= 1) {
        redisTemplate.expireAt(key, Date.valueOf(expirationDate))
    }
}
```

하지만 두 방법 모두 마음에 들지 않았고, 저는 `쿠폰을 등록함과 동시에 Dummy데이터를 넣은 Set자료구조를 생성`하는 방법으로 로직을 구현했습니다.

```java
createCoupon() {
    Coupon coupon = couponCreator.create()

    redisTemplate.opsForSet().add(coupon.id, DUMMY_DATA) // (1)
    redisTemplate.expireAt(coupon.id, coupon.expirationDate) // (2)
}
```

1. 쿠폰을 생성할 때 더미를 가지는 Set데이터를 생성합니다.
2. 레디스에 Set데이터가 존재하기 때문에 TTL을 설정할 수 있습니다. 

100개의 수량을 발급하기로 한 쿠폰의 Set데이터에는 더미 데이터를 포함해 총 **101개**의 데이터가 들어가게 됩니다. 이러한 내용은 설계자가 아닌 다른 개발자가 봤을 때 오해의 소지가 있다고 판단했고, 그 이유를 코드 내 주석으로 설명해 두었습니다.
![](https://velog.velcdn.com/images/qwerty1434/post/9168a5dc-a489-4102-ab99-a200ab05ab29/image.png)

# 마치며

지금까지 BB프로젝트를 진행하면서 동시성과 관련해 겪었던 고민들, 그리고 해당 내용을 어떻게 해결했는지 살펴봤습니다. 사실 Redis를 이용한 분산락은 이미 Redisson, Lettuce 등으로 잘 구현되어 있어 락을 활용하는 것 자체는 어렵지 않게 느껴졌습니다. 다만 제대로 이해하지 않고 사용했을 때 생각지 못한 곳에서 문제가 발생할 수 있다는 점, 그리고 `내 시스템의 어느 부분에 어떻게 락을 활용해야 할지`를 잘 고민하는 게 더 중요하다는 걸 이번 프로젝트를 통해 배울 수 있었습니다.

처음 얘기했던 것처럼 지금 제 코드가 완벽한 해결 방안이 아니며, 분명히 틀린 부분이 여럿 존재할 거라고 생각됩니다. 정답이 아니라 '이 사람은 이렇게 생각했구나', '이 사람은 이 부분을 이렇게 설계했구나' 정도의 **참고용**으로 봐주시면 감사하겠습니다 :)

# References

- [https://techblog.gccompany.co.kr/redis-kafka를-활용한-선착순-쿠폰-이벤트-개발기-feat-네고왕-ec6682e39731](https://techblog.gccompany.co.kr/redis-kafka%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EC%84%A0%EC%B0%A9%EC%88%9C-%EC%BF%A0%ED%8F%B0-%EC%9D%B4%EB%B2%A4%ED%8A%B8-%EA%B0%9C%EB%B0%9C%EA%B8%B0-feat-%EB%84%A4%EA%B3%A0%EC%99%95-ec6682e39731)
- [https://www.inflearn.com/questions/889653/redis-를-이용한-분산-락-구현의-성능-관련-질문](https://www.inflearn.com/questions/889653/redis-%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%B6%84%EC%82%B0-%EB%9D%BD-%EA%B5%AC%ED%98%84%EC%9D%98-%EC%84%B1%EB%8A%A5-%EA%B4%80%EB%A0%A8-%EC%A7%88%EB%AC%B8)
- https://tecoble.techcourse.co.kr/post/2023-08-16-concurrency-managing/
- https://velog.io/@hgs-study/redis-sorted-set
- https://dkswnkk.tistory.com/712
- https://techblog.woowahan.com/2709/
- https://f-lab.kr/blog/redis-command-for-atomic-operation
- https://dev.gmarket.com/69
- https://djlee118.tistory.com/122
- https://channel.io/ko/blog/tech-backend-be-detective-office
