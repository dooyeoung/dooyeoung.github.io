---
layout: post
title: "Interceptor AOP 차이"
subtitle: "Interceptor"
categories: "Spring Boot"
tags: ["Spring Boot"]
sidebar: ['article-menu']
---
Interceptor
0 스프링 MVC의 HTTP 요청 처리에 특화됨.
- 요청이 컨트롤러에 도달하기 전(preHandle), 컨트롤러 처리 후(postHandle), 뷰 렌더링 후(afterCompletion) 등 HTTP 요청 생명주기에 맞춰 동작.
- 웹 계층에서만 동작하며, 주로 URL 패턴 기반으로 특정 요청에 적용(예: /api/**).
- 스프링 컨텍스트 내의 컨트롤러 호출과 관련된 작업에 국한.

AOP
- 스프링 전반에 걸쳐 모든 빈의 메서드 호출에 적용 가능.
- 특정 클래스, 메서드, 패키지, 어노테이션 등 다양한 **포인트컷(Pointcut)**을 정의해 세밀한 적용 가능.
- 웹 계층뿐만 아니라 서비스, 리포지토리 계층 등 스프링 빈이 동작하는 모든 곳에서 사용 가능.



Interceptor
- HandlerInterceptor 인터페이스를 구현하고, WebMvcConfigurer를 통해 URL 패턴에 등록.
- preHandle, postHandle, afterCompletion 메서드를 통해 요청/응답 처리 시점에 개입.

AOP
- @Aspect 어노테이션을 사용해 Aspect를 정의하고, **포인트컷(Pointcut)**과 **어드바이스(Advice)**를 설정.
- @Before, @After, @Around 등 다양한 시점에서 메서드 실행 전/후/주변에 로직 적용.



Interceptor:
- 웹 요청 관련 공통 로직: 인증/인가, 로깅, 요청/응답 헤더 수정, CORS 처리.
- 예: API 요청마다 JWT 토큰 검증, 요청 처리 시간 로깅, 응답에 공통 메타데이터 추가.
- /api/admin/** 경로에 대해 관리자 권한 체크.
- 모든 API 요청에 X-Request-Id 헤더 추가.

AOP:
- 비즈니스 로직 관련 공통 로직: 서비스/리포지토리 계층의 메서드에 대한 로깅, 트랜잭션 관리, 예외 처리, 성능 모니터링.
- 예: 모든 서비스 메서드 실행 시간 측정, 특정 어노테이션이 붙은 메서드에 트랜잭션 적용.
- @Transactional 대신 AOP로 커스텀 트랜잭션 로직 적용.
- UserService의 모든 메서드 호출 시 입력/출력 데이터 로깅.



Interceptor: 스프링 MVC의 웹 요청 처리에 특화, HTTP 요청/응답과 관련된 공통 로직(인증, 로깅, 헤더 수정)에 적합.
AOP: 스프링 애플리케이션 전반의 비즈니스 로직 관련 공통 관심사(트랜잭션, 로깅, 예외 처리)에 적합, 더 유연하고 세밀한 적용 가능.
실무 선택 기준: 웹 요청 처리는 Interceptor, 서비스/리포지토리 계층의 공통 로직은 AOP. 두 기능을 조합해 사용하는 경우도 많음.