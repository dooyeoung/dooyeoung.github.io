---
layout: post
title: "Cloudwatch Log와 Lambda를 이용해 슬랙으로 RDS Slow Query 알림 받기"
categories: "AWS"
tags: ["aws", "cloudwatch", "lambda", "rds", "slow-query", "slack"]
sidebar: ['article-menu']
---

이 글에서는 AWS RDS에서 발생하는 Slow Query를 실시간으로 감지하여 Slack으로 알림을 받는 방법을 소개합니다. CloudWatch Log와 Lambda 함수를 연동하여 간단하게 구현할 수 있습니다.

### **0. 전체 흐름**

전체적인 구조는 다음과 같습니다.

1.  RDS의 Slow Query 로그를 CloudWatch Log로 보냅니다.
2.  CloudWatch Log는 특정 패턴(Slow Query)이 감지되면 Lambda 함수를 트리거합니다.
3.  Lambda 함수는 로그 내용을 파싱하여 Slack으로 메시지를 보냅니다.

### **1. Slack Incoming Webhook URL 설정**

먼저 Slack으로 메시지를 보내기 위해 Incoming Webhook을 생성해야 합니다. Slack 앱 디렉토리에서 "Incoming WebHooks"를 검색하여 설정하고, 알림을 받을 채널을 지정한 뒤 Webhook URL을 복사해 둡니다.

![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-1.png)
![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-2.png)
![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-3.png)

### **2. Lambda 함수 생성**

#### **2.1 Lambda 함수 생성**

AWS Lambda 콘솔에서 새로운 함수를 생성합니다. 런타임은 Node.js로 설정합니다.

![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-4.png)

#### **2.2 Lambda 코드 작성**

다음은 CloudWatch Log 이벤트를 받아 Slack으로 메시지를 보내는 Lambda 함수의 코드입니다. `SLACK_URL` 부분에 위에서 복사한 Webhook URL을 입력해야 합니다.

```javascript
const https = require('https');
const zlib = require('zlib');
const SLOW_TIME_LIMIT = 3; // 3초이상일 경우 슬랙 발송
const SLACK_URL = 'YOUR_WEBHOOK_URL';

exports.handler = (input, context) => {
    var payload = Buffer.from(input.awslogs.data, 'base64');
    zlib.gunzip(payload, async(e, result) => {
        
        if (e) { 
            context.fail(e);
        } 
        
        const resultAscii = result.toString('ascii');
        let resultJson;

        try {
            resultJson = JSON.parse(resultAscii);
        } catch (e) {
            console.log(`[알람발송실패] JSON.parse(result.toString('ascii')) Fail, resultAscii= ${resultAscii}`);
            context.fail(e);
            return;
        }
        
        for(let i=0; i<resultJson.logEvents.length; i++) {
            const logJson = toJson(resultJson.logEvents[i], resultJson.logStream);
        
            try {
                const message = slackMessage(logJson);
                
                if(logJson.queryTime > SLOW_TIME_LIMIT) {
                    await exports.postSlack(message, SLACK_URL);
                }    
            } catch (e) {
                console.log(`slack message fail= ${JSON.stringify(logJson)}`);
                return;
            }

        }
    });
};

function toJson(logEvent, logLocation) {
    const message = logEvent.message;
    const messages = message.split('\n');
    const queryTimePattern = /^([\d.]+)/;
    
    return {
        "logLocation": logLocation,
        "currentTime": messages[0].split('Time: ')[1],
        "user": messages[1].split(':')[1],
        "queryTime": messages[3].split(':')[1].trim().match(queryTimePattern)[1],
        "query": messages[messages.length - 1]
    }
}

function slackMessage(messageJson) {
    const title = `[${SLOW_TIME_LIMIT}초이상 실행된 쿼리]`
    const message = `언제(UTC): ${messageJson.currentTime}\n로그위치:${messageJson.logLocation}\n계정: ${messageJson.user}\nQueryTime: ${messageJson.queryTime}초\n쿼리: ${messageJson.query}`;
    
    return {
        attachments: [
            {
                color: '#2eb886',
                title: `${title}`,
                fields: [
                    {
                        value: message,
                        short: false
                    }
                ]
            }
        ]
    };
}

// ... (postSlack, options, request functions are omitted for brevity)
```

#### **2.3 Lambda 테스트 설정**

CloudWatch Log에서 Lambda로 Gzip으로 압축된 JSON 데이터를 보냅니다. 테스트를 위해 이와 동일한 형태의 테스트 이벤트를 생성해야 합니다.

다음과 같은 JSON 데이터를 [온라인 Gzip 압축 사이트](https://www.multiutil.com/text-to-gzip-compress/)를 통해 압축하여 테스트 데이터로 사용합니다.

```json
{
    "messageType": "DATA_MESSAGE",
    "owner": "123456789012",
    "logGroup": "/aws/rds/instance/mydb/slowquery",
    "logStream": "mydb",
    "subscriptionFilters": [
        "slow_query_filter"
    ],
    "logEvents": [
        {
            "id": "37793004157053145168628822406849282194742895186797658112",
            "timestamp": 1694696918000,
            "message": "# Time: 230914 13:08:38\n# User@Host: myuser[myuser] @  [10.0.10.72]\n# Thread_id: 12345  Schema: my_db  QC_hit: No\n# Query_time: 5.001142  Lock_time: 0.000000  Rows_sent: 1  Rows_examined: 0\n# Rows_affected: 0  Bytes_sent: 56\nuse my_db;\nSET timestamp=1694696918;\nselect sleep(5);"
        }
    ]
}
```

압축된 데이터를 Lambda 테스트 이벤트의 `awslogs.data` 값으로 설정합니다.

```json
{
  "awslogs": {
    "data": "H4sIAAAAAAAAAD2Qy07DMBBF/8XReference... (생략)"
  }
}
```

#### **2.4 테스트 실행 & 디버깅**

생성한 테스트 이벤트를 실행하여 Lambda 함수가 정상적으로 동작하고 Slack 알림이 오는지 확인합니다. 오류가 발생하면 Lambda의 실행 로그를 통해 디버깅합니다.

### **3. Cloudwatch와 Lambda 연동**

#### **3.1 Cloudwatch Log > 구독 필터 설정**

RDS의 Slow Query 로그 그룹으로 이동하여 "구독 필터"를 생성합니다. 위에서 생성한 Lambda 함수를 구독 대상으로 지정하고, 필터 패턴을 설정하여 원하는 로그만 필터링할 수 있습니다.

![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-5.png)

### **4. 알림 정상 발송 확인**

모든 설정이 완료되면, 실제로 DB에서 Slow Query를 발생시켜 Slack 알림이 정상적으로 오는지 확인합니다.

```sql
select sleep(4);
```

![](/assets/images/posts/2023-03-25-cloudwatch-lambda-slack-6.png)

### **참고 문서**

- [RDS Slow 쿼리 Slack으로 알람 보내기](https://velog.io/@yyong3519/RDS-Slow-%EC%BF%BC%EB%A6%AC-Slack%EC%9C%BC%EB%A1%9C-%EC%95%8C%EB%9E%8C-%EB%B3%B4%EB%82%B4%EA%B8%B0)
