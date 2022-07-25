---
layout: post
title: "AWS Load Balancer Controller 설치시 주의 사항"
subtitle: 정책 설정 놓치지 않기
categories: "serverless"
tags: ["eks", "alb"]
sidebar: ['article-menu']
---

eks에서 AWS Load Balancer Controller를 적용시 정책을 필수로 설정해야 합니다.
정책 설정없이 Controller를 구성할 경우 인그레스 연결 작업에서 권한에 대한 에러가 발생하게 됩니다.

## 정책 생성
아래 [공식 문서](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/install/iam_policy_v1_to_v2_additional.json)의 정책 내용을 확인한 뒤 aws console에서 정책을 생성합니다
<img src="/assets/images/posts/alb_policy_01.png" width="640px" style="margin-left:0">
<img src="/assets/images/posts/alb_policy_02.png" width="640px" style="margin-left:0">

aws cli로 작업할 경우 명령어는 아래와 같습니다
``` bash
aws iam create-policy --policy-name [POLICY_NAME] --policy-document [POLICY_DOCUMENT]
```

## 정책 연결
eks에서 사용중인 역할에 위에서 만든 정책을 추가합니다
<img src="/assets/images/posts/alb_policy_03.png" width="640px" style="margin-left:0">

aws cli로 작업할 경우 명령어는 아래와 같습니다
``` bash
aws iam attach-role-policy --role-name [ROLE_NAME] --policy-arn [POLICY_ARN]
```
