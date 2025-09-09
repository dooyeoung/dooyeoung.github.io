---
layout: post
title: "테스트 EKS 환경에 Kubectl로 접근하기 위한 설정 가이드"
categories: "Kubernetes"
tags: ["k8s", "kubectl"]
sidebar: ['article-menu']
---

개발이나 테스트 과정에서 원격 쿠버네티스 클러스터의 상태를 확인하거나 리소스를 직접 조작해야 할 때가 많습니다. 이 글에서는 로컬 머신에서 `kubectl`을 사용하여 AWS EKS(Elastic Kubernetes Service) 테스트 환경에 안전하게 접근하기 위한 준비 과정을 단계별로 안내합니다.

## **사전 준비사항**

### **필수 도구**
- **AWS CLI**: 최신 버전 (v2 권장)
- **kubectl**: 대상 EKS 클러스터와 호환되는 버전
- **권한 이해**: IAM 사용자, 역할, 정책에 대한 기본 개념

### **EKS 접근 권한 구조**
EKS 클러스터 접근은 다음 두 단계의 권한 확인을 거칩니다:
1. **AWS IAM 인증**: AWS 자격 증명을 통한 신원 확인
2. **Kubernetes RBAC 권한**: 클러스터 내부의 리소스 접근 권한

따라서 EKS 클러스터에 접근하려면 **Managers 그룹**과 같은 적절한 IAM 그룹에 포함되어야 하며, 해당 그룹이 클러스터 내 RBAC와 연결된 IAM 역할(예: `k8sAdmin`)을 가정할 수 있어야 합니다.

## **보안 고려사항**

### **최소 권한 원칙**
- 테스트 환경 접근을 위한 별도의 IAM 사용자 생성
- 필요한 최소한의 권한만 부여
- 정기적인 액세스 키 로테이션 (90일 주기 권장)

### **자격 증명 보안**
- 액세스 키를 코드나 공개 저장소에 노출하지 않기
- AWS profiles를 사용한 환경별 자격 증명 분리
- 가능한 경우 임시 자격 증명(STS) 사용 고려

## **1. IAM 사용자 및 액세스 키 생성**

### **IAM 사용자 생성**
AWS 콘솔에서 IAM 서비스로 이동하여 새로운 사용자를 생성합니다:

1. **IAM > 사용자 > 사용자 생성** 클릭
2. **사용자 이름**: `kubectl-test-user` (또는 적절한 이름)
3. **액세스 유형**: "프로그래밍 방식 액세스" 선택
4. **권한**: 기존 Managers 그룹에 추가 또는 필요한 정책 연결

### **액세스 키 생성**
사용자 생성 후 **보안 자격 증명** 탭에서 액세스 키를 생성합니다:

- **액세스 키 ID**: `AKIAIOSFODNN7EXAMPLE` (예시)
- **비밀 액세스 키**: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` (예시)

> ⚠️ **주의**: 비밀 액세스 키는 생성 시에만 확인할 수 있으므로 안전한 곳에 즉시 저장하세요.

## **2. AWS Profile 설정**

로컬 환경에서 여러 AWS 계정이나 환경을 다루는 경우, 명명된 프로필(named profile)을 사용하여 각 환경의 자격 증명을 분리하는 것이 좋습니다.

### **프로필 생성**
테스트 환경을 위한 별도의 AWS 프로필을 생성합니다:

```bash
$ aws configure --profile test
AWS Access Key ID [None]: ABCDEF  # 위에서 생성한 액세스 키 ID
AWS Secret Access Key [None]: GHIJKL  # 위에서 생성한 비밀 액세스 키
Default region name [None]: ap-northeast-2  # EKS 클러스터가 위치한 리전
Default output format [None]: json  # 출력 형식 (json 권장)
```

### **설정 파일 확인**
명령 실행 후 생성되는 설정 파일들을 확인해 보겠습니다:

**`~/.aws/config`** (AWS 설정 파일)
```ini
[profile test]
region = ap-northeast-2
output = json
```

**`~/.aws/credentials`** (자격 증명 파일)
```ini
[test]
aws_access_key_id = ABCDEF
aws_secret_access_key = GHIJKL
```

> 💡 **팁**: 프로필을 사용하는 이유는 개발/테스트/운영 환경의 자격 증명을 분리하여 실수로 잘못된 환경에 접근하는 것을 방지하기 위함입니다.

## **3. kubectl Context 추가**

### **EKS 클러스터 정보를 kubeconfig에 추가**
AWS CLI를 사용하여 EKS 클러스터 정보를 로컬의 `kubeconfig` 파일에 추가합니다:

```bash
aws eks update-kubeconfig \
    --region ap-northeast-2 \
    --name lemonbase-test \
    --role-arn arn:aws:iam::455628414130:role/k8sAdmin \
    --alias lemonbase-test \
    --profile test
```

### **명령어 옵션 설명**
- `--region`: EKS 클러스터가 위치한 AWS 리전
- `--name`: EKS 클러스터 이름
- `--role-arn`: 클러스터 관리 권한을 가진 IAM 역할 ARN
- `--alias`: kubectl에서 사용할 컨텍스트 별칭
- `--profile`: 사용할 AWS 프로필 이름

> ⚠️ **중요**: `--role-arn`에 지정된 역할은 EKS 클러스터의 `aws-auth` ConfigMap에서 Kubernetes 권한과 매핑되어 있어야 합니다.

## **4. 설정 확인 및 검증**

### **컨텍스트 설정 확인**
추가된 컨텍스트가 정상적으로 등록되었는지 확인합니다:

```bash
# 모든 컨텍스트 목록 확인
kubectl config get-contexts

# 현재 활성 컨텍스트 확인
kubectl config current-context

# kubeconfig 전체 설정 확인
kubectl config view
```

### **클러스터 연결 테스트**
실제 클러스터에 연결되는지 테스트합니다:

```bash
# 클러스터 정보 확인
kubectl cluster-info

# 모든 네임스페이스의 파드 목록 조회
kubectl get pods -A

# 노드 정보 확인
kubectl get nodes
```

## **문제 해결 (Troubleshooting)**

### **일반적인 오류 및 해결방법**

#### **1. 인증 실패 오류**
```
error: You must be logged in to the server (Unauthorized)
```
**해결방법**:
- AWS 자격 증명이 올바른지 확인
- IAM 사용자가 올바른 그룹에 속해있는지 확인
- `aws sts get-caller-identity --profile test`로 현재 인증 상태 확인

#### **2. 역할 가정 실패**
```
error: unable to assume role
```
**해결방법**:
- IAM 사용자에게 `sts:AssumeRole` 권한이 있는지 확인
- 역할의 신뢰 정책에서 해당 사용자나 그룹이 허용되는지 확인

#### **3. 클러스터를 찾을 수 없음**
```
error: cluster not found
```
**해결방법**:
- 클러스터 이름과 리전이 정확한지 확인
- AWS CLI 프로필이 올바른 계정을 가리키는지 확인

### **권한 확인 명령어**
```bash
# 현재 AWS 인증 정보 확인
aws sts get-caller-identity --profile test

# EKS 클러스터 목록 확인
aws eks list-clusters --region ap-northeast-2 --profile test

# 특정 클러스터 정보 확인
aws eks describe-cluster --name lemonbase-test --region ap-northeast-2 --profile test
```

## **고급 사용법**

### **다중 환경 관리**
여러 환경을 효율적으로 관리하기 위한 방법:

```bash
# 컨텍스트 전환
kubectl config use-context lemonbase-production
kubectl config use-context lemonbase-staging
kubectl config use-context lemonbase-test

# 현재 컨텍스트 확인
kubectl config current-context

# 특정 컨텍스트로 명령 실행 (컨텍스트 전환 없이)
kubectl --context=lemonbase-test get pods
```

### **kubeconfig 파일 구조**
`~/.kube/config` 파일의 구조를 이해하면 문제 해결에 도움이 됩니다:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://xxx.gr7.ap-northeast-2.eks.amazonaws.com
  name: arn:aws:eks:ap-northeast-2:455628414130:cluster/lemonbase-test
contexts:
- context:
    cluster: arn:aws:eks:ap-northeast-2:455628414130:cluster/lemonbase-test
    user: arn:aws:eks:ap-northeast-2:455628414130:cluster/lemonbase-test
  name: lemonbase-test
users:
- name: arn:aws:eks:ap-northeast-2:455628414130:cluster/lemonbase-test
  user:
    exec:
      command: aws
      args:
      - eks
      - get-token
      - --cluster-name
      - lemonbase-test
      - --region
      - ap-northeast-2
      - --role-arn
      - arn:aws:iam::455628414130:role/k8sAdmin
      - --profile
      - test
```

## **보안 베스트 프랙티스**

### **정기적인 보안 관리**
- **액세스 키 로테이션**: 90일마다 새로운 액세스 키로 교체
- **권한 최소화**: 불필요한 권한 제거
- **접근 로그 모니터링**: CloudTrail을 통한 API 호출 추적

### **대안적 인증 방법**
- **AWS SSO**: 조직에서 SSO를 사용하는 경우 고려
- **eksctl**: 더 간단한 EKS 클러스터 관리 도구
- **IAM Roles for Service Accounts (IRSA)**: 애플리케이션 수준에서의 권한 관리

## **요약**

이 가이드를 통해 다음과 같은 과정을 완료했습니다:

1. ✅ **IAM 사용자 생성** - 최소 권한으로 테스트 환경 접근 계정 준비
2. ✅ **AWS 프로필 설정** - 환경별 자격 증명 분리
3. ✅ **kubectl 컨텍스트 추가** - EKS 클러스터 연결 설정
4. ✅ **연결 검증** - 실제 클러스터 접근 테스트

### **다음 단계**
- Kubernetes 네임스페이스별 RBAC 권한 설정
- Helm 차트 배포 및 관리
- 모니터링 및 로깅 솔루션 연동

이제 안전하고 효율적으로 여러 EKS 환경을 관리할 수 있는 기반이 마련되었습니다. 문제가 발생하면 문제 해결 섹션을 참조하시기 바랍니다.
