---
layout: post
title: "CloudFront 캐시 무효화"
categories: "aws"
tags: ["cloudfront"]
sidebar: ['article-menu']
---

S3에 웹페이지를 배포하고 Cloud Front를 연결하여 사용할 시 S3내용 변경 후 Cloud Front에 즉시 반영 하고자 한다면 캐시 무효화 작업이 필요합니다.
CloudFront 설정에서 무효화 경로를 입력하여 설정 가능합니다.

<img class="post_img" src="/assets/images/posts/cloudfront.png">


AWS cli로 작업 한다면 다음과 같습니다.

``` bash
aws s3 sync dist "s3://[S3_BUCKET_NAME]/" --delete
aws cloudfront create-invalidation --distribution-id "[CF_DIST_ID]" --path /\*
```