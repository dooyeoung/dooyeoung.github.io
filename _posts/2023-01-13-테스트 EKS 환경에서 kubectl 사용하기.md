---
layout: post
title: "테스트 EKS 환경에 Kubectl로 접근하기 위한 설정 가이드"
categories: "DevOps"
tags: ["kubernetes", "kubectl", "aws", "eks"]
sidebar: ['article-menu']
---

개발이나 테스트 과정에서 원격 쿠버네티스 클러스터의 상태를 확인하거나 리소스에 접근이 필요할 때가 있습니다 
로컬에서 `kubectl`을 사용하여 AWS EKS 테스트 환경에 접근하는 방법을 정리합니다.

## AWS Access Key 생성
먼저 aws 콘솔에서 보안 자격 증명 메뉴에서 access key를 생성합니다.
<img class="post_img" src="/assets/images/posts/test_kubectl.png">



## AWS Profile 설정
로컬 환경에서 `aws configure --profile test` 입력하여 test 환경에 사용할 계정 정보를 설정 합니다. 위에서 생성한 access ID, key 정보를 입력합니다.
```shell
$ aws configure --profile test
AWS Access Key ID [None]: ABCDEF
AWS Secret Access Key [None]: GHIJKL
Default region name [None]: ap-northeast-2
Default output format [None]:
```
설정이 반영된 ~/.aws 파일의 내용은 아래와 같습니다.
```text
# ~/.aws/config
[profile test]
region = ap-northeast-2

# ~/.aws/credentials
[profile test]
aws_access_key_id = ABCDEF
aws_secret_access_key = GHIJKL
```

## kubectl Context 추가
AWS 테스트 환경의 Managers 그룹에 추가되어 있어야 kubectl 사용이 가능합니다.
로컬환경에서 아래 명령어를 실행합니다.
```text
# test
aws eks update-kubeconfig \
    --region ap-northeast-2 \ # eks cluster 이 위치한 region
    --name your-cluster \ # eks cluster 이름
    --role-arn arn:aws:iam::123123:role/k8sAdmin \ # Managers 그룹과 연결된 role arn
    --alias your-cluster-alias \ # kubectl 에서 사용할 context 이름
    --profile test # aws profile 이름
```

설정된 context 정보를 확인합니다. 또는 ~/.kube/config 를 확인합니다
```text
kubectl config get-contexts
kubectl config view
```
동작 확인을 위해 pod 정보를 조회합니다
```text
kubectl get pods -A
```
