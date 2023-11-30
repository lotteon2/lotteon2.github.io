---
title: 1주차. 테스트 코드
description: 테스트 코드에 대한 짧은 설명
author: changhyo
date: 2023-11-16 23:03 +0900
lastmod: 2023-11-16 23:03 +0900
categories: techoTalk
tags: testCode
---

# 이 글은

- 테스트 코드가 낯선 팀원에게 테스트 코드에 대한 지식을 공유하기 위해 쓰여진 글입니다.
- [인프런 - Practical Testing: 실용적인 테스트 가이드](https://www.inflearn.com/course/practical-testing-%EC%8B%A4%EC%9A%A9%EC%A0%81%EC%9D%B8-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EA%B0%80%EC%9D%B4%EB%93%9C/dashboard) 내용을 바탕으로 쓰여진 글입니다.
- 정답을 제시하는 게 아닙니다. 다만 저는 이런 의견을 가지고 테스트 코드를 작성하고 있다는 걸 설명하는 글입니다.

# 테스트 코드는 왜 작성해야 할까?

테스트 코드를 작성하는 일은 생각보다 귀찮은 작업입니다. 프로젝트를 진행하면 정말 짧은 시간 내에 기능을 완성해야 하기 때문에 생각보다 테스트 코드를 작성할 시간이 부족합니다. 테스트 코드를 작성해야 하는 명확한 이유가 없다면 갈수록 테스트 코드 작성에 소홀해지게 되고, 나중에는 테스트 코드 작성을 멈춰버리게 될 수도 있습니다.

또한 테스트 코드가 존재하지만 그 내용이 엉망이거나 너무 복잡하면 결국 제 기능을 못하게 됩니다. 그렇기 때문에 단순히 테스트 코드를 작성하는 것도 중요하지만 Production code를 지원할 수 있도록 테스트 코드를 `잘` 작성하는 게 우리에게는 중요합니다. 

## 테스트는

### 코드 수정에 대한 안전장치다.

테스트 코드가 존재하지 않는다면 A라는 기능을 만들기 위해 기존에 있던 B라는 기능을 조금 수정했을 때, B가 이전과 동일하게 정상적으로 동작하는지 확인하기가 어렵습니다.

요구사항을 만족하는 테스트 코드를 잘 만들었다면 구현부를 자주 수정하거나, 구현부를 완전히 새롭게 바꾸는 등의 과감한 리팩토링이 가능해 집니다.

### 문서다.

테스트 코드는 ‘프로덕션 기능을 설명하는 문서’가 됩니다. 또한 테스트 코드는 ‘다양한 케이스를 통해 프로덕션 코드를 이해하는 시각과 관점을 보완’해 줘 테스트 코드를 통해 다양한 엣지케이스나 예외 케이스를 떠올릴 수 있게 해 줍니다.

또한 테스트 코드는 어느 한 사람이 과거에 경험했던 고민과 결과물을 팀 차원으로 승격시켜, 모두의 자산으로 공유할 수 있게 도와줍니다. 테스트 코드가 존재함으로 인해 내가 했던 작업과 고민을 다른 사람이 다시 하지 않아도 되게 해줍니다. 개발자는 항상 팀으로 일하기 때문에 내가 작성한 코드, 문서가 다른 사람에게 어떻게 인식될지는 상당히 중요한 부분이며 이러한 부분을 테스트 코드가 보장해줄 수 있습니다.

그래서 저는 테스트 코드를 작성할 때 ‘아무것도 모르는 사람이 봤을 때’를 항상 생각하려 합니다. 내가 짠 코드를 내가 아는건 당연합니다. 내 코드를 모르는 사람이 내 코드를 이해하는 데 도움을 줄 수 있어야 문서로서의 테스트 코드가 의미를 달성한다고 생각합니다.

# 테스트 코드 잘 작성하는 법

## 1. 테스트 케이스를 세분화하라

요구 사항을 그대로 만족하는 `해피 케이스`와 그렇지 않은 `예외 케이스`로 나눠서 테스트를 작성하면 좋습니다.

## 2. 테스트하기 어려운 영역을 분리하라(외부로 빼라)

테스트하기 어려운 영역이란 관측할 때마다 다른 값에 의존하는 코드(현재 날짜/시간, 랜덤 값, 전역 변수/함수, 사용자 입력 등), 또는 외부 세계에 영향을 주는 코드(표준 출력, 메시지 발송, DB에 기록하기 등)를 말합니다.

### 예시

**안좋은 예시**

**Production Code** 

```java
		private static final LocalTime SHOP_OPEN_TIME = LocalTime.of(10,0);
		private static final LocalTime SHOP_CLOSE_TIME = LocalTime.of(22,0);

		public Order createOrder(){
        LocalDateTime currentDateTime = LocalDateTime.now();
        LocalTime currentTime = currentDateTime.toLocalTime();
        if(currentTime.isBefore(SHOP_OPEN_TIME) || currentTime.isAfter(SHOP_CLOSE_TIME)){
            throw new IllegalArgumentException("주문 시간이 아닙니다. 관리자에게 문의하세요.");
        }

        return new Order(LocalDateTime.now(), beverages);
    }
```

**Test Code**

```java
		@Test
    void createOrder(){
        Order order = cafeKiosk.createOrder();

        assertThat(order.getId()).isNotNull();
    }
```

**좋은 예시**

**Production Code** 

```java
		private static final LocalTime SHOP_OPEN_TIME = LocalTime.of(10,0);
		private static final LocalTime SHOP_CLOSE_TIME = LocalTime.of(22,0);

		public Order createOrder(LocalDateTime currentDateTime){
        LocalTime currentTime = currentDateTime.toLocalTime();
        if(currentTime.isBefore(SHOP_OPEN_TIME) || currentTime.isAfter(SHOP_CLOSE_TIME)){
            throw new IllegalArgumentException("주문 시간이 아닙니다. 관리자에게 문의하세요.");
        }

        return new Order(LocalDateTime.now(), beverages);
    }
```

**Test Code**

```java
		@Test
    void createOrderCurrentTime(){
        Order order = cafeKiosk.createOrder(LocalDateTime.of(2023,1,17,10,0));

        assertThat(order.getId()).isNotNull();
    }

		@Test
    void createOrderOutsideOpenTime(){
        assertThatThrownBy(() -> cafeKiosk.createOrder(LocalDateTime.of(2023,1,17,9,59)))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("주문 시간이 아닙니다. 관리자에게 문의하세요.");

    }
```

## 3. 한 문단에 한 주제

테스트 코드는 문서의 역할을 합니다. 글쓰기 관점에서 하나의 문단에 하나의 주제만 있는게 좋듯이, 한 가지 테스트에서는 한 가지 목적의 검증만 수행하는 게 좋습니다. DisplayName이 한 문장으로 깔끔하게 적기 어렵다면 해당 테스트는 분리하는 게 좋을 수 있습니다.

테스트 코드에 논리구조가 들어가면 이는 하나의 목적이 아니게 될 가능성이 큽니다.

```java
@DisplayName("상품 타입이 재고 관련 타입인지를 체크한다.")
@Test
void containsStockTypeEx(){
		// given
		ProductType[] productTypes = ProductType.values();
		for(ProductType productType : productTypes){
				if(productType == ProductType.HANDMADE){
						// when
						boolean result = ProductType.containsStockType(productType);

						// then
						assertThat(result).isFalse();
				}

				if(productType == ProductType.BAKERY || productType == ProductType.BOTTLE){
						// when
						boolean result = ProductType.containsStockType(productType);

						// then
						assertThat(result).isTrue();
				}
		}	
}
```

## 4. 완벽하게 제어하기

테스트를 하기 위한 환경을 구성할 때 모든 조건을 완벽하게 제어할 수 있어야 합니다. 앞선 설명에서 ‘현재시간’이라는 변수를 메서드 내에서 생성하지 않고 메서드 외부에서 인자로 받도록 설계한 것 역시 완벽한 제어를 위한 설계라고 할 수 있습니다.

외부 시스템과 연동하는 경우 Mock처리를 해서 ‘우리가 제어할 수 없는 부분을 정상 동작할 것이라고 가정하거나 정상적이지 않을 때 어떤 응답을 줄지를 설정해’ 테스트 하는 작업을 진행할 수 있습니다.

## 5. 테스트 환경의 독립성을 보장하자

테스트를 진행할 때 불필요한 작업을 넣지 않아야 합니다. 

```java
@DisplayName("주문이 완료됐고 결제가 성공했으면 배송 상태를 보여준다")
@Test
void giveDeliveryStatusWhenOrderConfirmedAndPaymentSuccessed() {
    // given
    OrderDetailStatus orderDetailStatus = OrderDetailStatus.CONFIRMED;
    PaymentStatus paymentStatus = PaymentStatus.SUCCESS;
    DeliveryStatus deliveryStatus = DeliveryStatus.COMPLETE;
    RefundStatus refundStatus = null;
    OrderDetail orderDetail = createOrderDetail(orderDetailStatus, paymentStatus, deliveryStatus, refundStatus);
    orderDetail.changeRefundStatus(RefundStatus.REJECTED); // 현재 테스트에서는 불필요한 행동
    
    // when
    String result = orderDetail.getFinalStatusAsString();

    // then
    Assertions.assertThat(result).isEqualTo(deliveryStatus.getMessage());

}
```

## 6. 테스트 간 독립성을 보장하자

- static한 공유 변수는 가급적 사용하지 말아야 합니다.
- 테스트 간에는 순서가 없어야 합니다. JUnit5의 테스트는 기본적으로 순서를 보장해주지 않습니다. 만약 A가 이렇게 변하고, B가 이렇게 변했을 때 최종적으로 C가 되는 테스트를 작성하고 싶은데 A와B와C의 과정을 하나의 테스트에 담기에는 너무 많고, 그래서 테스트가 순서를 가지고 A→B→C로 진행됐으면 좋겠다면 `다이나믹 테스트`라는 걸 사용하는 걸 권장합니다.

## 7. 한 눈에 들어오는 Test Fixture 구성하기

Test Fixture란 테스트를 위해 원하는 상태로 고정시킨 일련의 객체로, given절에서 생성한 모든 객체들이 Test Fixture가 됩니다.

### 1. @BeforeEach로 Test Fixture를 구성하지 말자.

- given절을 설계하다보면 중복되게 만드는 변수가 매우 많습니다. 이들의 중복을 제거하기 위해 `@BeforeEach`를 사용하면 공유 변수를 가지게 되는 것과 동일한 효과를 가져 테스트가 원하는대로 진행되지 않을 수 있습니다.
    - `@BeforeEach`를 사용하면 테스트 간 결합도가 생기게 만듭니다.
    - 그래서 `@BeforeEach`는 되도록 사용하지 않는게 좋습니다.
- 만약 given절의 코드를 모두 `@BeforeXXX`에 옮겨두는 방법을 생각할 수도 있습니다. 이 경우 given 데이터를 확인하기 위해 계속 스크롤을 왔다갔다 해야하기 때문에 문서로써의 기능을 하기 어려울 수 있습니다.
- 그렇다면 `@BeforeEach`는 언제 사용해야 될까요?
    - 각 테스트의 입장에서 `해당 내용을 아예 몰라도 테스트 내용을 이해하는 데 아무런 문제가 없을 때`
    - 해당 내용을 수정해도 모든 테스트에 아무런 영향을 주지 않을 때
    - ex) Product엔티티를 테스트하는 상황이고 Product엔티티를 생성하기 위해서는 XXX엔티티가 반드시 필요합니다. 하지만 Product엔티티를 테스트 할 때는 XXX엔티티를 몰라도 된다면(값을 변경하거나 할 일이 없음, 만약 값을 변경하더라도 테스트에는 아무런 영향을 안줌, 단순히 생성을 위해 필요하기만 함) XXX엔티티를 @BeforeEach에 둬도 좋습니다.

### 2. data.sql로 Test Fixture를 구성하지 말자

- data.sql로 given절을 구성하는 방법 역시 권장하지 않습니다.이것 역시 @BeforeEach에서의 설명과 마찬가지로 테스트 전체 구조를 파악하기 어렵게 만듭니다. 또한 data.sql자체로 관리해야 할 대상이 됩니다.

### 3. 테스트 클래스마다의 builder를 구성하자

- 테스트 클래스마다 해당 테스트에 필요한 최소한의 변수만으로 객체를 만들 수 있도록 builder를 구성하는 걸 추천드립니다.
- 예시
    
    ```java
    private ProductRequest createProductRequest(Long productId, Long productPrice, Long productQuantity){
        return ProductRequest.builder()
                .productId(productId)
                .originalPrice(productPrice)
                .discountedPrice(productPrice)
                .productStock(productQuantity)
                .productName("제품이름")
                .productThumbnail("제품썸네일")
                .build();
    }
    ```
    
    - 현재 테스트에서는 product의 name이 중요하지 않다면 builder에 name을 변수로 받지 않으면 됩니다.
    - 이런 Builder를 모아둔 하나의 abstract class를 만들고 그때그때 가져다 쓰는 방법 역시 추천드리지 않습니다.
        - 테스트 할 때마다 매번 새로운 Builder가 필요하게 되며, 여러 사람이 각자 자신에게 필요한 Builder를 하나의 abstract class에 추가하기 시작하면 관리가 더욱 어려워 집니다.

## 8. BDD 스타일로 작성하기

BDD를 지키게 해 주는 툴 - Given / When / Then

- Given: 시나리오 진행에 필요한 모든 준비 과정 (객체, 값, 상태 등 준비)
- When: 시나리오 행동 진행
- Then: 시나리오 진행에 대한 결과 명시, 검증

BDD스타일의 테스트 코드는 ‘어떤 환경에서(Given) 어떤 행동을 진행했을 때(When), 어떤 상태 변화가 일어난다(Then)’는 하나의 시나리오를 검증하게 됩니다.

# DisplayName을 잘 작성하는 방법

영문 메서드 이름만으로 테스트코드가 무슨 기능을 하는지 표현하기 어렵습니다. 그래서 저는 DisplayName을 꼭 쓰는걸 권장드립니다.

### 명사 나열보다 문장으로 작성하라

- `A이면 B이다` 또는 `A면 B가 아니고 C다`와 같은 문장 형식이 좋습니다.
- 아무것도 모르는 사람이 봤을 때 `음료 1개 추가 테스트`보다는 `음료를 1개 추가할 수 있다`가 조금 더 이해하기 쉽습니다.
- `~테스트`라고 끝나는 건 좋지 못합니다.
    1. 문장 형식이 아니다
    2. 당연히 테스트하는 코드다 (불필요한 정보)

### 테스트 행위에 대한 결과까지 기술하라

- `음료를 1개 추가할 수 있다.` 보다는 `음료를 1개 추가하면 주문 목록에 담긴다.`가 더 이해하기 쉽습니다.

### 도메인 용어를 사용해 한층 더 추상화된 내용을 담아라

- `특정 시간 이전에 주문을 생성하면 실패한다.` 보다는 `'영업 시작 시간' 이전에는 주문을 생성할 수 없다.`라는 내용이 더 좋습니다.
    - ‘특정 시간’은 우리 도메인에서만 사용하는 용어가 아니라 일상 용어다.
    - 또한 특정 시간은 사람마다 생각하는 게 다를 수 있는 모호한 용어다. 하지만 영업 시작 시간은 적어도 우리 팀원은 전부 동일한 시간으로 인지하고 있는 명확한 시간이다.
- 도메인 용어를 사용할 때 ‘메서드 자체의 관점’보다 ‘도메인 정책 관점’으로 보면 좋습니다.
    - 메서드에 너무 집중하지 말고 비즈니스 정책을 녹여서 이름을 표현하라

### 테스트 현상을 중점으로 기술하지 말 것

- 테스트의 내용과 무관한 내용은 기술하지 않는 게 좋습니다.
- `특정 시간 이전에 주문을 생성하면 실패한다.`에서 `실패한다`의 의미는 테스트가 실패한다는 것일뿐 테스트의 내용과는 무관합니다. `주문을 생성할 수 없다.`라는 테스트의 현상을 적어주는 게 더 좋습니다.

# 그 외 내용들

### Repository와 Service 로직이 같은데 굳이 작성해야 하나요?

- Repository에서 Native Query나 Querydsl을 활용한 쿼리를 짰을 때 해당 로직이 잘 작동하는지 테스트 해볼 필요가 있습니다.
- 지금은 Repository와 Service 로직이 같지만 코드가 발전하면 어떻게 변할지 모르기 때문에 두 곳 다 테스트 코드를 작성하는 걸 추천드립니다.

### List에 대한 일반적인 검증방법

- Size가 원하는대로 잘 나왔는지 확인: `hasSize`
- 원하는 값이 잘 나왔는지 확인: `extracting + containsExactlyInAnyOrder`

### 객체 매번 만들기 귀찮아요

```java
@SpringBootTest
@Transactional
class OrderServiceTest {

		@DisplayName("제품 아이디,가격,수량들을 입력 받아 주문 및 주문 상세내역들을 생성한다")
    @Test
    void insertOrder() {
        // given
        List<ProductRequest> productRequests  = List.of(
                createProductRequest(1L, 100L, 1L),
                createProductRequest(2L, 500L, 2L),
                createProductRequest(3L, 1000L, 3L),
                createProductRequest(4L, 10000L, 4L)
        );
        Address address = Address.builder()
                .zipcode("zipcode")
                .roadAddress("roadAddress")
                .detailAddress("detailAddress")
                .build();

        // when
        orderService.insertOrder(productRequests, address, 1L);

        // then
        List<Order> orders = orderRepository.findAll();
        List<OrderDetail> orderDetails = orderDetailRepository.findAll();
        Assertions.assertThat(orders).hasSize(1);
        Assertions.assertThat(orderDetails).hasSize(productRequests.size())
                .extracting("order")
                .containsOnly(orders.get(0));
    }

		private ProductRequest createProductRequest(Long productId, Long productPrice, Long productQuantity){
        return ProductRequest.builder()
                .productId(productId)
                .originalPrice(productPrice)
                .discountedPrice(productPrice)
                .productStock(productQuantity)
                .productName("제품이름")
                .productThumbnail("제품썸네일")
                .build();
    }
}
```

- `createProduct()`
    - 빌더를 사용하다보니 코드가 너무 길어짐 → 빌더를 메서드로 따로 빼면 좋음
    - 바꿔야하는 내용만 변수로 받고 테스트에 불필요한 값들은 고정값으로 둘 수 있음

### 주의 사항 - Transactional

@SpringBootTest는 내부에 @Transactional이 포함되어 있지 않지만,  @DataJpaTest는 내부에 @Transactional이 포함되어 있습니다. 

테스트 코드에서 @Transactional은 롤백됩니다.

테스트 코드에서 @Transactional을 사용하지 않을 거라면 각각의 테스트가 독립적이기 위해 @AfterEach시점에 데이터를 삭제하는 작업을 진행해 줘야 합니다.

테스트 코드에서 @Transactional을 사용하면 실제 코드에 @Transactional이 없는 상황이 발생하는 문제를 발견할 수 없습니다.

### private 메서드의 테스트는 어떻게 하나요?

- 할 필요 없고, 해서도 안됩니다. public method를 검증하는 과정에서 private method도 자동으로 검증됩니다.
- private 메서드를 테스트하고 싶은 생각이 든다면 객체를 분리할 시점이 아닌지 고민해봐야 합니다. (하나의 객체가 너무 많은 일을 하는건지? 이 객체가 이 일을 하는게 맞는지 고민해볼 필요가 있습니다)

### 테스트에서만 필요한 메서드가 생겼는데 프로덕션 코드에서는 필요 없다면?

- 테스트에만 사용하는 메서드는 만들어도 된다. 하지만, 보수적으로 접근하는게 좋습니다. 보수적으로 접근하라는 의미는 무분별하게 생성하지 말고 어떤 객체가 마땅히 가져도 될 행위라고 생각된다면, 또는 미래에도 충분히 사용될 거 같은 메서드라면 만들어도 괜찮다는 의미입니다.

# 치팅시트

테스트 코드를 조금이라도 빠르게 작성하기 위해 정리한 내용입니다. 정답이 아니니 추가적인 내용을 꼭 찾아보시길 바랍니다.

[https://velog.io/@qwerty1434/TestCode-치팅시트](https://velog.io/@qwerty1434/TestCode-%EC%B9%98%ED%8C%85%EC%8B%9C%ED%8A%B8)