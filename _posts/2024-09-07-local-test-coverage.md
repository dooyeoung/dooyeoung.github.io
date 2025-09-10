---
layout: post
title: "SonarCloud 없이 로컬에서 Test Coverage 측정하고 리포트 생성하기"
categories: "Testing"
tags: ["testing", "coverage", "python", "pytest", "sonar"]
sidebar: ['article-menu']
---


## [필수] 커버리지 측정하기 및 결과 파일 만들기
```shell
coverage run manage.py test --settings=server.settings.test --keepdb
coverage combine # 생성 된 .coverage.* 파일들 하나로 합쳐줌
```


## 커맨드라인에서 모든 파일의 커버리지 확인하는 방법
```shell
coverage report # .coverage 파일을 기반으로 coverage 리포트 생성
```
coverage 의 기본 report, 커맨드라인에서 확인 가능하며, 모든 파일의 coverage 가 측정되며 현재 브랜치의 변경만 확인하기 어렵습니다.



## 현재 브랜치의 변경사항에 대한 커버리지 확인하는 방법
```shell
coverage xml # coverage.xml 파일 생성
diff-cover ./coverage.xml --compare-branch=master
```
[diff-cover](https://pypi.org/project/diff-cover/) 설치 필요하며, 커맨드라인에서 현재 브랜치의 변경에 영향을 받은 coverage 만 측정 가능 합니다.



## diff-cover html 보고서로 확인하는 방법
```shell
# coverage.xml 파일이 준비되어야 합니다.
diff-cover ./coverage.xml --compare-branch=master --html-report=diff-report.html
open diff-report.html
```

총 coverage 측정결과 및 개별 파일의 coverage 확인 가능합니다.

--html-report 옵션으로 html 파일로 coverage 리포트 생성, html 파일 열어서 리포트 확인
![](/assets/images/posts/test_coverage_01.png)

페이지 하단으로 가서 커버되지 않은 라인 확인 가능합니다.

![](/assets/images/posts/test_coverage_02.png)
