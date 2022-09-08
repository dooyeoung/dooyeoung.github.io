---
layout: post
title: "CircleCI에서 원격으로 helm 제어하기"
subtitle: EKS Private 환경에서 helm으로 배포하기
categories: "DevOps"
tags: ["CircleCI"]
sidebar: ['article-menu']
---

## 문제
AWS EKS API 설정을 Private 환경으로 변경하면서 helm 사용도 제한이 되었습니다.

## 해결방안
아래와 같은 순서로 CI가 실행되며 4번 단계에서 실패하게 됩니다.

1. 테스트 자동화
2. 이미지 빌드
3. AWS CLI를 사용하여 ECR에 이미지 업로드
4. helm을 사용하여 이미지 업그레이드

CircleCI 배포 단계에서 helm을 사용하고 있고 이미지 업그레이드에 실패하게 됩니다.
EKS API 를 Private 환경에서 사용하기 위해서는 같은 VPC 또는 연결된 VPC 환경에서만 접근 가능합니다.
EKS와 피어링된 VPC 환경에 위치한 Bastion EC2에서 helm 사용이 가능한 상태입니다.

`helm을 사용하여 이미지 업그레이드 ` 에서
`AWS ssm 을 사용하여 bastion ec2에 helm 명령어 전달하여 이미지 업그레이드` 로 수정하였습니다.

## 작업 내용
사용한 코드는 아래와 같습니다
기존 배포 과정에서 사용한 helm 명령어입니다
``` yaml
deploy:
    docker:
    - image: cimg/python:3.9
    working_directory: ~/repo
    environment:
      CHART_NAME: your_chart
      ENV: prod
      ECR_REPO_NAME: your_ecr_repo
    steps: &deploy
    - checkout
    - aws-cli/setup
    - helm/install-helm-client: *install-helm-client
    - aws-eks/update-kubeconfig-with-authenticator: *update-kubeconfig
    - attach_workspace:
        at: .
    - run:
        name: Upgrade helm chart
        command: |
          IMAGE_ID=$(cat "image_ids/${ENV}")
          cp "./helm/${ENV}.values.yaml" ./helm/values.yaml
          helm upgrade \
            "${CHART_NAME}" \
            ./helm/ \
            --set image.repository="${AWS_ECR_ACCOUNT_URL}/${ECR_REPO_NAME}" \
            --set image.id="@${IMAGE_ID}" \
            --description "commit ${CIRCLE_SHA1} from CircleCI ${CIRCLE_BUILD_NUM}"
```

`ENV`, `CHART_NAME`, `ECR_REPO_NAME` : 변수로 주입받은 값입니다.

`AWS_ECR_ACCOUNT_URL` : context 에 설정된 공용 변수입니다.

`CIRCLE_SHA1`, `CIRCLE_BUILD_NUM` : CircleCI에서 제공하는 값입니다.

`IMAGE_ID`: 는 빌드 과정에서 생성된 ECR 도커 이미지 id 입니다. 아래와 같은 작업에 의해 할당됩니다.

``` bash
mkdir -p image_ids
docker tag \
"${DOCKER_LATEST_TAG}" \
"${AWS_ECR_ACCOUNT_URL}/${ECR_REPO_NAME}:latest"
docker push \
"${AWS_ECR_ACCOUNT_URL}/${ECR_REPO_NAME}:latest" \
| tee push_log
grep -o 'sha256:\S*' push_log > "image_ids/${ENV}"
```


EKS 와 피어링된 VPC에 위치한 Bastion 으로 helm 명령어를 전달하는 것으로 배포과정을 수정하였습나다.
``` yaml
  deploy:
    docker:
      - image: cimg/python:3.8.8
    working_directory: ~/repo
    environment:
      CHART_NAME: your_chart
      ENV: prod
      ECR_REPO_NAME: your_ecr_repo
    steps: &deploy
    - checkout
    - attach_workspace:
        at: .
    - run:
        name: Install python package
        command: |
          python -m venv venv
          . venv/bin/activate
          pip install boto3
    - run:
        name: Upgrade helm chart in bastion ec2
        command: |
          IMAGE_ID=$(cat "image_ids/${ENV}")
          . venv/bin/activate
          python .circleci/helm_upgrade.py \
            --repository_url="${CIRCLE_REPOSITORY_URL}" \
            --chart_name="${CHART_NAME}" \
            --env="${ENV}" \
            --image_repository="${AWS_ECR_ACCOUNT_URL}/${ECR_REPO_NAME}" \
            --image_id="@${IMAGE_ID}" \
            --ci_number="${CIRCLE_BUILD_NUM}" \
            --commit_id="${CIRCLE_SHA1}"`
```

`aws ssm`을 사용해야 하기에 `boto3` 패키지를 설치합니다.
`helm_upgrade.py` 파이썬 스크립트를 실행합니다. 아래 코드는 `aws ssm`으로 명령어를 전달하는 함수입니다

``` python
def main(repository_url, commit_id, ci_number, env, image_id, image_repository, chart_name):
    ssm_client = boto_client(
        "ssm",
        aws_access_key_id=AWS_ACCESS_KEY_ID,
        aws_secret_access_key=AWS_SECRET_ACCESS_KEY,
        region_name=AWS_DEFAULT_REGION,
    )
    instance_id = 'i-01eb01eb01eb'
    repository_url = '__repository_url__'
    dir_name = '__dir_name__'

    commands = [
        'cd /home/ec2-user',
        # 깃헙에서 레포지토리 다운로드
        f'''
            if [ ! -d "/home/ec2-user/{dir_name}" ]
            then
                git clone "{repository_url}" /home/ec2-user
            fi
        ''',

        # 폴더 이동
        f"cd /home/ec2-user/{dir_name}",

        # helm 준비
        f"cp helm/{env}.values.yaml helm/values.yaml",

        # ec2-user 권한으로 helm 실행
        f'''
        runuser -l ec2-user -c "helm upgrade '{chart_name}' \
            /home/ec2-user/{dir_name}/helm/ \
            --set image.repository='{image_repository}' \
            --set image.id='{image_id}' \
            --description 'commit {commit_id} from CircleCI {ci_number}' "
        '''
    ]

    # aws ssm에 실행할 명령어를 전달합니다
    run_crawl_job = ssm_client.send_command(
        DocumentName="AWS-RunShellScript",
        Parameters={"commands": commands},
        InstanceIds=[instance_id],
        CloudWatchOutputConfig={
            'CloudWatchLogGroupName': 'session-manager-command',
            'CloudWatchOutputEnabled': True
        }
    )
    command_id = run_crawl_job.get("Command").get("CommandId")

    # 상태값을 계속 확인하여 성공 또는 실패시 확인을 중지합니다
    command_status = "Pending"
    while (
        command_status == "Pending"
        or command_status == "InProgress"
        or command_status == "Delayed"
    ):
        time.sleep(3)
        command_status = get_command_status(
            command_id,
            instance_id,
            ssm_client,
        )
```
전체코드는 [Gist](https://gist.github.com/dooyeoung/ca95d8cc504dd9071f710bb15eefafb2)에서 볼수 있습니다 

