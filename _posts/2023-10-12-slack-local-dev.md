---
layout: post
title: "로컬에서 Slack 앱(App) 개발 환경 설정하기"
categories: "Development"
tags: ["slack"]
sidebar: ['article-menu']
---

Slack과 연동되는 기능을 개발할 때, 변경 사항을 매번 배포하지 않고 로컬 환경에서 테스트할 수 있다면 개발 효율이 크게 향상됩니다. 하지만 Slack의 Events API나 Interactivity 기능은 외부에서 접근 가능한 공개 URL을 요구하기 때문에, 로컬 환경(e.g. `localhost`)을 그대로 사용할 수 없습니다.

이 글에서는 `ngrok`을 사용하여 이 문제를 해결하고, 로컬에서 Slack 앱을 개발하고 테스트하기 위한 전체 설정 과정을 안내합니다.

### **1. ngrok 설치 및 실행**

`ngrok`은 로컬 서버를 외부에서 접근할 수 있는 안전한 터널을 통해 공개 URL로 노출시켜주는 도구입니다. 먼저 `ngrok`을 설치하고 아래 명령어를 실행하여 로컬 서버의 특정 포트(예: 8000)를 외부로 노출시킵니다.

```bash
ngrok http 8000
```

실행하면 `https://....ngrok-free.app` 과 같은 형태의 공개 URL을 얻을 수 있습니다. 이 URL을 이후 단계에서 사용하게 됩니다.

### **2. Slack 앱 생성 및 설정**

[Slack API 사이트](https://api.slack.com/apps)로 이동하여 새로운 앱을 생성하고, 다음과 같이 설정합니다.

-   **OAuth & Permissions:** 앱에 필요한 권한 범위(Scope)를 설정합니다. 예를 들어, 메시지를 보내려면 `chat:write` 권한이 필요합니다.
-   **Event Subscriptions:** Slack에서 발생하는 이벤트(예: 메시지 수신)를 구독하려면 이 기능을 활성화하고, **Request URL**에 위에서 발급받은 `ngrok` URL을 입력합니다.
-   **Interactivity & Shortcuts:** 사용자가 버튼을 클릭하는 등의 상호작용을 처리하려면 이 기능을 활성화하고, 마찬가지로 **Request URL**에 `ngrok` URL을 입력합니다.

![](/assets/images/posts/2024-01-12-slack-local-dev-1.png)

### **3. 환경 변수 설정**

Slack 앱의 **Basic Information** 페이지에서 `Client ID`, `Client Secret`, `Signing Secret` 값을 확인하여 로컬 개발 환경의 환경 변수 파일(`.env`)에 설정합니다.

```
# .env
SEND_SLACK=True
SLACK_APP_CLIENT_ID=...
SLACK_APP_CLIENT_SECRET=...
SLACK_SIGNING_SECRET=...
```

### **4. 로컬 환경에서 앱 연동**

이제 로컬에서 실행 중인 애플리케이션의 슬랙 연동 페이지(예: `https://your-local-domain/app/admin/integration`)에 접속하여, 개발용 Slack Workspace에 앱을 설치하는 과정을 진행합니다.

### **5. 테스트 메시지 발송**

연동이 완료되면, 간단한 파이썬 스크립트를 작성하여 실제로 메시지가 발송되는지 테스트할 수 있습니다. 아래 코드는 특정 사용자에게 리뷰 작성 요청 메시지를 보내는 예시입니다.

```python
import os
import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "server.settings.production")
django.setup()

from lemonbase.notification.infra.slack.messages.review_write_remind_slack_message import ReviewWriteRemindSlackMessage
from lemonbase.notification.infra.slack.slack_client_provider import SlackClientProvider
from lemonbase.notification.infra.slack.slack_message_sender import SlackMessageSender
from lemonbase.notification.application.services.slack_notification_app_service import SlackNotificationAppService


company_id = 1  # 테스트 회사 ID
person_id = 1   # 테스트 사용자 ID

client = SlackClientProvider.get_client(company_id)
slack_user = SlackNotificationAppService._get_slack_user(company_id, person_id)
slack_message = ReviewWriteRemindSlackMessage(
    channel=slack_user.channel_id, review_cycle_name='테스트 리뷰 사이클', cta_button_url="https://lemonbase.com"
)

SlackMessageSender.send(message=slack_message, client=client)
```

위와 같이 환경을 설정해두면, 코드를 변경할 때마다 새로 배포할 필요 없이 로컬에서 바로 Slack 연동 기능을 테스트하며 개발을 진행할 수 있습니다.
