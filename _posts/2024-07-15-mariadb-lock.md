---
layout: post
title: "MariaDB/InnoDB의 Lock 종류와 동작 방식 이해하기"
categories: "Database"
tags: ["mariadb", "database", "lock", "innodb"]
sidebar: ['article-menu']
---

데이터베이스에서 동시성(Concurrency) 제어는 데이터의 무결성을 지키는 매우 중요한 요소입니다. 여러 트랜잭션이 동시에 같은 데이터에 접근할 때 발생할 수 있는 문제를 막기 위해 데이터베이스는 '잠금(Lock)' 메커니즘을 사용합니다. 이 글에서는 MariaDB의 InnoDB 스토리지 엔진에서 사용되는 다양한 잠금의 종류와 그 동작 방식에 대해 알아봅니다.

## **1. 공유 잠금 (Shared Lock, S Lock)**

공유 잠금은 데이터를 **읽을 때** 사용하는 잠금입니다. 여러 트랜잭션이 하나의 데이터에 대해 동시에 공유 잠금을 가질 수 있습니다. 즉, 여러 사용자가 동시에 같은 데이터를 읽는 것을 허용합니다. 하지만 어떤 트랜잭션이 공유 잠금을 가지고 있는 데이터에 다른 트랜잭션이 배타 잠금(X Lock)을 거는 것은 불가능합니다.

```sql
SELECT ... LOCK IN SHARE MODE;
```

## **2. 배타 잠금 (Exclusive Lock, X Lock)**

배타 잠금은 데이터를 **변경(수정/삭제)할 때** 사용하는 잠금입니다. 배타 잠금이 걸린 데이터는 다른 어떤 트랜잭션도 공유 잠금이나 배타 잠금을 획득할 수 없습니다. 오직 잠금을 획득한 트랜잭션만이 해당 데이터에 접근할 수 있습니다.

```sql
SELECT ... FOR UPDATE;
```

## **3. 의도 잠금 (Intention Lock, IS/IX Lock)**

의도 잠금은 테이블 수준에 거는 잠금으로, 앞으로 특정 **행(row)에 잠금을 걸 것**이라는 '의도'를 표현하는 잠금입니다. 의도 잠금이 있기 때문에, 다른 트랜잭션이 테이블 전체에 대한 잠금(예: `LOCK TABLES`)을 거는 것을 방지할 수 있습니다.
만약 의도 잠금을 걸지 않고 행을 수정하고 있는 경우에 `ALTER TABLE`이나 `ALTER COLUMN` 등의 작업이 발생하는 경우 문제가 발생할 수 있습니다.


-   **의도 공유 잠금 (Intention Shared Lock, IS):** 트랜잭션이 특정 행에 공유 잠금(S Lock)을 걸 것임을 의미합니다. (`SELECT ... LOCK IN SHARE MODE`)
-   **의도 배타 잠금 (Intention Exclusive Lock, IX):** 트랜잭션이 특정 행에 배타 잠금(X Lock)을 걸 것임을 의미합니다. (`SELECT ... FOR UPDATE`)

여기서 Intention Lock은 아래의 프로토콜을 따르게 됩니다.

- 트랜잭션이 테이블의 row에서 공유 잠금을 획득하려면 먼저 테이블에서 IS 이상을 획득해야 합니다.
- 트랜잭션이 테이블의 행에 대한 베타 락을 획득하려면 먼저 테이블에서 IX을 획득해야 합니다.

테이블 레벨의 잠금 유형 호환성은 아래 메트릭스에 요약되어 있습니다.

| | X | IX | S | IS |
|---|---|---|---|---|
| **X** | Conflict | Conflict | Conflict | Conflict |
| **IX** | Conflict | Compatible | Conflict | Compatible |
| **S** | Conflict | Conflict | Compatible | Compatible |
| **IS** | Conflict | Compatible | Compatible | Compatible |

*잠금 호환성 매트릭스. Compatible은 동시 가능, Conflict는 충돌을 의미합니다.*

## **4. 레코드 잠금 (Record Lock)**

레코드 잠금은 이름 그대로 **인덱스의 레코드 하나**에만 거는 잠금입니다. `PRIMARY KEY`나 `UNIQUE KEY`로 특정 행을 조회하여 잠금을 걸 때 사용됩니다.

```sql
SELECT id FROM t WHERE id = 10 FOR UPDATE; -- id가 10인 레코드에만 X Lock을 겁니다.
```
이 때, 다른 트랜잭션에서 id = 10인 index record에 데이터를 변경하려고 한다면 해당 index는 이미 X Lock이 걸려있는 상태이기 때문에 transaction이 끝날 때까지 대기하게 됩니다.

```sql
UPDATE t SET id = 12 WHERE id = 10;  -- id=10 record에 X Lock이 걸려있어 대기 발생
```

## **5. 갭 잠금 (Gap Lock)**

갭 잠금은 인덱스 레코드 사이의 **'간격(gap)'**에 거는 잠금입니다. 
주로 `SELECT ... FOR UPDATE`나 `INSERT, UPDATE, DELETE`와 같은 쓰기 작업에서 트랜잭션이 다른 트랜잭션의 간섭 없이 일관된 데이터를 유지하도록 돕습니다.
이 잠금의 주된 목적은 다른 트랜잭션이 이 간격 사이에 새로운 데이터를 삽입하는 것을 막아 Phantom Read 문제를 방지하는 것입니다.
> Phantom Read: 트랜잭션이 동일한 조건으로 데이터를 읽을 때, 다른 트랜잭션이 새로운 데이터를 삽입하여 결과가 달라지는 현상입니다. 갭 잠금은 이런 상황을 방지합니다.



## 마무리
평소 ORM을 사용하면서 데이터베이스 잠금의 복잡한 동작 방식을 깊이 들여다볼 기회가 많지 않았습니다. 
기능이 잘 추상화되어 있다 보니, 그 내부에서 어떤 일이 일어나는지 굳이 신경 쓰지 않고도 개발이 가능했기 때문입니다.
예상치 못한 성능 저하나 데드락 문제를 마주했을 때 참고가 될 수 있는 내용이라 생각합니다
