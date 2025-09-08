---
layout: post
title: "MariaDB/InnoDB의 Lock 종류와 동작 방식 이해하기"
categories: "Database"
tags: ["mariadb", "mysql", "innodb", "lock", "database", "transaction"]
sidebar: ['article-menu']
---

데이터베이스에서 동시성(Concurrency) 제어는 데이터의 무결성을 지키는 매우 중요한 요소입니다. 여러 트랜잭션이 동시에 같은 데이터에 접근할 때 발생할 수 있는 문제를 막기 위해 데이터베이스는 '잠금(Lock)' 메커니즘을 사용합니다. 이 글에서는 MariaDB의 InnoDB 스토리지 엔진에서 사용되는 다양한 잠금의 종류와 그 동작 방식에 대해 알아봅니다.

### **1. 공유 잠금 (Shared Lock, S Lock)**

공유 잠금은 데이터를 **읽을 때** 사용하는 잠금입니다. 여러 트랜잭션이 하나의 데이터에 대해 동시에 공유 잠금을 가질 수 있습니다. 즉, 여러 사용자가 동시에 같은 데이터를 읽는 것을 허용합니다. 하지만 어떤 트랜잭션이 공유 잠금을 가지고 있는 데이터에 다른 트랜잭션이 배타 잠금(X Lock)을 거는 것은 불가능합니다.

```sql
SELECT ... LOCK IN SHARE MODE;
```

### **2. 배타 잠금 (Exclusive Lock, X Lock)**

배타 잠금은 데이터를 **변경(수정/삭제)할 때** 사용하는 잠금입니다. 배타 잠금이 걸린 데이터는 다른 어떤 트랜잭션도 공유 잠금이나 배타 잠금을 획득할 수 없습니다. 오직 잠금을 획득한 트랜잭션만이 해당 데이터에 접근할 수 있습니다.

```sql
SELECT ... FOR UPDATE;
```

### **3. 의도 잠금 (Intention Lock, IS/IX Lock)**

의도 잠금은 테이블 수준에 거는 잠금으로, 앞으로 특정 **행(row)에 잠금을 걸 것**이라는 '의도'를 표현하는 잠금입니다. 의도 잠금이 있기 때문에, 다른 트랜잭션이 테이블 전체에 대한 잠금(예: `LOCK TABLES`)을 거는 것을 방지할 수 있습니다.

-   **의도 공유 잠금 (Intention Shared Lock, IS):** 트랜잭션이 특정 행에 공유 잠금(S Lock)을 걸 것임을 의미합니다. (`SELECT ... LOCK IN SHARE MODE`)
-   **의도 배타 잠금 (Intention Exclusive Lock, IX):** 트랜잭션이 특정 행에 배타 잠금(X Lock)을 걸 것임을 의미합니다. (`SELECT ... FOR UPDATE`)

| | X | IX | S | IS |
|---|---|---|---|---|
| **X** | Conflict | Conflict | Conflict | Conflict |
| **IX** | Conflict | Compatible | Conflict | Compatible |
| **S** | Conflict | Conflict | Compatible | Compatible |
| **IS** | Conflict | Compatible | Compatible | Compatible |

*잠금 호환성 매트릭스. Compatible은 동시 가능, Conflict는 충돌을 의미합니다.*

### **4. 레코드 잠금 (Record Lock)**

레코드 잠금은 이름 그대로 **인덱스의 레코드 하나**에만 거는 잠금입니다. `PRIMARY KEY`나 `UNIQUE KEY`로 특정 행을 조회하여 잠금을 걸 때 사용됩니다.

```sql
-- id가 10인 레코드에만 X Lock을 겁니다.
SELECT id FROM t WHERE id = 10 FOR UPDATE;
```

### **5. 갭 잠금 (Gap Lock)**

갭 잠금은 인덱스 레코드 사이의 **'간격(gap)'**에 거는 잠금입니다. 이 잠금의 주된 목적은 다른 트랜잭션이 이 간격 사이에 새로운 데이터를 삽입하는 것을 막아 '유령 읽기(Phantom Read)' 문제를 방지하는 것입니다.

![](/assets/images/posts/2024-07-15-mariadb-lock-1.png)
*id가 4와 7인 레코드 사이의 간격(gap)에 갭 잠금이 걸릴 수 있습니다.*

### **6. 넥스트 키 잠금 (Next-Key Lock)**

넥스트 키 잠금은 **레코드 잠금과 갭 잠금을 합친 것**입니다. 특정 인덱스 레코드와 그 레코드 바로 앞의 간격을 함께 잠급니다. InnoDB의 `REPEATABLE READ` 격리 수준(Isolation Level)에서 기본적으로 사용되는 잠금 방식입니다.

이처럼 InnoDB는 다양한 수준의 잠금을 조합하여 데이터의 일관성과 동시성을 모두 보장합니다. 개발자는 이러한 잠금의 종류와 동작 방식을 이해함으로써 데드락(Deadlock)을 방지하고 애플리케이션의 성능을 최적화할 수 있습니다.
