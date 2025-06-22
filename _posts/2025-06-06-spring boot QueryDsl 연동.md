---
layout: post
title: "Spring Boot에서 QueryDsl 연동"
subtitle: "Spring Boot에서 QueryDsl 연동"
categories: "Spring Boot"
tags: ["Spring Boot"]
sidebar: ['article-menu']
---

## QueryDsl?
- Querydsl의 가장 큰 장점은 컴파일 시점에 오류를 감지할 수 있는 타입-세이프한 쿼리 작성입니다.
- SQL을 문자열로 작성하는 경우, 오타나 잘못된 컬럼명 등은 런타임에 오류를 발생시킵니다. 하지만 Querydsl은 엔티티에 기반한 Q-Type이라는 메타모델 클래스를 자동으로 생성하여, 자바 코드처럼 쿼리를 작성할 수 있습니다.


### build.gradle 설정
```
dependencies {
    ...
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta' // Spring Boot 3.x 이상 (Jakarta EE)
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta" // Q-Type 생성을 위한 어노테이션 프로세서
    annotationProcessor "jakarta.annotation:jakarta.annotation-api" // Querydsl 5.x 이상에서 필요
    annotationProcessor "jakarta.persistence:jakarta.persistence-api" // Querydsl 5.x 이상에서 필요
}

...

val querydslDir = "${layout.projectDirectory}/build/generated/querydsl"
sourceSets {
    getByName("main").java.srcDirs(querydslDir)
}
tasks.withType<JavaCompile> {
    options.generatedSourceOutputDirectory = file(querydslDir)
}
tasks.named("clean") {
    doLast {
        file(querydslDir).deleteRecursively()
    }
}
```

### QueryDsl 설정 추가
```java
@Configuration
public class QuerydslConfig {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

### QueryDsl 사용
```java
@Repository
@RequiredArgsConstructor
public class JpaUserListPagingQueryRepository {

    private final JPAQueryFactory jpaQueryFactory;
    private static final QUserEntity user = QUserEntity.userEntity;
    private static final QUserRelationEntity relation = QUserRelationEntity.userRelationEntity;

    public List<GetUserListResponseDto> getFollowerList(Long userId, Long lastFollowerId){
        return jpaQueryFactory
            .select(
                Projections.fields(
                    GetUserListResponseDto.class
                )
            )
            .from(relation)
            .join(user).on(relation.followerUserId.eq(user.id))
            .where(
                relation.followerUserId.eq(userId),
                hasLastData(lastFollowerId)
            )
            .orderBy(user.id.desc())
            .limit(20)
            .fetch();
    }
}
```

### QEntity 인식을 위한 ide 설정
build.generated.querydsl 디렉토리에서 오른쪽 마우스 클릭, 메뉴 하단의 `Mark Directory as` 이동하여 `Generated Sources Root`를 클릭합니다
<img class="post_img" src="/assets/images/posts/query_dsl1.png">

IntelliJ IDEA 메뉴의 file > Invalidate Caches를 사용하여 캐시를 제거합니다
<img class="post_img" src="/assets/images/posts/query_dsl2.png">
<img class="post_img" src="/assets/images/posts/query_dsl3.png">

