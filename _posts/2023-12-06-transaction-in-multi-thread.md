---
title: 테스트 코드 - 멀티쓰레드 환경의 트랜잭션
description: 멀티쓰레드 환경이 필요한 테스트코드 작성해보기
author: changhyo
date: 2023-12-06 15:01 +0900
lastmod: 2023-11-28 15:01 +0900
categories: backend
tags: multiThread, transaction, testCode
---

멀티쓰레드 환경의 테스트 코드를 작성하면서 알게된 내용을 공유하려고 합니다. 
혼자 생각하고 정리한 내용이라 부정확한 내용이 포함되어 있을 수 있습니다. 만약 틀린 내용이 있다면 알려주시면 감사하겠습니다.

# 자주 활용하는 클래스
## ExecutorService
*	<a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html">공식문서</a>
*	Executor를 상속받고 있습니다. Executor는 Runnable 객체를 실행시키는 execute 메서드를 가지고 있습니다.
*	ExecutorService로 쓰레드 풀을 생성하고 해당 쓰레드 풀에서 꺼낸 쓰레드로 Runnable Task를 실행함으로써 병렬 프로그래밍을 실행할 수 있습니다.


## CountDownLatch
*	<a href="https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html">공식문서</a>
*	여러 쓰레드의 작업이 끝나는 걸 기다릴 수 있게 도와줍니다.

## TransactionSynchronizationManager
*	<a href="https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html">공식문서</a>
*	쓰레드별 리소스 및 트랜잭션 동기화를 위해 사용하는 클래스입니다.
*	isActualTransactionActive, getCurrentTransactionName, getCurrentTransactionIsolationLevel등 트랜잭션의 상태를 확인할 수 있는 유용한 메서드를 제공합니다.


## PlatformTransactionManager
*	<a href="https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/PlatformTransactionManager.html">공식문서</a>
*	트랜잭션 관리를 위한 인터페이스입니다.
*	직접 트랜잭션을 시작하고 commit 또는 rollback할 수 있습니다.
*	PlatformTransactionManager를 이용해 직접 트랜잭션을 관리하기 보다는 선언적(@Transactional)으로 트랜잭션을 사용하는 걸 권장합니다. 저도 테스트 환경에서 기능을 확인하는 용도로만 사용하고 있습니다.



# 알게된 사실
## 1. 새로운 쓰레드를 생성하는 쓰레드와 생성된 쓰레드는 아무런 관계가 없다

![](https://velog.velcdn.com/images/qwerty1434/post/a7beb2ff-b084-4158-83e3-53f603bf284e/image.png)

다음과 같이 쓰레드를 생성하면 생성된 쓰레드의 코드가 쓰레드를 생성하는 코드 `내부`에 위치하며, 생성하는 곳에서 변수를 넘겨줄 수 있기 때문에 관계를 가지는 것으로 생각할 수 있습니다. 

하지만 저는 아래와 같이 동작하는 걸 보고 관계가 없다고 판단했습니다.

(내부-외부의 관계를 가지지 않지만, 이해를 돕기 위해 내부와 외부라는 용어를 계속해서 사용하도록 하겠습니다)

### 테스트코드의 트랜잭션은 열려있더라도 내부에서 생성한 쓰레드에는 트랜잭션이 열려있지 않다
![](https://velog.velcdn.com/images/qwerty1434/post/3b540c6b-6fe7-44fa-9611-31b7bced254c/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/73e16460-f734-4471-84ac-49b2867f9b1d/image.png)

외부 쓰레드와 별개로 내부의 쓰레드는 트랜잭션이 활성화 되어있지 않습니다.

### 외부에서 연 내부 쓰레드에서의 작업은 @Transactional로 롤백되지 않는다
테스트 코드에서 @Transactional을 활용하면 해당 메서드에서의 작업을 롤백할 수 있습니다. 

이 작업은 다른 메서드에 있는 Transactional의 Propagation이 기본 옵션인 REQUIRED방식으로 동작하기 때문입니다. 즉, 개별 메서드의 트랜잭션이 테스트 코드의 트랜잭션에 합류했기 때문에 다같이 롤백이 가능합니다.

만약 REQUIRES_NEW옵션으로 메서드를 진행한다면, 테스트 코드에서 @Transactional을 선언했더라도 작업이 롤백되지 않습니다.

아래 간단한 코드로 테스트해볼 수 있습니다.

#### teamService 코드
```java
@Service
@RequiredArgsConstructor
public class TeamService {
    private final TeamRepository teamRepository;
    @Transactional(propagation = Propagation.REQUIRES_NEW) // REQUIES_NEW
    public void save() {
        Team team = Team.builder()
                .name("팀이름")
                .build();
        teamRepository.save(team);
    }
}
```

#### 테스트 코드
```java
@SpringBootTest
class multiThreadTest{
    @Autowired
    private TeamService teamService;
    
    @Transactional
    @Test
    void propagation_rollback_test() {
        teamService.save();
    }
}
```

![](https://velog.velcdn.com/images/qwerty1434/post/ca6565bc-6552-41f8-810c-be9f0d7b9a00/image.png)

테스트에서 생성한 Team이 사라지지 않고 DB에 계속 남아있습니다.


내부에서 생성한 쓰레드에서 트랜잭션 작업을 진행하더라도 이는 별도의 쓰레드 및 트랜잭션이기 때문에 합류가 불가능합니다. 합류가 안되기 때문에 롤백 역시 불가능합니다.
#### teamService 코드
```java
@Service
@RequiredArgsConstructor
public class TeamService {
    private final TeamRepository teamRepository;
    @Transactional // 기본 설정(REQUIRED) 활용
    public void save() {
        Team team = Team.builder()
                .name(UUID.randomUUID().toString().substring(0,8))
                .build();
        teamRepository.save(team);
    }
}
```

#### 테스트 코드
```java
@SpringBootTest
class multiThreadTest{
    @Autowired
    private TeamService teamService;
    
    @Transactional
    @Test
    void propagation_rollback_test() throws InterruptedException {
        final int count = 10;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(count);

        for (int i = 0; i < count; i++) {
            executorService.execute(() -> {
                teamService.save();
                latch.countDown();
            });
        }

        latch.await();

    }
}
```

![](https://velog.velcdn.com/images/qwerty1434/post/eabc9f76-0d22-4fac-a359-95079318b604/image.png)


> 그러므로 멀티 쓰레드 환경을 검증하는 테스트 코드를 작성할 때는 @Transactional보다 @AfterEach를 통한 teardown메서드에서 deleteAllInBatch를 사용하는 걸 권장드립니다.



### 내부에서 연 쓰레드의 Exception은 해당 테스트로 전파되지 않는다

![](https://velog.velcdn.com/images/qwerty1434/post/172ad439-2149-4b3f-8d1b-66cd1da6186e/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/9d57e92a-3a75-4035-9e36-50c8f7e4e3a5/image.png)

내부 쓰레드에서 Exception을 발생시켰지만, 이를 생성한 기존 쓰레드에는 해당 Exception이 전파되지 않아 ok가 정상적으로 출력됩니다.


## 2. 외부에서 create코드를 실행했지만 멀티쓰레드 코드에서 해당 객체를 찾을 수 없다
MySQL InnoDB의 기본 Isolation Level은 Repeatable Read입니다.
Repeatable Read는 자기보다 먼저 실행된 트랜잭션의 데이터만 조회하며, 자신보다 이후에 실행된 트랜잭션의 데이터는 언두 로그를 참고해 조회합니다.

이는 어디까지나 update의 얘기이고, create의 경우에는 다른 트랜잭션에 의해 추가된 레코드가 발견될 수도 있습니다. 이를 Phantom Read라고 합니다. 

하지만 MySQL의 InnoDB는 MVCC방식을 통해 Repeatable Read에서도 Phantom Read가 발생하지 않습니다.

그로 인해 외부에서 create를 실행하고 내부에서 이를 조회하거나, 카운팅하는 쿼리를 실행하면 값을 가져오지 못하게 됩니다.

![](https://velog.velcdn.com/images/qwerty1434/post/6acd9d21-914e-41f4-8630-f5357c409ff4/image.png)


![](https://velog.velcdn.com/images/qwerty1434/post/e6650c2e-3295-4ee4-aa09-7114f399f387/image.png)

문제를 해결하는 방법 중 하나는 별도의 트랜잭션에서 데이터를 삽입한 뒤 트랜잭션을 종료시켜버리는 겁니다. 트랜잭션이 종료되었기 때문에 다른 트랜잭션에서도 해당 값을 정상적으로 조회할 수 있습니다. (또는 테스트 코드 메서드에 @Transactional을 없애고 save가 자신의 트랜잭션에서 실행되고 곧바로 커밋되게 할 수도 있습니다)

![](https://velog.velcdn.com/images/qwerty1434/post/ddb6b3f7-f500-40c7-aa51-cd3cd64c9616/image.png)


![](https://velog.velcdn.com/images/qwerty1434/post/2203cef9-7192-4cb7-a2d8-74686d131dbf/image.png)

## 기타 
주제와 상관없는 내용이지만 위 내용을 테스트하는 과정에서 몰랐던 사실을 알게되어 같이 기록해두려 합니다.

### 트랜잭션 isolation은 어디에 설정해야 할까?

앞서 외부 create코드를 내부 쓰레드에서는 볼 수 없는 이유가 Repeatable Read와 MVCC때문이고, 이를 해결하기 위해 별도의 트랜잭션에서 데이터를 create하는 방법을 설명드렸습니다.

사실 이 방법 말고 create된 데이터를 다른 트랜잭션이 확인하는 방법이 하나 더 있습니다. 바로 트랜잭션 격리수준을 READ UNCOMMITTED로 낮추는 것입니다. (실제 서비스에서 이 방법을 쓰는건 좋지 못합니다.)

이때 `데이터를 저장하는 트랜잭션`과 `데이터를 읽는 트랜잭션`중 어느 쪽의 격리수준을 낮춰야 할까요?

#### 기본 세팅
이전에 했던 테스트와 유사한 환경입니다.

![](https://velog.velcdn.com/images/qwerty1434/post/b8721e59-b5ce-4f88-9826-3e64b67f3692/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/e328ee18-4293-4630-b0e7-8a102cc92b2f/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/07240d20-1313-44ee-9059-9fae21382782/image.png)


1. 테스트 코드 자체에 @Transactional이 선언되어 있습니다.
2. save와 read는 각각 @Transactional이 선언되어 있습니다.
3. save는 테스트 코드의 @Transactional에 합류해 테스트 코드가 끝날 때 커밋됩니다.
4. read는 별도의 쓰레드에서 실행되 테스트 코드의 @Transactional에 합류하지 못합니다. 또한 save의 트랜잭션이 커밋되지 않아 해당 데이터를 조회하지도 못합니다.

#### 데이터를 저장하는 트랜잭션의 격리수준을 READ UNCOMMITTED로
![](https://velog.velcdn.com/images/qwerty1434/post/c51def7e-8b39-4c8d-825c-d56ccbe8164e/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/b737820a-d703-4de7-bbf7-341ef894ba7a/image.png)

여전히 커밋되지 않은 데이터를 조회하지 못합니다.

#### 데이터를 읽는 트랜잭션의 격리수준을 READ UNCOMMITTED로
![](https://velog.velcdn.com/images/qwerty1434/post/46a5d9fa-59e1-44c5-8527-d84c9be93f51/image.png)

![](https://velog.velcdn.com/images/qwerty1434/post/d20f55fb-e76b-497f-aeb8-fa436c0b4d59/image.png)

커밋되지 않은 데이터를 조회할 수 있습니다.

저는 `데이터를 저장하는 트랜잭션`에 격리수준을 설정해 `이 데이터는 커밋되지 않은 상태에서 읽어도 좋습니다`를 다른 트랜잭션에게 알려주는 느낌이 아닐까라고 생각했었는데,
실제로 확인해본 결과로는 `데이터를 읽는 트랜잭션`에 격리수준을 설정해 `나는 커밋되지 않은 상태의 데이터도 다 읽어버릴꺼야`를 선언하는 느낌이었습니다.

> 격리수준은 데이터를 읽는 쪽을 기준으로 설정되어야 합니다.



# 예제
프로젝트에서 검증을 위해 사용한 테스트 코드 입니다.

## 쿠폰 동시성 문제 확인하는 테스트
```java
    @DisplayName("멀티쓰레드 환경에서도 동시에 쿠폰 발급을 요청해도 정해진 수량만큼의 발급이 보장된다")
    @Test
    void issueCouponInMultiThread() throws InterruptedException, ExecutionException {
        // given
        int limitCount = 100;
        int applicantsCount = 1000;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(applicantsCount);

        Future<Long> couponCreate = executorService.submit(() -> {
            Store store = createStore();
            storeRepository.save(store);

            Coupon coupon = couponCreator(store, limitCount);
            return couponRepository.save(coupon).getId();

        });

        final Long couponId = couponCreate.get();

        // when
        LongStream.rangeClosed(1L, applicantsCount)
                .forEach(userId ->
                    executorService.execute(() -> {
                        try {
                            couponService.downloadCoupon(userId,couponId,LocalDate.now());
                        } catch (Exception ignored) {
                        } finally {
                            latch.countDown();
                        }
                    })
                );

        latch.await();

        long issuedCouponCount = issuedCouponRepository.count();

        // then
        assertThat(issuedCouponCount).isEqualTo(limitCount);

    }
```

## 낙관락을 확인하는 테스트
```java
    @DisplayName("재고 수정이 동시에 들어올 경우 가장 처음 수정만 유효하고, 뒤이은 수정은 실패한다(낙관락)")
    @Test
    void OptimisticLockWithVersion() throws InterruptedException, ExecutionException {
        // given
        final int concurrentRequestCount = 3;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(concurrentRequestCount);
        final Long flowerId = 1L;

        Future<Long> storeCreate = executorService.execute(() -> {
            TransactionStatus status = txManager.getTransaction(new DefaultTransactionAttribute());
            Store store = createStore();
            storeRepository.save(store);
            FlowerCargoId flowerCargoId = createFlowerCargoId(store.getId(), flowerId);
            FlowerCargo flowerCargo = createFlowerCargo(flowerCargoId, 100L, "장미", store);
            flowerCargoRepository.save(flowerCargo);
            txManager.commit(status);
            return store.getId();
        });

        final Long storeId = storeCreate.get();


        LongStream.rangeClosed(1L, concurrentRequestCount)
                .forEach( idx -> executorService.execute(() -> {
                    try {
                        cargoService.plusStockCount(storeId,flowerId,idx);
                    } catch (Exception ignored) {
                    } finally {
                        latch.countDown();
                    }
                }));

        latch.await();
        FlowerCargoId flowerCargoId = createFlowerCargoId(storeId, flowerId);
        FlowerCargo result = flowerCargoRepository.findById(flowerCargoId).get();

        Assertions.assertThat(result.getVersion()).isNotEqualTo(0);
    }
```





# References
*	https://devoong2.tistory.com/entry/JPA-에서-낙관적-락Optimistic-Lock을-이용해-동시성-처리하기
*	https://donghyeon.dev/junit/2021/10/17/낙관적락-테스트하기/
*	https://mangkyu.tistory.com/299
*	https://tecoble.techcourse.co.kr/post/2022-11-07-mysql-isolation/
