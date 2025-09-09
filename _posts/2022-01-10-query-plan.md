---
layout: post
title: "postgres 쿼리 플랜 확인하기"
categories: "postgres"
tags: ["SQL", "Visualizer" ]
sidebar: ['article-menu']
---

`postgres explain` 명령어로 쿼리 실행 과정을 확인 할 수 있습니다.
실행 과정을 파악하여 쿼리를 최적화 할 수 있습니다.
`explain` 결과로 내용을 파악할 수 있지만 결과를 그래프 형태로 보기 쉽게 변환하는 내용을 정리합니다.


## Explain 쿼리 작성

실행 과정을 확인할 쿼리를 아래와 같은 형식으로 작성합니다. 사용하기 쉽게 `*.sql` 파일에 작성합니다

``` sql
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON) SELECT * FROM activity
JOIN place ON activity.place_id = place.id
WHERE activity.created_at > '2022-01-01'
AND place_id = 7867
limit 100
```

## Explain 결과 저장

psql 명령어를 사용하여 `*.sql` 에 작성한 쿼리를 실행하고 플랜 결과 `*.json`으로 저장합니다.
```
psql [DB_URL] -f explain.sql >> plan.json
```

## 시각화
[tatiyants](http://tatiyants.com/pev/#/plans/new) 에 접속하여 플랜결과와 쿼리를 입력하여 시각화 가능합니다.

<img class="post_img" src="/assets/images/posts/query_plan_1.png">

시각화 예시는 아래와 같습니다. 
<img class="post_img" src="/assets/images/posts/query_plan_2.png">
