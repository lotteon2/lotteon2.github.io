---
title: 2주차. 트랜잭션과 관련된 문제들에 대해
description: 프로젝트를 진행하면서 자주 겪었던 트랜잭션과 관련된 문제들
author: changhyo
date: 2023-11-23 14:00 +0900
lastmod: 2023-11-23 14:00 +0900
categories: techoTalk
tags: transaction
---

# 트랜잭션이란

트랜잭션은 데이터베이스의 상태를 변화시키기 위해 수행하는 작업의 단위를 의미합니다.

일련의 작업 묶음이 안정적으로 처리되는걸 보장해주는 개념이 바로 트랜잭션이며 모든 작업이 성공해서 데이터베이스에 정상 반영하는 것을 커밋, 작업 중 하나라도 실패해서 작업 이전으로 되돌리는 것을 롤백이라고 합니다.

트랜잭션은 `ACID`라는 특성을 가지고 있으며, `4단계의 격리수준`을 가지고 있습니다. 우리가 자주 사용하는 MySQL InnoDB의 기본 격리수준은 Repeatable Read입니다.

# 내부 호출

```java
@Service
public class MyService {
		@Transactional
		public void txMethod() {
		}

		public void internalCall() {
				MyService.txMethod();
		}

}
```

1. MyService에는 @Transcational AOP를 적용한 txMethod라는 메서드가 존재합니다.
2. MyService 내부의 다른 메서드인 internalCall에서 txMethod를 호출하면 txMethod는 트랜잭션이 적용되지 않습니다.

@Transactional을 사용한 메서드가 존재하면 해당 객체는 원본 객체가 아닌 프록시 객체가 스프링 컨테이너에 등록됩니다. 

아주 간단히 살펴보면 다음과 같은 모양을 하고 있을 겁니다.

```java
public class MyServiceProxy {
		private final MyService myService;

		public void txMethod() {
				// rollback false;
				transaction.begin(); 
				myService.txMethod();
				transaction.end();
		}

}
```

- 프록시 객체는 원본 객체를 변수로 가지고 있습니다.
- 트랜잭션을 사용하는 메서드가 호출되면 프록시 객체는 트랜잭션을 시작합니다.
- 원본 객체의 메서드를 실행합니다.
- 프록시 객체는 트랜잭션을 닫습니다.

내부 호출은 프록시 객체를 호출하지 않습니다. internalCall메서드가 실행하는 txMethod();는 사실 this.txMethod()입니다. 여기서 this는 MyService를 의미합니다. 그렇기 때문에 프록시 객체가 아닌 원본 객체의 메서드만을 호출하고, 트랜잭션이 적용되지 않습니다.

# 테스트 코드에서의 트랜잭션

- 테스트 코드에서 @Transcational AOP를 사용하면 해당 테스트에서 했던 작업들을 테스트가 완료되는 시점에 모두 rollback하겠다는 의미입니다.
    
    ![Untitled](https://github.com/lotteon2/lotteon2.github.io/assets/25142537/70a31fc4-fd86-4ad0-8880-03dba6e64950)
    
    - 테스트 코드 마지막에 트랜잭션에 의해 롤백됐다는 내용을 확인할 수 있습니다.
- 주의사항 : 테스트 코드에서 @Transactional을 사용하게 되면 product code에서 @Transactional이 없을 때 이를 발견하지 못합니다.

# ****@Transactional을 이용한 테스트 코드 속 더티체킹****

```java
@SpringBootTest
@Transactional
class ProductServiceTest {
    @Autowired
    private ProductRepository productRepository;
    @Autowired
    private ProductService productService;
    @Autowired
    private EntityManager em;
    
    @Test
    public void changeStatus() {
				// 트랜잭션 open 

        // Product객체 생성
        Product product = Product.builder()
                .cnt(5)
                .build();

				// Product객체 저장
        productRepository.save(product);
        
				// Product객체의 상태 변경 메서드 실행
        productService.changeProductCnt(product.getProductId(),100); // 더티체킹

				// dirtyChecking으로 변경된 내용을 반영하기 위해 flush 강제 실행
				em.flush();
        em.clear();
        
				// 값을 변경한 객체 찾아오기
        Product changedProduct = productRepository.findById(product.getProductId()).get();
        
        // 값이 변경됐는지 검증
        Assertions.assertThat(changedProduct.getCnt()).isEqualTo(100);
    } // 트랜잭션 닫힘 -> flush() -> dirty checking에 의해 DB에 반영

}
```

- 더티체킹은 영속성 컨텍스트가 관리하는 엔티티의 변경된 부분을 감지해 트랜잭션이 끝나는 시점에 데이터베이스에 반영해 주는 기능입니다.
    - 일반적으로 ‘트랜잭션이 끝날 때’라고 하지만 정확히는 flush가 발생하는 시점입니다. flush는 em.flush()를 직접 실행하거나, 트랜잭션이 커밋되거나, JPQL쿼리를 실행할 때 발생합니다.

# 분산 트랜잭션 관리

(분산 트랜잭션과 관련된 이전 프로젝트 후기 및 자세한 내용은 [여기](https://velog.io/@qwerty1434/MSA%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0)를 참고해 주세요.)

트랜잭션은 기본적으로 DB단위에서 보장됩니다. 만약 여러 DB간의 정합성을 보장하길 원한다면 다른 방법을 찾아야 합니다.

## Two-Phase Commit

2단계 커밋(2PC)은 투표 단계와 커밋 단계로 나뉩니다. 투표 단계에서 중앙 조정자(coordinator)는 트랜잭션에 참여할 모든 워커(worker)에 연락해 상태 변경이 가능한지 여부를 확인 요청합니다. 모든 워커가 요청받은 상태 변경이 가능하다면 알고리즘은 커밋 단계로 넘어갑니다. 하나의 워커라도 요청받은 상태 변경이 불가능하다면 전체 연산은 그대로 종료됩니다.

2PC에서는 워커가 상태 변경이 가능하다는 걸 알려준 직후에 `즉시 변경 사항이 적용되지 않습니다.` 대신 워커는 `미래의 어느 시점에 그 변경을 수행할 수 있음을 보장`하고 있습니다. 이러한 보장이 가능하려면 워커는 해당 `레코드를 잠궈둬야` 합니다. 이는 성능적인 측면에서 좋지 못하며, 작업의 소요시간이 길어질수록 자원을 더 오래 잠궈둬야 합니다.
또한 2PC는 `개별 프로세스별로 커밋이 수행되는 시점이 다르다`는 문제가 존재하며 이로 인해 `격리성`이 깨질 수 있습니다. 격리성이란 여러 트랜잭션이 간섭 없이 동시에 작동할 수 있음을 의미하고, 이를 위해 어떤 트랜잭션이 진행되는 과정에서의 중간 상태 변경을 다른 트랜잭션이 확인할 수 없어야 합니다. 하지만 2PC는 개별 프로세스별로 커밋이 수행되는 시점이 달라 전체 트랜잭션이 완료되지 않은 시점에 그 중간상태를 다른 트랜잭션이 볼 수 있습니다.

## SAGA Pattern

SAGA는 트랜잭션으로 묶이는 여러 작업 중 특정 작업이 실패했을 때 이전에 발생한 작업들을 상쇄하는 `보상 트랜잭션`을 통해 분산 트랜잭션의 원자성을 보장하는 패턴입니다. SAGA는 요청이 왔을 때 곧바로 요청을 처리하기 때문에 2PC처럼 자원을 `잠궈둘(lock)` 필요가 없습니다.

SAGA에서의 원자성은 우리가 아는 일반적인 DB 트랜잭션의 ACID관점의 원자성이 아닙니다. DB에서의 롤백은 커밋 전에 발생하며 롤백이 일어나면 트랜잭션이 전혀 시작하지 않은 것처럼 되돌려집니다. 하지만 SAGA에서는 `이미 트랜잭션이 발생`했습니다. SAGA의 보상 트랜잭션을 통한 롤백은 `의미적 롤백`입니다. 한마디로 SAGA는 우리가 생각하는 완벽한 롤백을 해주지 못합니다!

SAGA패턴은 실제 롤백이 아니기 때문에 보상이 진행되는 그 순간에는 정합성이 보장되지 않을 수 있습니다. 하지만 결과적으로 시간이 흐른 뒤 모든 보상 트랜잭션이 수행되고 나면 트랜잭션이 롤백된 것과 다름없는 상태로 데이터의 정합성이 보장됩니다. 이러한 정합성을 `결과적 정합성`이라고 합니다.
