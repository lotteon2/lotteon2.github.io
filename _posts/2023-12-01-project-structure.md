---
title: 3주차. 내 프로젝트 구조
description: 설계와 아키텍처와 관련된 고민
author: changhyo
date: 2023-11-30 18:03 +0900
lastmod: 2023-11-30 18:03 +0900
categories: techoTalk
tags: clean architecture
---

평소 프로젝트 구조를 어떻게 설계하는게 좋은지? 3-layered-architecture를 주로 사용하고 있는데 어떤 계층에 어떤 코드가 들어가는게 가장 좋은지에 대한 고민이 있었습니다. 

고민 중 [좋은 글](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63)을 발견했고, 일부 내용을 이번 프로젝트에 직접 적용해보고 있습니다. 

이번 주 발표 주제는 해당 구조를 적용하면서 느낀 점들에 대한 공유해 보도록 하겠습니다.

## 예전 코드로 살펴보기

비즈니스 로직이란 무엇일까요? 이름에서 알 수 있듯이 우리의 비즈니스를 구성하는 논리적인 흐름입니다. 그리고 이 비즈니스 로직은 3-layered-architecture의 프로젝트라면 많은 경우 Service Layer에 위치하게 됩니다.

아래는 제가 약 1년전 프로젝트에서 작성한 서비스 레이어 메서드의 일부입니다.

![Untitled](https://github.com/lotteon2/lotteon2.github.io/assets/25142537/22a93499-8a69-4a8a-ae2e-e8aa3a6e23f5)

- hobbyHistory라는 객체를 저장하기 위해 많은 작업들이 서비스 레이어에서 진행되고 있습니다.
- 코드를 봤을 때 hobbyHistory라는 객체를 저장하는 로직이라는 것을 파악하는 것조차 쉽지 않습니다.

그래서 다음과 같은 설계 구조를 적용했습니다.

![Untitled](https://github.com/lotteon2/lotteon2.github.io/assets/25142537/7fb3c6d7-2b6a-4fec-b22e-db0be066022a)

handler는 Repository와 직접적으로 연관되어 있으면 상세한 구현 작업을 담당합니다.

> 저 같은 경우에는 객체의 저장을 담당하는 Creator, 객체를 불러오는 Reader, 객체의 관리를 담당하는 Manager를 주로 만들어 활용하고 있습니다.
> 

그래서 예전 코드를 현재 사용 중인 프로젝트 구조로 수정한다면 아마 다음과 같은 코드를 설계할 거 같습니다.

```java

public void createHobbyHistory() {
		Member member = memberReader.read();
		Hobby hobby = hobbyReader.read();
		
		HobbyHistory hobbyHistory = hobbyHistoryCreator.create();
		alarmManager.push();
		hobbyHistoryCreator.create();
}
```

이때 이전 코드에 있던 ‘조건 미달’이라는 검증을 진행하지 않은 것이 아닙니다. 다만 검증을 다른 계층에서 진행함으로써 서비스 레이어의 비즈니스 로직을 한눈에 파악할 수 있도록 변경했습니다. 

검증 로직, if-else코드, 중첩 for문 등 부피가 큰 복잡한 코드도 필요하다면 구현하는게 당연합니다. 다만 해당 코드의 책임을 어디에 둘 것인지, 해당 코드를 어디에 구현해서 얼마나 숨기고 보여줄 것인지를 결정하는 게 이번 글의 주된 내용인 셈입니다.

## 사용해보니

다음으로 프로젝트에 적용하면서 느낀 점들에 대해 간단히 설명드리겠습니다.

### 장점1. 서비스 로직이 이해하기 쉽고 간결해진다

이 부분은 위의 createHobby예시로 충분히 설명되었을 거라 생각해 추가적인 설명 없이 넘어가도록 하겠습니다.

### 장점2. repository를 활용할 때 필요한 공통 로직을 매번 작성하지 않아도 된다

Spring Data JPA를 사용하면 Optional객체를 반환하기 때문에 값을 확인하는 로직이 추가적으로 필요합니다. 만약 reader를 사용하지 않으면 다음과 같은 코드가 Service Layer에 반복될 것입니다.

```java
@Service
@Transactional
@RequiredArgsConstructor
public class myService() {
		private final StoreRepository storeRepository;
		
		public void method1(Long storeId) {
				Store store = storeRepository.findById(storeId)
																.orElseThrow(StoreAddressNotFoundException::new);
				// store를 활용한 로직1
		}

		public void method2(Long storeId) {
				Store store = storeRepository.findById(storeId)
																.orElseThrow(StoreAddressNotFoundException::new);
				// store를 활용한 로직2
		}

}
```

하지만 Reader객체를 만들어두면 다음과 같은 코드가 가능합니다.

```java
// StoreReader
@Component
@RequiredArgsConstructor
public class StoreReader() {
		public Store read(Long storeId) {
        return storeRepository.findById(storeId)
												.orElseThrow(StoreAddressNotFoundException::new);
    }
}

// Service
@Service
@Transactional
@RequiredArgsConstructor
public class myService() {
		private final StoreReader storeReader;		
		
		public void method1(Long storeId) {
				Store store = StoreReader.read(storeId);
				// store를 활용한 로직1
		}

		public void method1(Long storeId) {
				Store store = StoreReader.read(storeId);
				// store를 활용한 로직2
		}

}
```

- orElseThrow와 같은 코드를 매번 작성하지 않아도 됩니다.

### 장점3. 여러 Repository를 사용할 때도 주입 관련 코드가 깔끔하다

서비스 레이어에서 여러 도메인의 Repository를 활용해야 한다면 코드가 지저분해지기 쉽습니다. 특정 메서드 한 곳에서만 사용되는 Repository일지라도 Service는 모두 다 주입받아야 되며, 그 결과 코드가 금방 불어나 지저분해지게 됩니다.

```java
@Service
@Transactional
@RequiredArgsConstructor
public class myService() {
		// 서비스의 여러 메서드에서 사용되는 Repo
		private final StoreRepository storeRepository;
		// 단 하나의 메서드에서만 사용되는 Repo, 하지만 주입받기 위해 반드시 선언해야 한다 
		private final XXXRepository xxxRepository; 
}
```

handler계층에서 Repository를 주입받고, service는 handler계층을 주입받으면  Service계층이 한층 더 깔끔해 집니다.

```java
// StoreReader
@Component
@RequiredArgsConstructor
public class StoreReader() {
		// 여러 Repo를 주입받는다
		private final StoreRepository storeRepository;
		private final XXXRepository xxxRepository; 

		public Store read(Long storeId) {
        return storeRepository.findById(storeId)
												.orElseThrow(StoreAddressNotFoundException::new);
    }
}

// service
@Service
@Transactional
@RequiredArgsConstructor
public class myService() {
		// 여러 Repo를 주입받을 필요 없다
		private final StoreReader storeReader;
}
```

### 단점1. 단순한 로직의 경우 불필요한 작업이 될 수 있다

사용하다 보니 아쉬운 점도 있었습니다. 그것은 바로 Service계층이 단순히 Handler계층의 호출만 하는 경우입니다. 

물론 비즈니스 로직이 발전하면 둘은 다른 모습을 가지며 다른 의미를 지니게 될 것입니다. 하지만 로직이 변경될 가능성이 현저히 낮은 코드라면 불필요한 작업이라고 느껴지기도 합니다.

![Untitled](https://github.com/lotteon2/lotteon2.github.io/assets/25142537/b782b244-3b58-4e93-b91f-cb4de71e8121)

- Service Layer의 getStoreInfo는 단순히 storeReader를 호출하기만 합니다.

이러한 코드는 테스트 코드 작성 역시 애매했습니다. ‘Reader에 대한 테스트 코드를 작성한 시점에서 getStoreInfo의 테스트 코드가 굳이 필요한가’라는 생각이 들었습니다. (이 역시 비즈니스 로직이 발전할 가능성이 있다면 당연히 테스트 코드를 둘 다 작성해야 한다고 생각합니다)

끝으로 참고한 블로그 포스팅의 한 줄 요약과도 같은 내용을 부분을 가져와 봤습니다.

![Untitled](https://github.com/lotteon2/lotteon2.github.io/assets/25142537/87621af4-4cec-4134-a529-9def367752b6)

저 역시 이 의견에 동의하며, 그렇기 때문에 이번 프로젝트에 해당 프로젝트 구조를 적용해보게 되었습니다. 

[블로그](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63) 작성자 분께서 글을 정말 깔끔하고 이해하기 쉽게 잘 작성해 두셨습니다. 한번 들어가서 읽어보시는 걸 적극 추천드립니다. 

더불어 이 내용을 이해하셨다면 함수형 프로그래밍과 선언형 프로그래밍에 대해 알아보시는 것도 추천드립니다.

## References

- [https://geminikims.medium.com/지속-성장-가능한-소프트웨어를-만들어가는-방법-97844c5dab63](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63)
