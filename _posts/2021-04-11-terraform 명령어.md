---
layout: post
title: "Terraform 필수 명령어 가이드: init, plan, import, state"
categories: "DevOps"
tags: ["terraform", "devOps", "iac"]
sidebar: ['article-menu']
---

테라폼을 사용하여 인프라를 관리할때 자주 사용하는 패턴과 명령어를 정리합니다


## init
``` bash
terraform init
```
작업 디렉토리를 초기화 합니다. tfenv 를 사용하여 버전을 바꾸거나 workspace를 변경할때 실행해야 합니다.

--- 

## plan
작성한 테라폼 코드의 실행 결과를 확인해 볼 수 있습니다. 실제 인프라에 반영되지는 않습니다. 작업시 plan을 실행하여 변경내용을 자주 확인하는것이 좋습니다.
``` bash
terraform plan
```

--- 

## import
aws에 이미 생성된 리소스를 테라폼으로 옮겨올 수 있습니다. [문서 링크](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance#import)
``` bash
terraform import
```

### 준비

remote 저장소를 사용하고 로컬에서 import 명령어를 실행 시킬 경우 provider 설정을 수정해야 합니다. 아래와 같이 테라폼에서 aws에 접근 할 수 있도록 계정 정보를 변경합니다
``` terraform
provider "aws" {
    alias  = "ap_northeast_2"
    region = "ap-northeast-2"
    profile    = "test"   
    access_key = "AKAKAKAKAKAKAKAKAK"
    secret_key = "wmpS1wmpS1wmpS1wmpS1wmpS1"
}
```

그외 다른 provider를 사용하고 있다면 apikey를 직접 입력합니다.
``` terraform
provider "cloudamqp" {
    # apikey = var.cloudamqp_apikey
    apikey = "e61d888a-2c7a-2c7a-2c7a-e61d888a"
}
provider "sendgrid" {
    # api_key = var.sendgrid_api_key
    api_key = "SG.QXiXbA6JQaqh8iEh1gsNrg"
}
```

### 실행

이미 생성된 rds를 테라폼으로 옮겨오는 명령어 예시입니다. 테라폼 코드에 `test_rds` 에 대해 작성을 한 뒤 명령어를 실행합니다. 설정한 저장소에 state가 갱신되는것을 확인 할 수 있습니다.
``` bash
terraform import aws_db_instance.test_rds test_rds
```

---

## state list
현재 백엔드 저장소에 등록되어 관리되고 있는 리소스 목록을 확인 할 수 있습니다.
``` bash
terraform state list
```

---

## state show
테라폼으로 관리되고 있는 리소스의 상세 정보를 확인합니다.
``` bash
terraform state show [_RESOURCE_]
```

---

## state rm
테라폼으로 관리하고 있는 리소스를 삭제합니다. 실제 인프라가 삭제되는 것은 아닙니다. 
작업 중 `import` 명령어로 옮겨온 리소스를 지워야 할때가 아니면 자주 사용하지 않습니다.
``` bash
terraform state rm
```

---

## apply
테라폼 코드를 인프라에 반영합니다. `plan` 명령어를 실행 후 결과를 확인 한 다음 실행하는 것을 권장합니다.
``` bash
terraform apply
```

---


## 그 외 유용한 도구

- tfenv
  - 여러개의 workspace를 관리 할 경우 테라폼 버전이 다를 수 있으며 테라폼 버전 관리를 위해 `tfenv` 사용을 권장합니다.

- terraforming
  - 이미 생성된 aws 인프라 리소스를 테라폼 코드로 확인 할 수 있는 [프로젝트](https://github.com/dtan4/terraforming)입니다. `import` 명령어 없이 초기단계부터 테라폼으로 구축할 경우 큰 도움이 됩니다
