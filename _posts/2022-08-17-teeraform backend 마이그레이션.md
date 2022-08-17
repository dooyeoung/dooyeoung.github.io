---
layout: post
title: "Terraform Backend를 S3로 변경"
subtitle: EKS Private 환경에서 terraform backend s3로 변경하기
categories: "DevOps"
tags: ["terraform"]
sidebar: ['article-menu']
---

`Terraform` 으로 `AWS` 인프라를 관리하고 있으며 `EKS API Endpoint` 접근 설정을 `Private`로 변경해야 했습니다.
`Terraform`에서 `K8S` 관련 리소스에 접근할때 EKS API Endpoint를 사용하므로 `Terraform Backend` 변경이 필요했습니다.

`EKS API Endpoint`를 `Private`으로 변경하게되면 EKS와 같은 `VPC` 또는 이와 연결된 `VPC`에서만 접근 가능합니다.
변경전에는 `Terraform Cloud`를 사용하고 있었으며 EKS와 연결된 `VPC`에 존재하는 Bastion에 `Terraform CLI`를 설치하하였습니다.
`Terraform Backend`는 `S3`와 `DynamoDB` 조합으로 구성하였습니다. 
`S3`에 `tfstate` 파일을 저장하고 `DynamoDB`에서 `State Lock` 을 실행하는 구조입니다.

운영환경과 개발환경을 AWS 계정을 분리하여 사용중입니다. `Terraform Workspace`를 `prod`와 `test`로 구분하여 사용하고 있습니다.
`Public` 환경에서 `Terraform Cloud`를 사용할때는 `AWS IAM` 설정만으로 편하게 접근 할 수 있었습니다.

그러나 `EKS` 설정과 State 저장소 `S3`의 접근을 `Private`으로 변경해야 하는 상황입니다.

`Backend`에서 `Workspace`를 관리하는 구조이지만
`Backend`를 공통으로 사용할수 없었기에 `prod` 환경의 `Terraform Backend`, `test` 환경의 `Terraform Backend` 를 다르게 설정하기로 하였습니다.


## S3, DynamoDB 추가

`Terraform` [공식문서](https://www.terraform.io/language/settings/backends/s3)를 참고하여 `S3`와 `Dynamo`를 추가합니다.
`S3`는 `Public` 접근을 제한하였으며 `Terraform`만 접근 하도록 정책을 추가하였습니다.

``` terraform
resource "aws_s3_bucket" "tfstate" {
    bucket = "your-terraform-tfstate"
    acl    = "private"

    versioning {
        enabled = true
    }
}

resource "aws_s3_bucket_ownership_controls" "tfstate" {
    bucket = aws_s3_bucket.tfstate.id

    rule {
        object_ownership = "BucketOwnerPreferred"
    }
}

resource "aws_s3_bucket_policy" "tfstate" {
    bucket = aws_s3_bucket.tfstate.id
    policy = jsonencode({
        Version = "2012-10-17",
        Statement = [
        {
            Effect   = "Allow"
            Action   = "s3:ListBucket"
            Resource = "${aws_s3_bucket.tfstate.arn}",
            Principal = {
            AWS = "arn:aws:iam::468446841302:user/terraform"
            }
        },
        {
            Effect   = "Allow",
            Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
            Resource = "${aws_s3_bucket.tfstate.arn}/test",
            Principal = {
            AWS = "arn:aws:iam::468446841302:user/terraform"
            }
        },
        ]
    })
}
```

공식문서를 참고하여 `state lock`에 사용할 `DynamoDB`도 추가합니다.
``` terraform
resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = "terraform-state-lock"
  hash_key     = "LockID"
  billing_mode = "PAY_PER_REQUEST"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Terraform Backend 설정

`Terraform Backend`에 사용할 `S3`와 `DynamoDB` 정보를 입력합니다

``` terraform
terraform {
    required_version = ">= 0.13"

    backend "s3" {
        key                  = "terraform.tfstate"
        region               = "ap-northeast-1"
        encrypt              = true
    }
...
```


## Terraform Backend 수정

`State` 저장소인 `Backend`가 변경되었으며 기존 State를 새로운 저장소로 복사하도록 초기화 명령어에 옵션을 추가하여 실행합니다.
`Workspace` 별 `Backend`를 구분하기 위해 `init` 명령어에 `-backend-config`를 추가하여 정보를 주입받도록 하였습니다.
``` bash
terraform init -migrate-state  \
    -backend-config="bucket=your-terraform-tfstate-test" \
    -backend-config="dynamodb_table=terraform-state-lock" \
    -backend-config="workspace_key_prefix=your-prefix"
```


## Bastion 정책 추가

`Bastion`에서 `Terraform CLI` 사용시 `Terraform state lock`이 실행되는데 이때 `DynamoDB`에 접근할 수 있도록
정책을 추가합니다.
``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:PutItem"
            ],
            "Resource": [
                "arn:aws:dynamodb:ap-northeast-1:468446841302:table/terraform-state-lock"
            ]
        }
    ]
}
```

정책을 추가하고 `Bastion`이 사용중인 `role`에 정책을 연결합니다.

<img class="post_img" src="/assets/images/posts/terraform_backend.png">


## Direnv 사용하여 환경변수 설정

`Terraform` 작업 디렉토리에서만 환경변수가 활성화 되도록 `direnv`를 사용합니다.
[공식문서](https://direnv.net/docs/installation.html)를 참고하여 설치합니다.

``` bash
curl -sfL https://direnv.net/install.sh | bash
chmod +x direnv
```

작업 디렉토리에 `.envrc` 파일을 생성합니다. `TF_VAR_`를 사용하면 `Terraform Variable`에서 사용할 수 있습니다.
`AWS` 사용자 토큰 정보도 설정하여 인프라 접근이 가능하게 합니다.

``` bash
export TF_VAR_grafana_google_oauth_client_id='257581726932-.....com'
export TF_VAR_grafana_google_oauth_client_secret='9Y83bQ...AA'
export TF_VAR_sendgrid_api_key='QXiXbA6JQaqh8iEh1...AAA'
export TF_VAR_cloudamqp_apikey='e61d888a...AAAA'
export TF_VAR_env='test'
export AWS_SECRET_ACCESS_KEY='AWS_SECRET_ACCESS_KEY'
export AWS_ACCESS_KEY_ID='AWS_ACCESS_KEY_ID'
export AWS_DEFAULT_REGION='ap-northeast-1'
```
`.envrc` 작성 후 `direnv allow`를 입력하여 디렉토리에서 환경변수를 감지할 수 있도록 설정합니다.

