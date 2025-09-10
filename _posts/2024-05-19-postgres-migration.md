---
layout: post
title: "MySQL에서 postgres 마이그레이션 체험기"
categories: "Database"
tags: ["postgres", "database", "migration", "mysql"]
sidebar: ['article-menu']
---

로컬 환경에서 `pgloader`를 사용하여 MySQL에서 postgres 마이그레이션을 진행하는 과정을 정리합니다.



## Docker Compose 설정

개발 환경에서 postgres 실행하기 위해 `docker-compose.yml` 파일을 다음과 같이 수정했습니다.

```yaml
  postgres:
    image: "postgres:17beta3"
    command:
      [
        "postgres",
        "-c",
        "idle_in_transaction_session_timeout=10s",
        "-c",
        "lock_timeout=10s",
        "-c",
        "shared_preload_libraries=pg_stat_statements",
      ]
    environment:
      POSTGRES_DB: db
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_INITDB_ARGS: '--encoding=UTF-8 --lc-collate=C --lc-ctype=C'
    ports:
      - "5432:5432"
    volumes:
      - "./.postgres:/var/lib/postgresql/data"
    restart: always
```

## `pgloader`를 이용한 데이터 이관

`pgloader`는 다양한 데이터베이스 간의 마이그레이션을 도와주는 강력한 도구입니다. `pgloader.conf` 파일을 작성하여 데이터 원본(MySQL)과 대상(postgres) 정보를 입력하여 데이터를 이관할 수 있습니다.

```
LOAD DATABASE
    FROM mysql://root:rootpw@127.0.0.1:3306/db 
    INTO postgresql://admin:admin@127.0.0.1:5432/db

    WITH include drop, create tables, create indexes, reset sequences, foreign keys,
        prefetch rows = 1000

    SET statement_timeout TO '0'
    
    CAST type datetime to timestamptz;
```
`pgloader` 실행
```bash
pgloader -v pgloader.conf
```

## 마이그레이션 전처리 작업

`pgloader`가 대부분의 작업을 자동으로 처리해주지만, 일부 SQL 구문은 수동으로 수정해야 했습니다.

-   **View 마이그레이션:** MySQL에서 사용하던 View 생성 쿼리를 postgres 문법에 맞게 수정하여 다시 실행했습니다.
-   **UUID 필드 마이그레이션:** 기존에 `VARCHAR` 형태로 저장하던 UUID를 postgres의 네이티브 `UUID` 타입으로 변경했습니다.
    ```sql
    DELIMITER //
    CREATE FUNCTION uuid_generator(input_uuid VARCHAR(32))
    RETURNS CHAR(36)
    DETERMINISTIC
    BEGIN
    -- 32자 UUID를 받아서 하이픈을 추가해 36자 형식으로 반환
    RETURN CONCAT(
    SUBSTR(input_uuid, 1, 8), '-',
    SUBSTR(input_uuid, 9, 4), '-',
    SUBSTR(input_uuid, 13, 4), '-',
    SUBSTR(input_uuid, 17, 4), '-',
    SUBSTR(input_uuid, 21, 12)
    );
    END //
    DELIMITER ;

    ALTER TABLE db.table ALTER COLUMN entity_id TYPE UUID USING uuid_generator(entity_id);
    ```

-   `GROUP_CONCAT` → `STRING_AGG` 변환: MySQL의 `GROUP_CONCAT` 함수를 postgres의 `STRING_AGG` 함수로 변경했습니다.
    ```sql
    mysql
    GROUP_CONCAT(DISTINCT `lb_person`.`entity_id` SEPARATOR ',') AS `visible_person_entity_ids`
    
    postgres
    STRING_AGG(DISTINCT lb_person.entity_id::text, ',') AS visible_person_entity_ids
    ``` 
-   `db_collation` 변경: `utf8mb4_bin`에서 `C`로 collation 설정을 변경했습니다.
    ```sql
    # mysql
    db_collation='utf8mb4_bin'
    
    # postgres
    db_collation='C',
    ```


## 마무리
`pgloader` 덕분에 테스트 데이터 이관 자체는 매우 편리했습니다. 마이그레이션의 핵심은 `pgloader`를 잘 활용하고, 일부 호환되지 않는 SQL 구문을 수정하는 것이었습니다.
실제 데이터를 대상으로 작업할때는 `pgloader` 실행 환경 메모리 부족 문제를 피하기 위해 작업 단위를 나눠서 진행하는 방향도 고려해야 합니다.
