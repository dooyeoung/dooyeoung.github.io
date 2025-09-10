---
layout: post
title: "Sentry 에러 리포트에 set_context로 풍부한 정보 추가하기"
categories: "DevOps"
tags: ["sentry", "error-tracking", "logging", "monitoring"]
sidebar: ['article-menu']
---

Sentry는 훌륭한 에러 트래킹 도구이지만, 때로는 에러만으로는 문제의 원인을 파악하기에 정보가 부족할 때가 있습니다. 예를 들어, 특정 사용자의 요청이나 특정 데이터 처리 과정에서만 발생하는 에러의 경우, 해당 시점의 컨텍스트 정보가 함께 기록된다면 디버깅이 훨씬 수월해질 것입니다.
이 글에서는 Sentry SDK의 `set_context` 기능을 사용하여 에러 리포트에 풍부한 추가 정보를 기록하는 방법을 소개합니다.

## set_context`로 추가 정보 기록하기

Sentry는 에러가 발생했을 때, 해당 에러와 관련된 구조화된 데이터를 함께 저장할 수 있는 `set_context` 기능을 제공합니다. `capture_exception`이 호출되기 전에 `set_context`를 사용하여 원하는 데이터를 추가할 수 있습니다.
다음은 `try...except` 구문에서 에러를 처리할 때, 디버깅에 필요한 추가 정보를 `error_details`라는 이름의 컨텍스트에 담아 Sentry로 보내는 예제입니다.

```python
import sentry_sdk

try:
    ...
except Exception as e:
    error_details = {
        "user_id": user.id,
        "company_id": company.id,
        "base_error_message": base_error_message,
        "analysis_report_publish_queue_id": analysis_report_publish_queue_id,
        "analysis_report_entity_id": analysis_report_entity_id,
        "stderr_output": stderr_output
    }

    # capture_exception 전에 호출해야 합니다.
    sentry_sdk.set_context("error_details", error_details)
    sentry_sdk.capture_exception(e)
```

이렇게 추가된 정보는 Sentry 대시보드의 **Contexts** 섹션에서 확인할 수 있어, 에러 발생 당시의 상황을 정확하게 파악하는 데 큰 도움이 됩니다.

![](/assets/images/posts/sentry_context_01.png)

## set_tags와 차이점

`set_context`와 유사하게 추가 정보를 기록할 수 있는 `set_tags`라는 기능도 있습니다. 둘의 가장 큰 차이점은 **검색 및 필터링 가능 여부**입니다.

-   **`set_tags`**: Key-Value 형태로 문자열을 저장하며, Sentry 대시보드에서 태그를 이용해 에러를 검색하거나 필터링할 수 있습니다. (예: `user.id:123`) 값은 최대 200자, 줄바꿈 문자 `\n` 불가합니다.
-   **`set_context`**: 복잡한 구조의 객체(JSON)를 저장할 수 있지만, 컨텍스트의 내용으로 에러를 검색하거나 필터링할 수는 없습니다. 오직 에러 상세 정보 페이지에서만 확인할 수 있습니다.

따라서 검색이나 필터링이 필요한 정보는 `set_tags`를, 에러 상세 내용을 풍부하게 만들기 위한 구조화된 데이터는 `set_context`를 사용하는 것이 좋습니다.

```python
sentry_sdk.set_tags({"page.locale": "de-at", "page.type": "article"})
```

