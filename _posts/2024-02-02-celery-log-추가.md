---
layout: post
title: "Celery 작업 로그 추가하기"
categories: "Celery"
tags: ["celery", "log", "python", "django"]
sidebar: ['article-menu']
---

### **Celery 작업 로그 추가하기**

Celery를 사용하다 보면 어떤 작업이 언제 요청되었고, 성공적으로 실행되었는지 추적해야 할 때가 있습니다. 특히 worker에 문제가 발생했을 때, 어떤 작업이 유실되었는지 파악하기 어려울 수 있습니다.

이 문제를 해결하기 위해 Celery의 `before_task_publish` signal을 사용하여 작업이 broker에게 전달되기 전에 로그를 기록하는 방법을 소개합니다.

#### **1. 로그 저장을 위한 모델 추가**

먼저, Celery 작업 정보를 저장할 Django 모델을 추가합니다. `task_id`, `task_name`, `task_args`, `task_kwargs` 등을 저장할 수 있습니다.

```python
from django.db import models

from lemonbase.common.models import CreatedAtMixin, BaseModel


class CeleryLog(CreatedAtMixin, BaseModel):
    task_id = models.CharField(max_length=36)
    task_name = models.CharField(max_length=256)
    task_args = models.TextField()
    task_kwargs = models.TextField()

    class Meta:
        db_table = 'lb_celery_log'
        indexes = [
            models.Index(fields=['task_id']),
            models.Index(fields=['task_name']),
            models.Index(fields=['created_at']),
        ]
```

#### **2. `before_task_publish` signal을 이용한 로그 기록**

Celery는 작업 처리 과정에서 다양한 signal을 제공합니다. 그중 `before_task_publish`는 task가 broker로 publish 되기 직전에 호출되는 signal입니다. 이 signal을 이용하여 task 정보를 데이터베이스에 기록할 수 있습니다.

```python
import sentry_sdk
from celery.signals import before_task_publish

from lemonbase.celery.domain.models.celery_log import CeleryLog


@before_task_publish.connect
def task_before_task_publish_handler(sender, headers, **kwargs):
    try:
        CeleryLog.objects.create(
            task_name=sender,
            task_id=headers['id'],
            task_args=headers['argsrepr'],
            task_kwargs=headers['kwargsrepr'],
        )
    except Exception as exception:
        sentry_sdk.capture_exception(exception)
```

#### **활용 방안**

이렇게 기록된 로그는 다음과 같이 활용할 수 있습니다.

*   **유실된 작업 추적:** `lb_celery_log` 테이블과 Celery의 결과 저장 백엔드(e.g., `django_celery_results_taskresult`)를 비교하여, 요청은 되었지만 실행되지 않은 작업을 찾아낼 수 있습니다.
*   **수동 재실행:** 로그에 `task_name`, `task_args`, `task_kwargs`가 모두 기록되어 있으므로, 문제가 발생했을 때 엔지니어가 수동으로 작업을 재실행할 수 있습니다.
*   **성능 분석:** 작업 요청 시간(`created_at`)과 실제 작업 완료 시간을 비교하여, web -> broker -> worker 구간의 병목을 분석하고 성능을 개선하는 데 활용할 수 있습니다.
