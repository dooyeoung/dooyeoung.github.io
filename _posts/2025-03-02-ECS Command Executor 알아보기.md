---
layout: post
title: "ECS Execute Command 알아보기"
subtitle: ECS Command 접속하여 상태 알아보기
categories: "DevOps"
tags: ["ECS"]
sidebar: ['article-menu']
---

## ECS execute-command는 무엇인가? 
EAmazon ECS의 execute-command는 ECS 컨테이너의 쉘에 접속해 명령어를 실행할 수 있는 기능으로, EKS에서 pod에 접속해 내용을 확인하는 것과 유사합니다. 이 기능을 사용하면 컨테이너 내부에서 디버깅, 로그 확인, 설정 변경 등을 자유롭게 할 수 있습니다.

### 언제 왜 사용하는가?
디버깅 및 문제 해결
- 애플리케이션이 예기치 않게 실패하거나 오류 로그만으로 원인을 파악하기 어려울 때.
- 컨테이너 내부로 들어가 실시간으로 프로세스 상태, 환경 변수, 파일 시스템 등을 확인하며 문제를 진단할 수 있습니다. ECS 콘솔이나 CloudWatch로는 제공되지 않는 세부적인 리소스 정보를 컨테이너 내부에서 직접 확인할 수 있습니다. 셸에 접속해 실행 중인 프로세스(top)나 메모리, 디스크 사용량을 빠르게 확인할 수 있습니다.

임시 설정 변경 또는 테스트
- 배포 전 설정 파일을 수정하거나, 특정 명령어를 실행하여 동작을 확인하고 싶을 때.
- 컨테이너를 재배포하지 않고도 내부에서 환경 변수나 설정 파일을 수정한 뒤 테스트용 스크립트를 실행할 수 있습니다.


## 작업 순서

### 1.MFA사용 중이라면 STS를 통해 자격 증명 획득
AWS 환경에서 MFA를 사용하는 경우, 특히 영구 자격 증명 대신 임시 자격 증명이 필요하며 아래는 간단한 파이썬 스크립트입니다.

``` python
import argparse
import boto3

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--profile', '-p', help='aws configure 에 등록된 profile 을 입력해주세요', default='default')
    parser.add_argument(
        '--serial-number', '-s', help='MFA 시리얼 넘버를 입력해주세요 (arn:aws:iam::{account}:mfa/{username})'
    )
    parser.add_argument('--token', '-t', help='6자리 MFA 코드를 입력해주세요')
    args = parser.parse_args()

    session = boto3.Session(profile_name=args.profile)
    mfa_serial = args.serial_number or input('MFA 시리얼 넘버를 입력해주세요 (arn:aws:iam::{account}:mfa/{username}): ')
    mfa_token = args.token or input('6자리 MFA 코드를 입력해주세요: ')
    sts = session.client('sts')
    MFA_validated_token = sts.get_session_token(SerialNumber=mfa_serial, TokenCode=mfa_token)
    credentials = MFA_validated_token['Credentials']

    print(".env에서 사용할때")
    print(f"AWS_ACCESS_KEY_ID={credentials['AccessKeyId']}")
    print(f"AWS_SECRET_ACCESS_KEY={credentials['SecretAccessKey']}")
    print(f"AWS_SESSION_TOKEN={credentials['SessionToken']}")
```

아래와 같이 호출하여 자격증명 내용을 확인하세요.
``` bash
# 참고: --token은 MFA 앱에서 생성된 실제 6자리 코드로 대체하세요.
python ./get_aws_access_token.py --profile=test --serial-number=arn:aws:iam::11223344:mfa/gotp --token 1112222
```

### 2.ECS 서비스 옵션 활성화
ECS 서비스에서 `execute-command` 기능을 활성화해야 합니다. 아래 명령어를 사용해 옵션을 켜고, 서비스를 재배포합니다. 옵션을 비활성화 하려면 `--enable-execute-command` 대신 `--disable-execute-command` 을 사용하면 됩니다.

``` bash
# 클러스터 정보 확인 (현재 설정 상태 점검)
aws ecs describe-clusters --clusters your-cluster

# 옵션 활성화
aws ecs update-service \
--cluster your-cluster \
--service your-service \
--enable-execute-command

# 서비스 재배포 (변경 적용)
aws ecs update-service \
--cluster your-cluster \
--service your-service \
--force-new-deployment
```
> 필요 없을 경우 --disable-execute-command로 비활성화 가능.

### 3.IAM 인라인 정책 추가
Bastion 위에서 ECS Execute Command 기능을 사용할 것이며 Bastion의 IAM User에 인라인 정책 추가 했습니다.
CLI로 `execute-command`를 실행하려면 IAM 사용자에게 아래 정책을 추가해야 합니다. 이 정책은 ECS API 호출 권한을 부여합니다.

``` json
{
  "Version": "2012-10-17",
	"Statement": [
      {
	    "Sid": "VisualEditor0",
		"Effect": "Allow",
		"Action": "ecs:ExecuteCommand",
		"Resource": "*"
	  }
  ]
}
```

### 4.ECSTaskRole 인라인 정책 추가
ECS를 사용하고 있다면 S3, DynamoDB에 접근에 대한 정책 관리를 위해 TaskRole에 대한 IAM Role이 존재할 것입니다. 
ECS execute-command는 AWS Systems Manager(SSM)의 Session Manager 기능을 활용해 컨테이너 내부로 접속하기 위해 ssmmessages 관련 권한이 필요합니다.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
	  "Sid": "VisualEditor0",
	  "Effect": "Allow",
      "Action": [
	  	"ssmmessages:CreateDataChannel",
		"ssmmessages:OpenDataChannel",
		"ssmmessages:OpenControlChannel",
		"ssmmessages:CreateControlChannel"
	  ],
	  "Resource": "*"
	}
  ]
}
```


### 5.Amazon ECS Exec Checker 사용한 설정 확인
<img src="https://github.com/aws-containers/amazon-ecs-exec-checker/blob/main/demo.gif?raw=true" />
<a href="https://github.com/aws-containers/amazon-ecs-exec-checker">check-ecs-exec.sh</a>는 CLI 환경과 ECS 클러스터/태스크가 ECS Exec 기능을 사용할 준비가 되었는지 확인합니다. aws cli, jq 패키지가 필수입니다. 아래와 같이 호출합니다.
``` bash
./check-ecs-exec.sh <YOUR_ECS_CLUSTER_NAME> <YOUR_ECS_TASK_ID>
```

### 6.명령어 실행
서비스/태스크 확인을 위한 명령어입니다.
``` bash
aws ecs list-tasks --cluster your-cluster
aws ecs describe-services --cluster your-cluster --services your-service
aws ecs describe-tasks --cluster  your-cluster --tasks arn:aws:ecs:ap-northeast-2:123123:task/your-cluster/123123
```

컨테이너 쉘에 접속하는 명령어입니다.
``` bash
aws ecs execute-command \
--cluster your-cluster \
--region ap-northeast-2 \
--task your-task-id \
--container your-container \
--interactive  \
--command "/bin/sh"
```

### 7.마무리
ECS `execute-command`를 통해 컨테이너 내부에서 디버깅, 설정 조정 등을 쉽게 수행할 수 있습니다. 위 단계를 따라 설정을 완료했다면, 이제 자유롭게 컨테이너에 접속할 수 있습니다.