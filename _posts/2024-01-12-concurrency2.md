---
title: 8주차. 락을 통한 동시성 제어 - 재고편
description: 프로젝트 내에서 락을 통해 동시성을 제어한 경험에 대해
author: changhyo
date: 2023-01-12 11:12
lastmod: 2024-01-12 11:12
categories: techoTalk
tags: lock concurrency multiThread coupon
---

락을 통한 동시성 제어의 두 번째 이야기, **재고편**입니다. 이번 글에서는 Redisson을 이용한 분산락으로 어떻게 재고의 동시성 문제를 해결했는지 공유드려 보겠습니다.

Redis를 이용한 분산락은 Redisson 또는 Lettuce를 통해 손쉽게 사용할 수 있습니다. Redisson과 Lettuce에 대한 설명은 잘 정리된 글이 많기 때문에 간단히 살펴보고 넘어가겠습니다.

> Lettuce는 atomic한 setnx명령을 통해 분산락을 구현할 수 있습니다. spin lock방식으로 retry로직을 개발자가 직접 작성해줘야 합니다.
Redisson은 pub/sub방식으로 다른 쓰레드에게 락의 해제를 알리며 Lettuce와 달리 별도의 Retry로직을 개발자가 작성해주지 않아도 됩니다.
> 

많은 사람들이 pub/sub방식이 spin lock방식보다 redis에 가하는 부담이 적고, Timeout설정을 손쉽게 할 수 있다는 이유로 Redisson을 선택합니다. 저 역시 동일한 이유로 Redisson을 선택했습니다. 

살펴볼 내용과 관련된 재고의 요구사항은 다음과 같습니다. 

> '재고는 주문이 발생했을 때 차감되며, 하나의 주문에는 `여러 가게의 여러 상품`이 포함될 수 있다'
> 

지금부터 해당 요구사항을 만족시키면서 어떻게 동시성을 보장했는지 살펴보겠습니다.

# 예제 코드

설명에 등장하는 코드는 이해를 돕기 위한 코드로 실제 프로젝트에서 작성한 코드와는 차이가 있습니다.

## 하나의 재고만 차감할 때

Redisson을 통한 락을 사용하는 코드의 형태는 일반적으로 아래와 같습니다.

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final StockRepository stockRepository;
    private final RedissonClient redissonClient;

    @Transactional
    public void subtractStock(Long storeId, Long productId, Long productCount) {
        RLock lock = redissonClient.getLock(storeId+"::"+productId); // (1)
        try {
            boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS); // (2)
            if(!available) {
                // 락 획득 실패 (3)
                throw new RuntimeException();
            } else {
                // 락 획득 성공 (3)
                stockRepository.subtract(storeId, productId, productCount);
            }
        } catch(InterruptedException e) {
        } finally() {
            lock.unlock(); // (4)
        }
    }
}
```

1. 획득하려는 락을 정의합니다. 여기서는 StoreId와 productId를 조합해 락을 생성했습니다.
2. 락 획득을 시도합니다. 위 예제는 5초간 락 획득을 시도하며, 락을 획득한 뒤 1초가 지나면 자동으로 락이 해제됩니다.
3. 락 획득에 성공했을 때, 실패했을 때 동작할 로직을 정의합니다.
4. 락을 해제합니다.

충분히 직관적이고 좋은 코드지만 락을 관리하는 로직과 재고를 차감하는 로직이 엉켜있는 점이 아쉽습니다. 아래와 같이 락을 관리하는 코드와 재고를 차감하는 코드의 Layer를 분리하고 락을 획득한 뒤 재고를 차감하는 로직을 호출하는 방식으로 코드를 분리할 수 있습니다.

```java
@Component
@RequiredArgsConstructor
public class MyFacade{
    private final MyService myService;
    private final RedissonClient redissonClient;

    public void subtractStock(Long storeId, Long productId, Long productCount) {
        RLock lock = redissonClient.getLock(StoreId+"::"+productId);
        try {
            boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
            if(!available) {
            } else {
                myService.subtractStock(storeId, productId, productCount);
            }
        } catch(InterruptedException e) {
        } finally() {
            lock.unlock();
        }
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final StockRepository stockRepository;

    @Transactional
    public void subtractStock(Long storeId, Long productId, Long productCount) {
        stockRepository.subtract(storeId, productId, productCount);
    }
}
```

이렇게 코드를 분리하면 가독성 말고도 `트랜잭션 범위가 짧아진다`는 장점이 있습니다. 커넥션 풀을 이용하는 방식에서 하나의 트랜잭션이 오랫동안 커넥션을 점유하면 그만큼 나머지 쓰레드에서 커넥션을 획득하는 시간이 늦어지고, 이는 시스템 전체의 응답시간에 악영향을 미치게 될 수도 있습니다. 그러므로 트랜잭션의 범위는 **필요한 만큼만, 가능하면 짧게** 유지시켜 주는게 좋습니다.

락을 해제하는 부분도 약간의 수정이 필요합니다. 위 코드는 멀티쓰레드 환경의 테스트(관련 내용은 [테스트 코드 - 멀티쓰레드 환경의 트랜잭션](https://lotteon2.github.io/posts/transaction-in-multi-thread/) 편을 참고해 주세요)를 진행해보면 원하는 결과를 얻지 못합니다.

이유는 A쓰레드에서 생성한 락을 B쓰레드에서 닫는게 가능하기 때문인데요. 그래서 아래와 같이 해당 쓰레드에서 생성한 락만 해제하도록 코드를 변경해줘야 합니다. 

```java
if(lock.isLocked() && lock.isHeldByCurrentThread()) {
    lock.unlock();
}
```

## 하나의 가게에서 주문한 여러 상품의 재고를 차감할 때

여러 상품의 재고를 차감하는 요청이 들어왔을 때 아래처럼 처리하는 로직을 떠올리기 쉽습니다.

```java
@Component
@RequiredArgsConstructor
public class MyFacade{
    private final MyService myService;

    public void subtractStock(Long storeId, List<Long> productIds, List<Long> productCounts) {
        for(int i=0; i < productIds.length(); i++) {
            Long productId = productIds.get(i);
            Long productCount = productCounts.get(i);
            RLock lock = redissonClient.getLock(storeId+"::"+productId);
            try {
                boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
                if(!available) {
                } else {
                    myService.subtractStock(storeId, productId, productCount);
                }
            } catch(InterruptedException e) {
            } finally() {
                if(lock.isLocked() && lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }		
        }
    }
}
```

위 코드는 상품별로 락을 획득하고, 재고 차감에 대한 요청도 상품 개별 단위로 요청하고 있습니다. 이렇게 설계하면 재고 차감 요청이 `하나의 트랜잭션으로 묶이지 않아` 문제가 발생했을 때 다같이 롤백되지 못합니다.

그래서 저는 차감해야 할 상품 정보를 모두 myService에게 넘기는 방식으로 코드를 구현했습니다. 락의 범위가 `가게의 상품`에서 `가게`로 증가했지만 성능 저하는 거의 없을 것으로 판단됩니다.

```java

@Component
@RequiredArgsConstructor
public class MyFacade{
    private final MyService myService;

    public void subtractStock(Long storeId, List<Long> productIds, List<Long> productCounts) {
        // 가게 단위로 락을 설정합니다.
        RLock lock = redissonClient.getLock(storeId);
        try {
            boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
            if(!available) {
            } else {
                // 상품들을 모두 한번에 전달합니다.
                 myService.subtractStock(storeId, productIds, productCounts);
            }
        } catch(InterruptedException e) {
        } finally() {
            if(lock.isLocked() && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

MyFacade에서 전달하는 데이터 형식에 맞춰 MyService의 코드도 수정해 줍니다.

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final StockRepository stockRepository;
  
    @Transactional
    public void subtractStock(Long storeId, List<Long> productIds, List<Long> productCounts) {
        for(int i=0; i < productIds.length(); i++) {
            Long productId = productIds.get(i);
            Long productCount = productCounts.get(i);
            stockRepository.subtract(storeId, productId, productCount);
        }
    }
}
```

주문으로 들어온 `모든 상품에 대한 재고 차감을 하나의 트랜잭션에서 시도`하기 때문에 AllorNothing을 보장받을 수 있게 되었습니다.

## 여러 가게에서 주문한 여러 상품의 재고를 차감할 때

위에서 살펴본 '하나의 재고'에서 '하나의 가게'로 확장될 때 발생한 문제와 완전히 동일한 상황입니다. 이전 방식대로 가게 단위로 락을 획득하고 가게 단위로 재고 차감을 시도하면 '하나의 주문 내에서 B가게의 재고를 차감하다가 문제가 발생했을 때 이전에 차감했던 A가게의 재고를 함께 롤백할 수 없는' 문제가 발생합니다.

뾰족한 방법이 떠오르지 않으니 우선 직관적으로 할 수 있는 방법을 시도해 봅시다. 가장 처음 설계했던 것처럼 락을 획득하고 해제하는 작업을 트랜잭션 내부에서 함께 진행하면 어쨌든 요구사항을 만족시킬 수 있습니다.

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final StockRepository stockRepository;
  
    @Transactional
    public void subtractStock(Map<Long,List<ProductDto>> req) {
        for (Long storeId : req.keySet()) {
            List<ProductInfoDto> value = req.get(storeId);
            List<Long> productIds = value.getProductIds();
            List<Long> productCounts = value.getProductCounts();
            for(int i=0; i < productIds.length(); i++) {
                Long productId = productIds.get(i);
                Long productCount = productCounts.get(i);
    		
                RLock lock = redissonClient.getLock(storeId+"::"+productId);
                try {
                    boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
    						
                if(!available) {
                } else {
                    myService.subtractStock(storeId, productId, productCount);
                }
                } catch(InterruptedException e) {
                } finally() {
                    if(lock.isLocked() && lock.isHeldByCurrentThread()) {
                        lock.unlock();
                    }
                }
            }
    
        }
    }
}
```

지금껏 트랜잭션 범위와 코드의 가독성에 대한 많은 얘기를 했지만 결국 처음 방식과 동일한 코드로 돌아와버렸습니다 :(

트랜잭션의 범위를 줄이는 방법은 찾지 못했지만 아래와 같은 방법으로 코드를 설계하면 코드의 가독성은 높여줄 수 있습니다. Service Layer의 subtrackStock의 트랜잭션은 기본 설정으로 Propagation  설정이 REQUIRED라서 상위 트랜잭션에 합류합니다. Facade Layer에 트랜잭션을 설정함으로써 개별적으로 실행되는 Service Layer의 subtrackStock작업을 하나의 트랜잭션으로 묶을 수 있습니다. 

```java

@Component
@RequiredArgsConstructor
public class MyFacade{
    private final MyService myService;

    @Transactional
    public void subtractStock(Map<Long,List<ProductDto>> req) {
        for (Long storeId : req.keySet()) {
            List<ProductInfoDto> value = req.get(storeId);
            List<Long> productIds = value.getProductIds();
            List<Long> productCounts = value.getProductCounts();

            RLock lock = redissonClient.getLock(storeId);
            try {
                boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
				
                if(!available) {
                } else {
                    myService.subtractStock(storeId, productIds, productCounts);
                }
            } catch(InterruptedException e) {
            } finally() {
                if(lock.isLocked() && lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        }
    }
}
```

참고로 저는 Facade계층에 SQS알림을 전송하는 로직이 포함되어 있어 위와 같은 방식으로 코드를 변경하지는 않았습니다.

# 기타

## JPA의 변경감지 사용 시 주의사항

마지막으로 트랜잭션 내부에서 락을 획득하는 방식을 사용할 때 주의해야 할 사항을 하나 소개 드리려 합니다. 그건 바로 DirtyChecking을 활용한 Update를 진행할 때 `업데이트가 반영되기 전에 락이 먼저 해제`되어 원하는 결과를 얻지 못할 가능성이 존재합니다.

설명을 위해 초반에 사용했던 코드를 다시 가져왔습니다.

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final StockRepository stockRepository;
    private final RedissonClient redissonClient;

    @Transactional
    public void subtractStock(Long storeId, Long productId, Long productCount) {
        RLock lock = redissonClient.getLock(storeId+"::"+productId);
        try {
            boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);
            if(!available) {
                throw new RuntimeException();
            } else {
                stockRepository.subtract(storeId, productId, productCount);
            }
        } catch(InterruptedException e) {
        } finally() {
            if(lock.isLocked() && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
        // (1)
    } // (2)
}
```

전파될 상위 트랜잭션이 존재하지 않는다고 가정했을 때 변경감지에 의해 업데이트된 결과가 반영되는 시점은 (2)입니다. 하지만 락은 try문을 벗어날 때 즉시 해제되기 때문에 DB에 데이터는 커밋되지 않았지만 다른 쓰레드가 락을 획득할 수 있는 시점이 존재합니다. (1) 위치에 thread.sleep()을 걸고 코드를 실행하면 문제점을 명확히 확인해볼 수 있습니다.

문제를 해결하는 방법은 크게 두 가지입니다. 첫 번째는 이전에 했던 것처럼 락을 관리하는 코드와 재고를 차감하는 코드를 분리하는 방법입니다. 이 경우 트랜잭션이 종료되어 DirtyChecking이 실행된 뒤 락을 해제하기 때문에 문제가 발생하지 않습니다. 두 번째는 DirtyChecking을 사용하지 않는 방법입니다. jpql코드로 직접 Update를 실행하면 즉시 쿼리가 실행되기 때문에 위와 같은 문제를 피할 수 있습니다.

# References

- https://channel.io/ko/blog/distributedlock_2022_backend
- [https://velog.io/@znftm97/동시성-문제-해결하기-V1-낙관적-락Optimisitc-Lock-feat.데드락-첫-만남](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V1-%EB%82%99%EA%B4%80%EC%A0%81-%EB%9D%BDOptimisitc-Lock-feat.%EB%8D%B0%EB%93%9C%EB%9D%BD-%EC%B2%AB-%EB%A7%8C%EB%82%A8)
- [https://velog.io/@znftm97/동시성-문제-해결하기-V2-비관적-락Pessimistic-Lock](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V2-%EB%B9%84%EA%B4%80%EC%A0%81-%EB%9D%BDPessimistic-Lock)
- [https://velog.io/@znftm97/동시성-문제-해결하기-V3-분산-DB-환경에서-분산-락Distributed-Lock-활용](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V3-%EB%B6%84%EC%82%B0-DB-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-%EB%B6%84%EC%82%B0-%EB%9D%BDDistributed-Lock-%ED%99%9C%EC%9A%A9)
- https://incheol-jung.gitbook.io/docs/q-and-a/spring/redisson-trylock
