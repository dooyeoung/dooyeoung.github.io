---
layout: post
title: "MySQL에서 postgres 마이그레이션 체험기"
categories: "Database"
tags: ["postgres", "RDS"]
sidebar: ['article-menu']
---

최근에 MySQL을 주력으로 사용하던 환경에서 벗어나 postgres 데이터베이스를 마이그레이션해 볼 기회가 있었습니다. postgres 성능, 다양한 확장 기능(Extension) 등에 대한 호기심이 있었기 때문입니다. 이 글에서는 로컬 환경에서 `pgloader`를 사용하여 MySQL에서 postgres 마이그레이션을 진행했던 과정을 공유합니다.

### **왜 마이그레이션을 결심했나?**

-   MySQL과 postgres 성능, 관리 편의성, 확장성 등을 직접 비교해보고 싶었습니다.
-   postgres 제공하는 다양한 Extension 중 쓸만한 기능이 있는지 탐색하고 적용해보고 싶었습니다.

### **로컬 환경에서 마이그레이션 과정**

#### **1. `pgloader`를 이용한 데이터 이관**

`pgloader`는 다양한 데이터베이스 간의 마이그레이션을 도와주는 강력한 도구입니다. `pgloader.conf` 파일을 작성하여 데이터 원본(MySQL)과 대상(postgres) 정보를 입력하면 손쉽게 데이터를 이관할 수 있습니다.

**`pgloader.conf` 설정**

```
LOAD DATABASE
    FROM mysql://root:rootpw@127.0.0.1:3306/lemonbase 
    INTO postgresql://lemonbase:lemonbasepw@127.0.0.1:5432/lemonbase

    WITH include drop, create tables, create indexes, reset sequences, foreign keys,
        prefetch rows = 1000

    SET statement_timeout TO '0'
    
    CAST type datetime to timestamptz;
```

**`pgloader` 실행**

```bash
pgloader -v pgloader.conf
```

#### **2. Docker Compose 설정**

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
      POSTGRES_DB: lemonbase
      POSTGRES_USER: lemonbase
      POSTGRES_PASSWORD: lemonbasepw
      POSTGRES_INITDB_ARGS: '--encoding=UTF-8 --lc-collate=C --lc-ctype=C'
    ports:
      - "5432:5432"
    volumes:
      - "./.postgres:/var/lib/postgresql/data"
    restart: always
```

#### **3. 수동 마이그레이션 작업**

`pgloader`가 대부분의 작업을 자동으로 처리해주지만, 일부 SQL 구문은 수동으로 수정해야 했습니다.

-   **View 마이그레이션:** MySQL에서 사용하던 View 생성 쿼리를 postgres 문법에 맞게 수정하여 다시 실행했습니다.
-   **UUID 필드 마이그레이션:** 기존에 `VARCHAR`로 저장하던 UUID를 postgres의 네이티브 `UUID` 타입으로 변경했습니다.
-   **`GROUP_CONCAT` → `STRING_AGG` 변환:** MySQL의 `GROUP_CONCAT` 함수를 postgres의 `STRING_AGG` 함수로 변경했습니다.
-   **`db_collation` 변경:** `utf8mb4_bin`에서 `C`로 collation 설정을 변경했습니다.

### **정리**

-   **더 좋은가?**
    -   로컬 환경에서 실제 프로젝트를 실행해보았을 때, MySQL과 체감할 만한 성능 차이는 느끼지 못했습니다. (물론 테스트 서버나 프로덕션 환경에서는 다를 수 있습니다.)
-   **어려웠나?**
    -   생각보다 어렵지 않았습니다. `pgloader`라는 좋은 도구 덕분에 데이터 이관 자체는 매우 편리했습니다. 마이그레이션의 핵심은 `pgloader`를 잘 활용하고, 일부 호환되지 않는 SQL 구문을 수정하는 것이었습니다.
