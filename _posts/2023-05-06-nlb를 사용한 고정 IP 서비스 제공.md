---
layout: post
title: "AWS NLB를 이용한 고정 IP 서비스 제공하기"
categories: "AWS"
tags: ["aws", "nlb", "alb", "networking", "ip"]
sidebar: ['article-menu']
---

외부 고객사나 파트너사가 우리의 서비스에 접근할 때, 보안을 위해 방화벽에 특정 IP 주소를 허용해야 하는 경우가 종종 있습니다. 하지만 AWS의 Application Load Balancer(ALB)는 고정 IP를 제공하지 않아 이런 요구사항을 충족하기 어렵습니다.

이 글에서는 Network Load Balancer(NLB)를 ALB 앞에 배치하여 서비스에 고정 IP를 할당하는 방법을 소개합니다.

### **요구사항**

- 고객사의 방화벽에서 목적지 IP를 설정하기 위해 서비스 제공 IP를 고정해야 합니다.
- ALB는 고정 IP를 지원하지 않으므로, NLB를 추가로 설정하여 이 문제를 해결합니다.

### **아키텍처**

전체적인 구조는 다음과 같습니다.

`Client` → `Route 53` → `NLB (고정 IP)` → `ALB` → `ECS/EC2 등 서비스`

![](/assets/images/posts/2023-09-26-nlb-static-ip-1.png)

### **설정 단계**

#### **1. 탄력적 IP (Elastic IP) 생성**

먼저, 각 가용 영역(Availability Zone)에 할당할 탄력적 IP(EIP)를 생성합니다.

![](/assets/images/posts/2023-09-26-nlb-static-ip-2.png)

#### **2. 대상 그룹 (Target Group) 생성**

NLB가 트래픽을 전달할 대상으로 ALB를 지정하는 대상 그룹을 생성합니다.

- 대상 유형(Target type)을 **Application Load Balancer**로 선택합니다.
- 프로토콜은 `TCP`, 포트는 `80`과 `443`을 설정합니다.
- 트래픽을 전달할 ALB를 선택합니다.

![](/assets/images/posts/2023-09-26-nlb-static-ip-3.png)

#### **3. NLB (Network Load Balancer) 생성**

이제 NLB를 생성할 차례입니다.

- **리스너 설정:** `TCP:80`과 `TLS:443` 리스너를 추가하고, 위에서 생성한 대상 그룹으로 트래픽을 전달하도록 설정합니다.
- **네트워크 매핑:** NLB를 배치할 VPC와 서브넷을 선택하고, 각 서브넷에 1단계에서 생성한 탄력적 IP를 할당합니다.

![](/assets/images/posts/2023-09-26-nlb-static-ip-4.png)

#### **4. 확인 및 테스트**

설정이 완료되면 `nslookup` 이나 `dig` 명령어로 NLB의 DNS 이름이 우리가 할당한 고정 IP로 잘 변환되는지 확인합니다.

```bash
nslookup your-nlb-dns-name.elb.ap-northeast-2.amazonaws.com
```

로컬에서 테스트하기 위해 `/etc/hosts` 파일에 NLB의 고정 IP와 서비스 도메인을 추가하고 `curl` 명령으로 응답을 확인합니다.

```bash
# /etc/hosts 파일 수정
sudo vi /etc/hosts
# ...
43.201.204.175 your-service.com

# curl 테스트
curl -vvv https://your-service.com
```

마지막으로, Route 53에서 서비스 도메인의 DNS 레코드를 ALB의 주소 대신 NLB의 주소로 변경하면 모든 과정이 완료됩니다.

### **참고 자료**

- [Application Load Balancer-type Target Group for Network Load Balancer](https://aws.amazon.com/ko/blogs/networking-and-content-delivery/application-load-balancer-type-target-group-for-network-load-balancer/)
- [How do I associate a static IP address with my Application Load Balancer?](https://repost.aws/knowledge-center/alb-static-ip)
