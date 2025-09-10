---
layout: post
title: "Spring Boot - 생성자 주입을 사용해야 하는 이유"
subtitle: "필드 주입, 수정자 주입, 생성자 주입 완벽 비교 및 모범 사례 가이드"
categories: ["Spring Boot"]
tags: ["Spring Boot", "DI", "IoC", "Autowired", "Test"]
sidebar: ['article-menu']
---

Spring 프레임워크의 핵심 철학 중 하나는 IoC(Inversion of Control, 제어의 역전)이며, 이를 구현하는 대표적인 기술이 바로 DI(Dependency Injection, 의존성 주입)입니다. DI 덕분에 우리는 객체 간의 의존관계를 코드 내부가 아닌 외부(Spring 컨테이너)에서 설정하여, 유연하고 테스트하기 쉬운 코드를 작성할 수 있습니다.

Spring에서 의존성을 주입하는 방식은 크게 세 가지가 있지만, 왜 Spring 팀은 **생성자 주입**을 강력하게 권장할까요? 각 방식의 장단점을 통해 그 이유를 명확히 알아보겠습니다.

## 1. 필드 주입 (Field Injection)

필드에 `@Autowired` 어노테이션을 직접 붙이는 방식은 매우 간편하지만, 다음과 같은 심각한 단점 때문에 현재는 안티 패턴으로 여겨집니다.

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository; // 필드 주입
}
```

- **테스트의 어려움**: 단위 테스트는 Spring의 도움 없이 진행되는데, 필드 주입을 사용하면 의존성을 외부에서 넣어줄 방법이 없습니다. new UserService()로 객체를 만들어도 내부에 있는 userRepository는 계속 null 상태이기 때문입니다. 결국
   테스트를 위해 복잡한 방법이 필요헙니다.
- **순환 참조 위험**: 두 클래스가 서로를 필드 주입으로 참조하면, 애플리케이션 실행 시점에 오류를 잡지 못하고 프로세스가 종료될 수 있습니다.
- **단일 책임 원칙 위반 가능성**: 의존성 추가가 너무 쉬워 하나의 클래스가 비대해지기 쉽습니다.

## 2. 생성자 주입 (Constructor Injection)

이러한 필드 주입의 단점을 해결하는 것이 바로 **생성자 주입**입니다. 생성자를 통해 의존성을 주입하는 방식으로, 다음과 같은 명확한 장점이 있어 Spring 팀이 강력하게 권장합니다.

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    @Autowired // 생성자가 하나일 경우 생략 가능
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```
- **불변성 보장**: 의존성을 `final`로 선언할 수 있어, 객체가 생성된 이후 변경될 가능성이 원천 차단됩니다.
- **테스트 용이성**: 순수 Java 테스트에서 `new UserService(mockUserRepository)`와 같이 의존성을 명확히 주입하며 객체를 생성할 수 있습니다.
- **순환 참조 방지**: 순환 참조 발생 시, 애플리케이션 실행 시점에 `BeanCurrentlyInCreationException`을 발생시켜 문제를 즉시 알려줍니다.
- **의존성 누락 방지**: 객체 생성 시점에 필요한 모든 의존성이 주입되어야만 하므로, 컴파일 단계에서부터 의존성 누락을 막을 수 있습니다.


## 3. Lombok @RequiredArgsConstructor

생성자 주입의 유일한 단점은 의존성이 많아질수록 생성자 코드가 길어지는 것입니다.
Lombok의 `@RequiredArgsConstructor`을 사용하면 반복적인 생성자 코드를 개발자 대신 자동으로 만들어주는 역할을 합니다.
`@RequiredArgsConstructor`는 `final` 또는 `@NonNull`이 붙은 필드만을 포함하는 생성자를 자동으로 만들어줍니다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final OrderRepository orderRepository;

    // 아래 생성자가 자동으로 생성됩니다.
    /*
    public UserService(UserRepository userRepository, OrderRepository orderRepository) {
        this.userRepository = userRepository;
        this.orderRepository = orderRepository;
    }
    */
}
```

결국 `@RequiredArgsConstructor`는 `@Autowired`를 대체하는 기술이 아니라, 
Spring이 권장하는 생성자 주입 패턴(final 필드를 갖는 유일한 생성자)을 가장 깔끔하게 완성시켜주는 도구입니다.

## 4. 마무리

`@RequiredArgsConstructor`를 사용하면 `@Autowired`가 필요 없는 것으로 오해할 수 있습니다. 하지만 `@Autowired`는 다음과 같은 상황에서 유용합니다.

- 생성자가 여러 개일 때 어떤 생성자를 사용해 의존성을 주입할지 Spring에게 명시적으로 알려주기 위해 `@Autowired`가 필요합니다.
- `@Autowired(required = false)` 옵션을 통해 수정자(Setter) 주입 시 특정 Bean이 없어도 예외가 발생하지 않도록 처리할 수 있습니다.

의존성을 주입할 때, 다음 우선순위를 따르는 것을 권장합니다.
- `private final` 필드 + Lombok `@RequiredArgsConstructor`를 먼저 사용합니다. 불변성, 안정성, 테스트 용이성, 코드 간결성을 모두 만족하는 최상의 조합입니다.
- Lombok 사용하지 않는다면 `private final` 필드 + 생성자 직접 작성하는 방식을 사용합니다. 이 때 생성자가 유일하다면 `@Autowired`는 생략 가능합니다.
