---
layout: post
title: "postgres Auto-Increment 중복 키 오류 해결 방법 (Sequence 갱신)"
categories: "AWS"
tags: ["cloudfront", "postgres"]
sidebar: ['article-menu']
---

얼마전 개발환경에서 dummy 데이터 추가 작업을 진행하였습니다. 일부 오브젝트 추가시 `auto increcement` 필드에 값을 직접 입력하여 구성하였습니다.

오브젝트 추가 후 `(psycopg2.IntegrityError) duplicate key value violates unique constraint 'id_key'` 오류가 발생하였습니다.

원인을 파악해보니 `postgres`에서 사용할 id 값이 중복인 문제였습니다. `postgres`은 `auto increcement` 필드에 대해 `seqence` 값을 따로 관리합니다. 

문제 해결을  `postgres` 이 관리하는 `seqence` 값을 테이블의 max id 값으로 갱신하였습니다.

아래 문제 확인을 위해 사용한 쿼리문을 정리합니다.


테이블의 `auto increcement` 필드의 `sequence` 정보를 확인합니다.
``` sql
select pg_get_serial_sequence('user', 'id');
```
`public.user_id_seq1`와 같은 값을 반환합니다.


위에서 조회한 `sequence` 값의 다음값을 확인해봅니다.
``` sql
select nextval('public.user_id_seq1');
```

테이블의 `id(auto increcement)` 최대값을 확인합니다.
``` sql
select max(id) from "user";
```

`public.user_id_seq1` 값을 갱신하여 `dupliated` 오류를 해결합니다.
``` sql
SELECT setval('public.user_id_seq1', (SELECT MAX(id) FROM "user")+1);
```
