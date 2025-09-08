---
layout: post
title: "테스트 EKS 환경에 Kubectl로 접근하기 위한 설정 가이드"
categories: "Kubernetes"
tags: ["kubernetes", "kubectl", "aws", "eks", "iam"]
sidebar: ['article-menu']
---

개발이나 테스트 과정에서 원격 쿠버네티스 클러스터의 상태를 확인하거나 리소스를 직접 조작해야 할 때가 많습니다. 이 글에서는 로컬 머신에서 `kubectl`을 사용하여 AWS EKS(Elastic Kubernetes Service) 테스트 환경에 안전하게 접근하기 위한 준비 과정을 단계별로 안내합니다.

### **1. IAM 사용자 생성**

가장 먼저, EKS 클러스터에 접근할 수 있는 권한을 가진 IAM 사용자를 생성해야 합니다. 보안을 위해 최소한의 권한만 가진 사용자를 생성하고, 프로그래밍 방식 액세스를 위한 **액세스 키 ID**와 **비밀 액세스 키**를 발급받아 안전한 곳에 저장합니다.

### **2. AWS Profile 설정**

로컬 환경에서 여러 AWS 계정이나 환경을 다루는 경우, 명명된 프로필(named profile)을 사용하여 각 환경의 자격 증명을 분리하는 것이 좋습니다. `aws configure` 명령어에 `--profile` 옵션을 주어 테스트 환경을 위한 프로필을 생성합니다.

```bash
$ aws configure --profile test
AWS Access Key ID [None]: YOUR_TEST_ACCESS_KEY
AWS Secret Access Key [None]: YOUR_TEST_SECRET_KEY
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

위 명령을 실행하면 `~/.aws/` 디렉토리 아래에 `config`와 `credentials` 파일이 생성되거나 업데이트됩니다.

**`~/.aws/config`**
```ini
[profile test]
region = ap-northeast-2
```

**`~/.aws/credentials`**
```ini
[test]
aws_access_key_id = YOUR_TEST_ACCESS_KEY
aws_secret_access_key = YOUR_TEST_SECRET_KEY
```

### **3. `kubectl` Context 추가**

이제 AWS CLI를 사용하여 EKS 클러스터 정보를 로컬의 `kubeconfig` 파일에 추가할 차례입니다. `aws eks update-kubeconfig` 명령어를 사용하면 이 과정을 쉽게 처리할 수 있습니다.

```bash
aws eks update-kubeconfig \
    --region ap-northeast-2 \
    --name lemonbase-test \
    --role-arn arn:aws:iam::ACCOUNT_ID:role/k8sAdmin \
    --alias lemonbase-test \
    --profile test
```

-   `--role-arn`: EKS는 IAM 역할 기반으로 쿠버네티스 내부의 RBAC 권한을 매핑합니다. 클러스터 관리자에게 부여된 IAM 역할의 ARN을 전달해야 합니다.

### **4. 설정 확인**

모든 설정이 완료되면, `kubectl` 명령어로 컨텍스트가 잘 추가되었는지 확인합니다.

```bash
# 추가된 컨텍스트 목록 확인
kubectl config get-contexts

# 현재 컨텍스트가 방금 추가한 컨텍스트로 설정되었는지 확인
kubectl config current-context
```

마지막으로, 클러스터에 정상적으로 접근되는지 확인하기 위해 파드 목록을 조회하는 등 간단한 명령어를 실행해봅니다.

```bash
kubectl get pods -A
```

위와 같이 프로필과 컨텍스트를 설정해두면, 여러 쿠버네티스 클러스터를 다룰 때도 `kubectl config use-context <context-alias>` 명령어로 손쉽게 클러스터를 전환하며 작업할 수 있습니다.
