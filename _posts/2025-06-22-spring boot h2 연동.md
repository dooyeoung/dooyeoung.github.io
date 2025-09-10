---
layout: post
title: "Spring Boot - H2: 내 컴퓨터에 DB 설치 없이 개발하기"
subtitle: "H2 데이터베이스 연동 가이드"
categories: ["Spring Boot"]
tags: ["spring-boot", "jpa", "h2", "database"]
sidebar: ['article-menu']
---

H2는 별도의 설치가 필요 없는 '연습용 DB'라고 생각하면 쉽습니다. 보통 메모리 위에서 동작하기 때문에 속도가 매우 빠릅니다.
이번 글에서는 H2 데이터베이스를 Spring Boot 프로젝트에 연동하는 방법을 정리합니다.

## 1. 프로젝트에 H2와 JPA 설정
`build.gradle.kts` 파일의 `dependencies` 블록에 아래 두 줄을 추가합니다.

```
dependencies {
    runtimeOnly('com.h2database:h2')
    implementation('org.springframework.boot:spring-boot-starter-data-jpa')
}
```
- `com.h2database:h2`: H2 데이터베이스 그 자체입니다.
- `spring-boot-starter-data-jpa`: Spring Boot에서 JPA를 쉽게 사용하도록 도와주는 패키지이며, H2 `@Entity` 같은 어노테이션을 사용할 수 있습니다.

> 의존성을 추가했다면, Gradle 프로젝트를 꼭 새로고침 해주세요! (IntelliJ의 코끼리 아이콘 클릭)

## 2.application.yml에 H2 설정

이제 Spring Boot에게 우리가 H2를 어떻게 사용하고 싶은지 알려줄 차례입니다. `src/main/resources/application.yml` 파일에 아래 내용을 추가해주세요. (만약 파일이 없다면 새로 만들어주세요)

```yaml
spring:
  h2:
    console:
      enabled: true
      path: /h2-console

  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 

  jpa:
    hibernate:
      ddl-auto: create-drop
    open-in-view: false
```
jpa.hibernate.ddl-auto 옵션별 동작 방식은 아래와 같습니다.

|옵션|시작 시|종료 시| 데이터 보존 |추천 환경|
|-|-|-|--------|-|
|create      |기존 테이블 삭제 후 재생성  |아무것도 안 함 | X      |개발 초기, 테스트 |
|create-drop |기존 테이블 삭제 후 재생성  |테이블 삭제    | X      |개발 초기, 테스트 |
|update      |변경분만 반영     |아무것도 안 함 | O      |개발 초기  |
|validate    |엔티티-테이블 일치 여부 검사 |아무것도 안 함 | O      |개발 후반, 운영 |
|none        |아무것도 안 함              |아무것도 안 함 | O      |운영 환경 (필수) |

## 3. H2 콘솔 접속하기
Spring Boot 애플리케이션을 실행하고, 웹 브라우저를 열어 아래 주소로 접속합니다.
> `http://localhost:8080/h2-console`

로그인 화면에서 JDBC URL 입력란에 `application.yml`에 작성했던 주소인 `jdbc:h2:mem:testdb` 을 입력합니다.
JDBC URL을 수정하고, 사용자 이름에 `sa`를 입력한 뒤 Connect 버튼을 클릭하면 관리화면에 접속됩니다.

## 4. 동작 확인하기

아직은 데이터베이스에 아무 테이블도 없어서 허전하죠. 간단한 `Member` 테이블을 만들고, 초기 데이터까지 넣어서 정말 잘 작동하는지 확인해봅시다.

1.  **`Member` 엔티티 클래스 만들기**
    ```java
    @Entity
    public class Member {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
        private String name;
        // Getter, Setter, ...
    }
    ```

2.  **초기 데이터 넣기 (`data.sql`)**
    `src/main/resources/` 폴더 아래에 `data.sql` 이라는 이름의 파일을 만들고, 아래 내용을 추가해주세요. Spring Boot는 시작할 때 이 파일을 자동으로 읽어서 실행해줍니다.
    ```sql
    INSERT INTO member (name) VALUES ('홍길동');
    INSERT INTO member (name) VALUES ('김철수');
    ```

서버를 재시작하고, 다시 H2 콘솔에 접속하면. 왼쪽 화면에 `MEMBER` 테이블이 생성됩니다. SQL 입력창에 `SELECT * FROM MEMBER` 입력후 실행하면 데이터를 조회할 수 있습니다. 
