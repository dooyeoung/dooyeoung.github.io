---
layout: post
title: "EKS Private 변경 후 bastion에서 API Endpoint 접근"
categories: "AWS"
tags: ["EKS"]
sidebar: ['article-menu']
---

퍼블릭 엑세스였기에 로컬에서 K9S를 사용하여 편하게 사용하였으나 보안 문제로 접근 설정을 수정하기로 하였습니다.

`EKS` 클러스터 엔드포인트 엑세스 설정을 `public` 에서 `private`로 수정할 경우 동일한 `VPC` 또는 연결된 `VPC` 환경에서만 `EKS API Endpoint` 접근이 가능합니다.

`EKS API Endpoint` 접근을 `Private` 으로 변경 후 피어링 된 `VPC`에 위치한 `bastion` 의 `K8S`을 사용하기로 했습니다.


## EKS 엑세스 설정

EKS 네트워크 엑세스 설정을 프라이빗으로 번경합니다.

<img class="post_img" src="/assets/images/posts/eks_private_01.png">

## Bastion Private IP 확인

`EKS`에 접근을 허용할 `Bastion`의 내부 `IP`를 확인합니다.

<img class="post_img" src="/assets/images/posts/eks_private_02.png">

## 보안그룹 설정

`EKS` 네트워크 설정에서 추가 보안그룹에 `Bastion`으로 부터 수신된 `HTTPS 443` 접근을 허용합니다.
<img class="post_img" src="/assets/images/posts/eks_private_03.png">

<img class="post_img" src="/assets/images/posts/eks_private_04.png">
