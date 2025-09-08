---
layout: post
title: "SonarCloud 없이 로컬에서 Test Coverage 측정하고 리포트 생성하기"
categories: "Testing"
tags: ["python", "pytest", "coverage", "testing", "diff-cover"]
sidebar: ['article-menu']
---

테스트 커버리지는 코드의 품질을 가늠하는 중요한 지표 중 하나입니다. SonarCloud와 같은 정적 분석 도구는 PR 단위로 커버리지를 측정해주어 편리하지만, 코드를 푸시하기 전에 로컬 환경에서 미리 커버리지를 확인하고 싶을 때가 많습니다.

이 글에서는 `pytest-cov`와 `diff-cover` 라이브러리를 사용하여 로컬 환경에서 테스트 커버리지를 측정하고, 내가 수정한 코드에 대한 커버리지만을 골라 확인하는 방법을 소개합니다.

### **1. `pytest-cov` 설치 및 설정**

`pytest-cov`는 `pytest`에서 테스트 커버리지를 측정할 수 있게 해주는 플러그인입니다. 먼저 관련 라이브러리를 설치합니다.

```bash
pip install pytest pytest-cov
```

그 다음, 프로젝트 루트 디렉토리에 `pytest.ini` 파일을 생성하고 다음과 같이 커버리지를 측정할 소스 코드의 경로를 지정합니다.

```ini
[pytest]
cov-report = 
cov = ./lemonbase
```

### **2. Coverage 데이터 생성**

이제 `pytest`를 실행하여 테스트를 수행하고 커버리지 데이터를 생성합니다. `--cov` 옵션을 주면 `pytest-cov`가 동작하여 커버리지를 측정합니다.

```bash
pytest --cov
```

테스트가 병렬로 실행되는 경우, `.coverage.*` 형태의 파일이 여러 개 생성될 수 있습니다. `coverage combine` 명령어로 이 파일들을 하나로 합쳐줍니다.

```bash
coverage combine
```

### **3. Coverage 리포트 확인**

#### **전체 커버리지 확인**

`coverage report` 명령어로 전체 파일에 대한 커버리지를 터미널에서 간단히 확인할 수 있습니다.

```bash
coverage report
```

하지만 이 방법은 프로젝트 전체 커버리지를 보여주므로, 내가 수정한 부분의 커버리지만을 확인하기는 어렵습니다.

#### **변경된 코드의 커버리지만 확인하기 (`diff-cover`)**

`diff-cover`는 Git의 변경 사항을 기반으로 커버리지를 계산해주는 아주 유용한 도구입니다. 먼저 `diff-cover`를 설치하고, `coverage.xml` 파일을 생성합니다.

```bash
pip install diff-cover
coverage xml
```

이제 `diff-cover`를 실행하여 `master` 브랜치와 비교한 코드 변경분의 커버리지를 확인합니다.

```bash
diff-cover ./coverage.xml --compare-branch=master
```

#### **HTML 리포트로 시각적으로 확인하기**

`diff-cover`는 HTML 리포트 생성 기능도 제공합니다. `--html-report` 옵션을 사용하여 리포트를 생성하고, 브라우저에서 열어 확인할 수 있습니다.

```bash
diff-cover ./coverage.xml --compare-branch=master --html-report=diff-report.html
open diff-report.html
```

![](/assets/images/posts/2024-05-20-local-test-coverage-1.png)
*생성된 리포트에서 변경된 파일 목록과 각 파일의 커버리지를 한눈에 볼 수 있습니다.*

![](/assets/images/posts/2024-05-20-local-test-coverage-2.png)
*파일을 클릭하면 어떤 라인이 커버되지 않았는지 시각적으로 확인할 수 있습니다.*

이처럼 로컬 환경에서 `pytest-cov`와 `diff-cover`를 활용하면, PR을 올리기 전에 코드 품질을 스스로 점검하고 개선하는 데 큰 도움이 됩니다.
