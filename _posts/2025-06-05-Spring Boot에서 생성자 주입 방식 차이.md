---
layout: post
title: "Spring Boot에서 생성자 주입 방식 차이"
subtitle: "Spring Boot에서 @RequiredArgsConstructor와 @Autowired 비교"
categories: "Spring Boot"
tags: ["Spring Boot"]
sidebar: ['article-menu']
---

## @RequiredArgsConstructor 란?

@RequiredArgsConstructor는 Lombok 라이브러리에서 제공하는 어노테이션으로, final 또는 @NonNull 필드들을 초기화하는 생성자를 자동으로 생성해줍니다. 

### 동작 방식
```
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final @NonNull PasswordEncoder passwordEncoder;
    private String optionalField; // 생성자에 포함되지 않음
    ...
}
```

위 코드에서 @RequiredArgsConstructor는 다음과 같은 생성자를 자동 생성합니다:
```
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
```

### 특징
- final 필드를 사용하므로 객체 생성 후 필드 값이 변경되지 않습니다.
- @NonNull 필드에 null이 전달되면 NullPointerException가 발생합니다.

## @Autowired란?
Spring 프레임워크의 어노테이션으로 필드, 생성자, 또는 Setter 메서드에 적용하여 Spring이 자동으로 의존성을 주입합니다.

Spring 컨텍스트에 빈으로 등록되어 있어야 주입됩니다.

### 동작 방식
```
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private PasswordEncoder passwordEncoder; # 필드 주입(Field Injection)
    private String optionalField;
    ...
}
```
Spring 컨텍스트에서 Bean 등록
```
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

- Spring Boot 애플리케이션이 시작되면, Spring은 @Service, @Component, @Repository, @Controller 등이 붙은 클래스를 스캔하여 빈으로 등록합니다.
- @Bean으로 PasswordEncoder를 수동 등록합니다.
- @Autowired가 붙은 userRepository와 passwordEncoder 필드를 보고, Spring은 컨텍스트에서 해당 타입(UserRepository, PasswordEncoder)의 빈을 찾아 주입합니다.

### 특징
- 초보자도 쉽게 이해하고 사용할 수 있음. 의존성 주입 지점을 어노테이션으로 명확히 표시
- @Autowired 대신 @RequiredArgsConstructor을 사용하는 것이 불변성과 테스트 용이성 측면에서 더 나은 선택