---
layout: post
title: "Lambda Layer 추가하여 패키지 사용하기"
categories: "DevOps"
tags: ["lambda"]
sidebar: ['article-menu']
---
lambda 로 작업시 python 기본 패키지 이외 다른 패키지를 사용시 패키지 설치가 필요합니다.
소스코드와 패키지 파일도 같이 업로드 하여 사용할 수 있지만 여러개의 lambda 함수를 사용중이라면 
layer를 추가하여 패키지를 공통으로 사용하는 방향으로 관리하는것이 좋다고 생각합니다.


## lambda 도커 이미지 준비
사용하려는 런타임 환경의 docker 이미지를 pull 합니다.
``` bash
docker pull lambci/lambda:build-python3.8
```

컨테이너를 실행합니다. 설치할 패키지 추출을 위해 volumn 옵션을 사용하였습니다.
``` bash
docker run --rm -it 
--volume /Users/user/Documents/lambda:/output \
--entrypoint '/bin/sh' \
--name="lambda38" \
lambci/lambda:build-python3.8
```

## 컨테이너에서 패키지 설치

컨테이너 실행 후 사용할 패키지를 적당한 위치에 설치합니다.
`lambda/python` 디렉토리르 생성하였으며 `requests` 패키지를 설치하였습니다. 설치 위치는 상관없으나 반드시 `python` 이름의 디렉토리 안에 패키지가 설치되어야 합니다. 
``` bash
mkdir -p lambda/python
cd lambda/python
pip install requests -t .
```

## 패키치 추출
설치 후 디렉토리르 python 디렉토리를 압축합니다.
``` bash
cd ..
zip -r requests.zip python/
```

`volume` 연결된 `/Users/user/Documents/lambda` 디렉토리에서 파일을 확인 할 수 있게 압축한 파일을 복사합니다. 
``` bash
cp requests.zip /output
```

## lambda layer 추가하기
aws lambda 페이지로 이동하여 layer를 추가하합니다.

<img class="post_img" src="/assets/images/posts/lambda_layer.png">

lambda 함수에서 layer 등록합니다.

<img class="post_img" src="/assets/images/posts/lambda_layer2.png">
