---
layout: post
title: "로컬 개발 환경에서 Datadog APM 설정하고 사용하기"
categories: "APM"
tags: ["datadog", "apm", "docker", "local-development", "django"]
sidebar: ['article-menu']
---

Datadog과 같은 APM(Application Performance Monitoring) 도구는 보통 개발 서버나 프로덕션 환경에 적용하지만, 로컬 개발 환경에서도 APM을 활용하면 기능을 개발하는 단계에서부터 성능 문제를 미리 파악하고 코드를 최적화하는 데 큰 도움이 됩니다.

이 글에서는 Docker를 이용해 로컬 환경에 Datadog Agent를 실행하고, 로컬에서 실행 중인 Django 애플리케이션에 연결하여 APM 트레이스를 수집하는 방법을 소개합니다.

### **1. Datadog Agent 실행**

먼저 로컬 환경에서 Datadog Agent를 컨테이너로 실행해야 합니다. `docker-compose.yml` 파일에 다음과 같이 `datadog-agent` 서비스를 추가합니다.

```yaml
datadog-agent:
  image: gcr.io/datadoghq/agent:latest
  environment:
    DD_API_KEY: "YOUR_DATADOG_API_KEY"
    DD_SITE: "datadoghq.com"
    DD_APM_ENABLED: "true"
    DD_ENV: "local"
    DD_SERVICE: "your-service-name"
    DD_LOGS_ENABLED: "true"
    DD_APM_NON_LOCAL_TRAFFIC: "true"
  ports:
    - "8126:8126"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /proc/:/host/proc/:ro
    - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
```

-   `DD_API_KEY`: Datadog에서 발급받은 API 키를 입력합니다.
-   `DD_ENV`, `DD_SERVICE`: 트레이스를 구분하기 위한 환경과 서비스 이름을 지정합니다.

설정이 완료되면 아래 명령어로 Datadog Agent를 실행합니다.

```bash
docker-compose up -d datadog-agent
```

### **2. 애플리케이션 설정**

이제 로컬에서 실행할 애플리케이션이 Datadog Agent로 트레이스 정보를 보낼 수 있도록 환경 변수를 설정합니다.

```
# .env
DD_AGENT_HOST=localhost
DD_TRACE_ENABLED=true
```

### **3. 애플리케이션 실행**

Datadog의 트레이싱 라이브러리인 `ddtrace-run`을 사용하여 애플리케이션을 실행합니다. `ddtrace-run`은 애플리케이션 코드의 변경 없이 자동으로 주요 라이브러리(Django, Flask, Requests 등)의 요청을 트레이싱합니다.

```bash
# Makefile 예시
$ make dd-local

# ddtrace-run을 사용한 전체 실행 명령어
$ uv run ddtrace-run python manage.py runsslserver lemonbase.test:7777 --settings=server.settings.local
```

### **4. Datadog 대시보드에서 확인**

애플리케이션을 실행하고 몇 가지 API를 호출한 뒤, Datadog 대시보드의 **APM > Traces** 메뉴로 이동하면 로컬 환경에서 발생한 요청들이 트레이싱되는 것을 확인할 수 있습니다.

![](/assets/images/posts/2024-08-12-datadog-local-1.png)
*Datadog APM Traces 페이지에 로컬 환경의 트레이스가 표시됩니다.*

![](/assets/images/posts/2024-08-12-datadog-local-2.png)
*개별 트레이스를 클릭하면 DB 쿼리, 외부 API 호출 등 상세한 내역을 확인할 수 있습니다.*

**참고:** Datadog은 모든 요청을 트레이싱하지 않고 샘플링(sampling)을 할 수 있으므로, 요청을 보낸 직후에 트레이스가 보이지 않더라도 당황하지 말고 여러 번 요청을 보내 확인해보세요.
