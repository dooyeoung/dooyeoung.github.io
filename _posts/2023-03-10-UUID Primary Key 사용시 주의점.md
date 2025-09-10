---
layout: post
title: "UUID를 Primary Key로 사용시 주의할 점"
categories: "Database"
tags: ["database", "uuid", "primary-key", "performance", "rds"]
sidebar: ['article-menu']
---

개발을 하다 보면 `id`와 `entity_id`를 모두 식별자로 사용하게 됩니다. 이때 '둘 중 하나만 사용해도 되지 않을까?' 하는 질문을 받을 수 있습니다.
결론부터 말하자면, PK는 Auto-increment Integer로 사용하고, UUID는 외부 식별자(Surrogate Key)로 사용하는 방식이 여러 이점을 가집니다.

## Auto-increment PK의 장점: 성능 최적화

Auto-increment Integer를 PK로 사용하면 테이블 데이터 `INSERT`시 뛰어난 성능을 보입니다.

- Integer PK는 값이 1씩 증가하므로, 데이터가 디스크에 순차적으로 쌓이는 'Append' 방식으로 추가됩니다. 이는 데이터 저장 관점에서 랜덤하게 저장되는 UUID에 비해 B-Tree 인덱스 관리에 훨씬 효율적입니다. 페이지 분할(Page Split)이 덜 발생하여 데이터 저장 속도가 빠릅니다.
- PK는 대부분 클러스터형 인덱스로 생성되는데, 순차적인 데이터는 인덱스 구조를 효율적으로 유지하여 `SELECT` 나 `ORDER BY` 작업 시에도 이점을 가집니다.

> Ordered UUID (v1, v6 등)는 어떨까?
> UUID v1이나 v6처럼 시간 값을 기반으로 순차적으로 생성되는 'Ordered UUID'를 사용하면 Integer와 유사한 성능을 가지지만 UUID v1은 MAC 주소를 포함하여 보안에 취약할 수 있고, 모든 데이터베이스 시스템에서 완벽하게 순서를 보장하지 않기에 주의가 필요합니다.

## UUID를 외부 식별자(Surrogate Key)로 사용하는 이유

그렇다면 UUID는 언제 사용해야 할까요? UUID는 시스템 외부로 노출되거나, 여러 시스템 간에 데이터를 주고받을 때 사용하는 '외부 식별자'로서 장점을 가집니다.

- DDD 관점에서 Entity는 시스템 전체에서 고유한 식별자를 가져야 합니다. Auto-increment `id`는 하나의 테이블 안에서는 유일하지만, 시스템의 모든 Entity를 통틀어보면 중복될 수 있습니다. 반면 UUID는 중복없이 부여 가능하므로 여러 마이크로서비스나 분산 환경에서도 충돌 없이 사용가능 합니다.
- Auto-increment PK는 데이터가 데이터베이스에 `INSERT` 되고 `COMMIT` 되기 전까지는 어떤 값을 가질지 예측이 어렵습니다. 하지만 UUID는 애플리케이션 로직에서 미리 생성할 수 있어, 데이터 저장 전에 다른 객체와의 관계를 맺는 등 유연한 설계가 가능해집니다. 
     (물론 PK 생성 책임을 DB에 두는 것이 관심사 분리 측면에서 더 나은 구조일 수 있습니다.)
- Auto-increment `id`를 외부 API의 URL 등에 그대로 사용하면 (`/users/1`, `/users/2` ...), 외부에서 전체 데이터의 수나 사용자 규모를 쉽게 유추할 수 있습니다. UUID를 사용하면 이러한 추측을 원천적으로 차단이 가능합니다.

## 결론

각 방식의 장점을 종합하면 다음과 같은 결론을 내릴 수 있습니다.

- **pk id**: Integer Auto-increment 타입을 사용합니다. `INSERT`, `SELECT`, `ORDER BY` 등 데이터베이스 내부 작업의 성능을 극대화합니다.
- **entity_id (외부 식별자)**: UUID 타입을 사용합니다. 보안을 강화하고, DDD 관점의 고유성을 보장하며, 외부 시스템과의 연동을 용이하게 합니다.

이처럼 두 가지 방식의 장점을 모두 취하는 것이 안정적이고 확장성 있는 시스템을 설계하는 데 효과이라 생각합니다.
내부 시스템 간의 통신에는 Ordered UUID를 PK로 사용하는 것을 고려해볼 수 있지만, 일반적인 상황에서는 위와 같이 적절한 혼용 방식을 권장합니다.
