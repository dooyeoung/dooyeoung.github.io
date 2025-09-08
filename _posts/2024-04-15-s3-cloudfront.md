---
layout: post
title: "S3와 CloudFront를 연동하여 정적 콘텐츠 제공하기"
categories: "AWS"
tags: ["aws", "s3", "cloudfront", "cdn", "route53", "kms"]
sidebar: ['article-menu']
---

AWS S3는 정적 파일을 저장하는 훌륭한 서비스이지만, 전 세계 사용자에게 빠르고 안전하게 콘텐츠를 제공하기 위해서는 CDN(Content Delivery Network) 서비스인 CloudFront와 함께 사용하는 것이 좋습니다. CloudFront를 사용하면 사용자와 가까운 엣지 로케이션에 콘텐츠를 캐싱하여 지연 시간을 줄이고, OAI(Origin Access Identity)를 통해 S3 버킷으로의 직접 접근을 제어하여 보안을 강화할 수 있습니다.

이 글에서는 S3와 CloudFront를 연동하는 전체 과정을 정리합니다.

### **1. S3 버킷 설정**

먼저 정적 파일을 저장할 S3 버킷을 생성합니다. 중요한 것은 버킷 정책(Bucket Policy)을 설정하여 CloudFront OAI를 통해서만 접근할 수 있도록 제한하는 것입니다. 이렇게 하면 사용자가 S3 URL로 직접 파일에 접근하는 것을 막을 수 있습니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-bucket-name/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::YOUR_AWS_ACCOUNT_ID:distribution/YOUR_CLOUDFRONT_DISTRIBUTION_ID"
                }
            }
        }
    ]
}
```

![](/assets/images/posts/2024-04-15-s3-cloudfront-1.png)

### **2. CloudFront 배포 생성**

다음으로 CloudFront 배포를 생성합니다.

-   **원본(Origin):** 위에서 생성한 S3 버킷을 선택합니다.
-   **원본 액세스(Origin Access):** "Origin access control settings"를 선택하고, "원본 액세스 제어 설정 만들기"를 통해 OAI를 생성합니다. CloudFront가 이 OAI를 사용하여 S3 버킷에 접근하게 됩니다.
-   **뷰어(Viewer) > 캐시 키 및 원본 요청:** 목적에 맞게 캐시 정책과 응답 헤더 정책을 설정합니다. 예를 들어, CORS 설정이 필요하다면 `Managed-CORS-S3Origin`과 같은 관리형 정책을 사용할 수 있습니다.
-   **설정(Settings) > 대체 도메인 이름(CNAME):** CloudFront에 연결할 자신만의 도메인(예: `cdn.example.com`)을 입력합니다.
-   **사용자 정의 SSL 인증서:** 위에서 설정한 도메인을 위한 SSL 인증서를 ACM(AWS Certificate Manager)에서 발급받아 연결합니다.

![](/assets/images/posts/2024-04-15-s3-cloudfront-2.png)

### **3. Route 53 설정**

CloudFront 배포가 완료되면, Route 53에서 위에서 설정한 대체 도메인 이름(CNAME)에 대한 레코드를 생성합니다.

-   레코드 유형은 `A`를 선택합니다.
-   "별칭(Alias)"을 활성화하고, 대상으로 위에서 생성한 CloudFront 배포를 선택합니다.

![](/assets/images/posts/2024-04-15-s3-cloudfront-3.png)

### **4. KMS 설정 (필요시)**

만약 S3 버킷의 객체들이 KMS로 암호화되어 있다면, CloudFront가 해당 객체를 읽을 수 있도록 KMS 키 정책을 수정해야 합니다. KMS 키 정책에 CloudFront 서비스 주체(`cloudfront.amazonaws.com`)가 `kms:Decrypt` 작업을 수행할 수 있도록 권한을 추가해야 합니다.

![](/assets/images/posts/2024-04-15-s3-cloudfront-4.png)

위 과정을 모두 마치면, 이제 S3에 저장된 정적 파일을 CloudFront를 통해 빠르고 안전하게 서비스할 수 있습니다.
