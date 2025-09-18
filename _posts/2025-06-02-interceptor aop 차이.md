---
layout: post
title: "Spring Boot - Interceptor와 AOP 언제 무엇을 써야 할까?"
subtitle: "AOP와 Interceptor의 비교 및 적용방법"
categories: ["Spring Boot"]
tags: ["spring-boot", "aop", "interceptor", "aspect"]
sidebar: ['article-menu']
---

개발을 하다 보면 여러 비즈니스 로직에서 반복적으로 나타나는 공통 기능들이 있습니다. 
예를 들어, 모든 API 요청에 대한 로깅, 특정 메서드의 실행 시간 측정, 트랜잭션 처리 등이 있습니다.
만약 이러한 공통 기능들을 각 비즈니스 로직 코드 안에 직접 작성한다면 많은 중복으로 유지보수가 어려워집니다.

```java
@Service
public class ProductService {

    public Product getProduct(Long id) {
        // 공통 기능: 로깅
        System.out.println("getProduct 메서드 시작");
        
        // 핵심 비즈니스 로직

        // 공통 기능: 로깅
        System.out.println("getProduct 메서드 종료");
        return product;
    }

    public void saveProduct(Product product) {
        // 공통 기능: 로깅
        System.out.println("saveProduct 메서드 시작");
      
        // 핵심 비즈니스 로직
      
        // 공통 기능: 로깅
        System.out.println("saveProduct 메서드 종료");
    }
}
```

위 코드처럼 비즈니스 로직과 공통 기능이 섞여 있으면 다음과 같은 문제가 발생합니다.

Spring 프레임워크는 위와 같은 횡단 관심사를 분리하고 관리할 수 있도록 `Interceptor`와 `AOP` 두 가지 도구를 제공합니다. 
이 둘은 비슷해 보이지만, 명확한 차이점과 각자의 역할이 있습니다. 

## Interceptor


```java
// LogInterceptor.java
@Component
public class LogInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("REQUEST [" + request.getMethod() + " " + request.getRequestURI() + "]");
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("RESPONSE [" + response.getStatus() + "]");
    }
}

// WebConfig.java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LogInterceptor logInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(logInterceptor)
                .addPathPatterns("/api/**") // /api/** 경로에만 인터셉터 적용
                .excludePathPatterns("/api/public/**"); // 특정 경로는 제외
    }
}
```

`Interceptor`는 이름 그대로 HTTP 요청을 '가로채는' 역할을 하며, Spring MVC 프레임워크의 일부입니다.
DispatcherServlet이 컨트롤러를 호출하기 전후에 개입하여 공통 작업을 처리하는 데 특화되어 있습니다.
HTTP 요청을 먼저 확인하고 들여보내거나 내보내는 역할을 합니다.
`HandlerInterceptor` 인터페이스를 구현하고, `WebMvcConfigurer`에 등록하여 사용합니다.



## 2. AOP
`AOP(Aspect-Oriented Programming)`는 '관점 지향 프로그래밍'을 의미합니다. 
횡단 관심사를 'Aspect'라는 별도의 모듈로 분리하여 애플리케이션 전반에 걸쳐 필요한 곳에 적용하는 방식입니다. 
HTTP 요청뿐만 아니라 서비스, 리포지토리 등 Spring 컨테이너가 관리하는 모든 Bean 객체 메서드 호출에 개입할 수 있습니다.

Spring AOP는 프록시 기반으로 동작하고 AOP가 적용된 Bean을 호출하면, Spring은 실제 Bean 대신 프록시 객체를 생성하여 주입합니다.
프록시 객체가 공통 기능을 먼저 실행한 후, 실제 Bean의 메서드를 호출합니다.
```java
@Aspect
@Component
public class TimerAop {
  @Pointcut(value = "within(com.example.filter.controller.UserApiController)")
  public void timerPointcut(){}

  @Before(value = "timerPointcut()")
  public void before(JoinPoint jointPoint){
    Arrays.stream(jointPoint.getArgs()).forEach(
      it -> {
        if (it instanceof UserRequest){System.out.println("before: "+ it);}
      }
    );
  }

  @After(value = "timerPointcut()")
  public void after(JoinPoint joinPoint){
    Arrays.stream(joinPoint.getArgs()).forEach(
      it -> {
        if (it instanceof UserRequest){System.out.println("after: " + it);}
      }
    );
  }
  
  @AfterReturning(value = "timerPointcut()", returning = "result") 
  public void afterReturning(JoinPoint joinPoint, Object result){
    System.out.println("after returning: "+ result);
    Arrays.stream(joinPoint.getArgs()).forEach(
      it -> {System.out.println("after returning: " + it);}
    );
  }

  @Around(value = "timerPointcut()")
  public void around(ProceedingJoinPoint jointPoint) throws Throwable{
    System.out.println("메소드 실행 전");
    Arrays.stream(jointPoint.getArgs()).forEach(
      it -> {
        if(it instanceof UserRequest){
          System.out.println("around: "+it);
          ((UserRequest) it).setAge(100);
        }
      }
    );
    jointPoint.proceed(); // 실제 메소드 실행
    System.out.println("메소드 실행 후");
  }
}
```

위 예시 코드의 동작순서는 다음과 같습니다. UserApiController 클래스의 메서드가 호출되면

- @Before: 메서드 실행 전에 UserRequest 인자를 출력
- @Around (전): "메소드 실행 전" 출력, UserRequest 인자를 출력하고 age를 100으로 설정
- 메서드 실행: joinPoint.proceed()로 실제 메서드 호출
- @Around (후): "메소드 실행 후" 출력
- @After: 메서드 실행 후 UserRequest 인자를 출력
- @AfterReturning: 메서드의 반환값과 모든 인자를 출력



## 3. Interceptor, AOP 비교 정리

| 구분 | Interceptor | AOP                            |
| :--- | :--- |:-------------------------------|
| **적용 대상** | Spring MVC의 HTTP 요청 | Spring 컨테이너가 관리하는 모든 Bean의 메서드 |
| **동작 원리** | DispatcherServlet의 콜백 메커니즘 | 프록시(Proxy) 패턴                  |
| **세밀함** | URL 패턴 기반 (상대적으로 덜 세밀함) | Pointcut 표현식을 통해 매우 세밀한 제어 가능  |
| **결합도** | Spring MVC 프레임워크에 의존 | Spring 프레임워크에 의존               |
| **주요 사용 사례**| 인증/인가, 요청 로깅, IP 필터링 등 | 비즈니스 로직의 트랜잭션, 성능 측정, 예외 처리 등  |

## 4. 마무리

- HTTP 요청/응답과 직접적인 관련이 있다면`Interceptor`를 사용하세요.
    - 사용자의 JWT 토큰을 검증하여 인증 상태 확인, 특정 URL 패턴에 대한 접근 권한 체크, 모든 API 응답에 공통 헤더 추가

- 서비스, 리포지토리 등 비즈니스 로직 전반에 적용이 필요하다면 `AOP`를 사용하세요.
    - 모든 서비스 계층 메서드의 실행 시간 측정, 특정 어노테이션이 붙은 메서드에 트랜잭션 적용, 메서드 호출 시 파라미터와 반환 값 로깅

