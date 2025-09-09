---
layout: post
title: "Cloudwatch Log와 Lambda를 이용해 슬랙으로 RDS Slow Query 알림 받기"
categories: "AWS"
tags: ["AWS", "Cloudwatch", "lambda", "rds"]
sidebar: ['article-menu']
---

이 글에서는 CloudWatch Log와 Lambda 함수를 연동하여 AWS RDS에서 발생하는 Slow Query를 감지하고 Slack으로 알림을 받는 방법을 정리합니다.


## 전체 흐름

전체적인 구조는 다음과 같습니다.

1.  RDS의 Slow Query 발생시 CloudWatch 에 기록합니다.
2.  CloudWatch Log에서 특정 패턴(Slow Query)에 대해 필터를 적용하여 Lambda 함수를 트리거합니다.
3.  Lambda 함수는 로그 내용을 파싱하여 Slack으로 메시지를 보냅니다.

## Slack Incoming Webhook URL 설정
먼저 Slack으로 메시지를 보내기 위해 Incoming Webhook을 생성해야 합니다. 

![](/assets/images/posts/slowquery_01.png)
![](/assets/images/posts/slowquery_02.png)
![](/assets/images/posts/slowquery_03.png)

## Lambda 함수 생성

AWS Lambda 콘솔에서 새로운 함수를 생성합니다. 런타임은 Node.js로 설정합니다.
![](/assets/images/posts/slowquery_04.png)
![](/assets/images/posts/slowquery_05.png)

```javascript
const https = require('https');
const zlib = require('zlib');
const SLOW_TIME_LIMIT = 3; // 3초이상일 경우 슬랙 발송
const SLACK_URL = 'WEBHOOK_URL';

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
        
        console.log(`result json = ${resultAscii}`);
        
        for(let i=0; i<resultJson.logEvents.length; i++) {
            const logJson = toJson(resultJson.logEvents[i], resultJson.logStream);
            console.log(`logJson=${JSON.stringify(logJson)}`);
        
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

// 타임존 UTC -> KST
function toYyyymmddhhmmss(timestamp) {

    if(!timestamp){
        return '';
    }

    function pad2(n) { return n < 10 ? '0' + n : n }

    var kstDate = new Date(timestamp + 32400000);
    return kstDate.getFullYear().toString()
        + '-'+ pad2(kstDate.getMonth() + 1)
        + '-'+ pad2(kstDate.getDate())
        + ' '+ pad2(kstDate.getHours())
        + ':'+ pad2(kstDate.getMinutes())
        + ':'+ pad2(kstDate.getSeconds());
}

function slackMessage(messageJson) {
    const title = `[${SLOW_TIME_LIMIT}초이상 실행된 쿼리]`;
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

exports.postSlack = async (message, slackUrl) => {
    return await request(exports.options(slackUrl), message);
}

exports.options = (slackUrl) => {
    const {host, pathname} = new URL(slackUrl);
    return {
        hostname: host,
        path: pathname,
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
    };
}

function request(options, data) {

    return new Promise((resolve, reject) => {
        const req = https.request(options, (res) => {
            res.setEncoding('utf8');
            let responseBody = '';

            res.on('data', (chunk) => {
                responseBody += chunk;
            });

            res.on('end', () => {
                resolve(responseBody);
            });
        });

        req.on('error', (err) => {
            console.error(err);
            reject(err);
        });

        req.write(JSON.stringify(data));
        req.end();
    });
}
```


## Lambda 테스트 설정
Cloudwatch Log에서 Lambda로 요청을 보낼 때는 Json형태의 데이터를 Gzip압축을 하여 보내도록 되어있어 사전 작업이 필요합니다.
```json
{
    "messageType": "DATA_MESSAGE",
    "owner": "611494913268",
    "logGroup": "/aws/rds/instance/lb-master/slowquery",
    "logStream": "lb-master",
    "subscriptionFilters": [
        "slow query"
    ],
    "logEvents": [
        {
            "id": "37793004157053145168628822406849282194742895186797658112",
            "timestamp": 1694696918000,
            "message": "# Time: 230914 13:08:38\n# User@Host: ryan-write[ryan-write] @  [10.0.10.72]\n# Thread_id: 166243068  Schema: test  QC_hit: No\n# Query_time: 5.001142  Lock_time: 0.000000  Rows_sent: 1  Rows_examined: 0\n# Rows_affected: 0  Bytes_sent: 56\nuse test;\nSET timestamp=1694696918;\nselect sleep(5);"
        }
    ]
}
```
Json을 데이터를 온라인 gzip 압축 사이트를 통해 압축하면 테스트 데이터를 얻을 수 있습니다.
``` text
H4sIAAAAAAAAA11RXW/TMBR9...==
```

AWS 콘솔의 람다 페이지로 돌아와서 람다 테스트를 진행합니다.
![](/assets/images/posts/slowquery_06.png)
![](/assets/images/posts/slowquery_07.png)
![](/assets/images/posts/slowquery_08.png)


## Cloudwatch와 Lambda 연동
![](/assets/images/posts/slowquery_09.png)
![](/assets/images/posts/slowquery_10.png)
![](/assets/images/posts/slowquery_11.png)
![](/assets/images/posts/slowquery_12.png)

구독 필터 패턴을 사용하여 알림을 받고자 하는 로그만 추출하는것이 중요합니다.
[구독 필터 관련 문서](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html) 를 참고하여 작성하였으며
SlowQuery 로그 발생시 `Query_time`이라는 단어가 포함되어 있어서 필터에 사용하였습니다.
![](/assets/images/posts/slowquery_13.png)
![](/assets/images/posts/slowquery_14.png)

실제 슬랙으로 전달된 알림 형태는 아래와 같습니다.
![](/assets/images/posts/slowquery_15.png)

