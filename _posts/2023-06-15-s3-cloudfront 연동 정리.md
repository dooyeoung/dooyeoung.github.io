---
layout: post
title: "S3와 CloudFront를 연동하여 정적 콘텐츠 제공하기"
categories: "AWS"
tags: ["AWS", "s3", "cloudfront", "route53"]
sidebar: ['article-menu']
---

AWS S3는 정적 파일을 서빙할 수 있는 서비스이지만 사용자에게 더 저렴하고 빠르게 콘텐츠를 제공하기 위해서는 CDN 서비스인 CloudFront와 함께 사용하는 것이 좋습니다. 
CloudFront를 사용하면 사용자와 가까운 엣지 로케이션에 콘텐츠를 캐싱하여 지연 시간을 줄이고, OAI를 통해 S3 버킷으로의 직접 접근을 제어하여 보안을 강화할 수 있습니다.

이 글에서는 S3와 CloudFront를 연동하는 전체 과정을 정리합니다.

## S3 버킷 설정
CloundFront를 사용시 S3 버킷은 비공개로 생성하여 불필요한 보안 위협을 최소화합니다.
![](/assets/images/posts/s3_cf_01.png)
![](/assets/images/posts/s3_cf_02.png)
![](/assets/images/posts/s3_cf_03.png)

보안을 위해 KMS 설정도 진행합니다.


## 인증서 생성 (AWS Certificate Manager)
![](/assets/images/posts/s3_cf_04.png)
![](/assets/images/posts/s3_cf_05.png)
![](/assets/images/posts/s3_cf_06.png)
create records in route 53 클릭 후 route53에서 생성된것 확인합니다.

### CloudFront 배포 생성
![](/assets/images/posts/s3_cf_07.png)
![](/assets/images/posts/s3_cf_08.png)
![](/assets/images/posts/s3_cf_09.png)
![](/assets/images/posts/s3_cf_10.png)
목적에 따라 캐시 설정 필요하며 현재 구성에서는 Response headers policy 설정 필요하여 추가하였습니다.


### **3. Route 53 설정**

![](/assets/images/posts/s3_cf_11.png)
![](/assets/images/posts/s3_cf_12.png)

CloudFront 배포가 완료되면, Route 53에서 위에서 설정한 대체 도메인 이름(CNAME)에 대한 레코드를 생성합니다.

-   레코드 유형은 `A`를 선택합니다.
-   "별칭(Alias)"을 활성화하고, 대상으로 위에서 생성한 CloudFront 배포를 선택합니다.


### **4. KMS 설정 (필요시)**
![](/assets/images/posts/s3_cf_13.png)
만약 S3 버킷의 객체들이 KMS로 암호화되어 있다면, CloudFront가 해당 객체를 읽을 수 있도록 KMS 키 정책을 수정해야 합니다.

위 과정을 모두 마치면, 이제 S3에 저장된 정적 파일을 CloudFront를 통해 빠르고 안전하게 서비스할 수 있습니다.
