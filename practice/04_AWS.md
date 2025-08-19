# 📗 08.20 AWS
## CloudWatch를 활용해 서버 상태 Slack으로 알림
### 1. SNS 주제 생성

- 유형에서 표준 선택

### 2. Lambda 함수 생성

- 함수 생성 시 런타임 확인 (Node.js 16.x를 사용)
- Lambda 함수 코드 추가

```java
// 구성 -> 환경변수로 webhook을 받도록 함
const ENV = process.env
if (!ENV.webhook) throw new Error('Missing environment variable: webhook')

const webhook = ENV.webhook;
const https = require('https')

const statusColorsAndMessage = {
    ALARM: {"color": "danger", "message":"위험"},
    INSUFFICIENT_DATA: {"color": "warning", "message":"데이터 부족"},
    OK: {"color": "good", "message":"정상"}
}

const comparisonOperator = {
    "GreaterThanOrEqualToThreshold": ">=",
    "GreaterThanThreshold": ">",
    "LowerThanOrEqualToThreshold": "<=",
    "LessThanThreshold": "<",
}

exports.handler = async (event) => {
    await exports.processEvent(event);
}

exports.processEvent = async (event) => {
    console.log('Event:', JSON.stringify(event))
    const snsMessage = event.Records[0].Sns.Message;
    console.log('SNS Message:', snsMessage);
    const postData = exports.buildSlackMessage(JSON.parse(snsMessage))
    await exports.postSlack(postData, webhook);
}

exports.buildSlackMessage = (data) => {
    const newState = statusColorsAndMessage[data.NewStateValue];
    const oldState = statusColorsAndMessage[data.OldStateValue];
    const executeTime = exports.toYyyymmddhhmmss(data.StateChangeTime);
    const description = data.AlarmDescription;
    const cause = exports.getCause(data);

    return {
        attachments: [
            {
                title: `[${data.AlarmName}]`,
                color: newState.color,
                fields: [
                    {
                        title: '언제',
                        value: executeTime
                    },
                    {
                        title: '설명',
                        value: description
                    },
                    {
                        title: '원인',
                        value: cause
                    },
                    {
                        title: '이전 상태',
                        value: oldState.message,
                        short: true
                    },
                    {
                        title: '현재 상태',
                        value: `*${newState.message}*`,
                        short: true
                    },
                    {
                        title: '바로가기',
                        value: exports.createLink(data)
                    }
                ]
            }
        ]
    }
}

// CloudWatch 알람 바로 가기 링크
exports.createLink = (data) => {
    return `https://console.aws.amazon.com/cloudwatch/home?region=${exports.exportRegionCode(data.AlarmArn)}#alarm:alarmFilter=ANY;name=${encodeURIComponent(data.AlarmName)}`;
}

exports.exportRegionCode = (arn) => {
    return  arn.replace("arn:aws:cloudwatch:", "").split(":")[0];
}

exports.getCause = (data) => {
    const trigger = data.Trigger;
    const evaluationPeriods = trigger.EvaluationPeriods;
    const minutes = Math.floor(trigger.Period / 60);

    if(data.Trigger.Metrics) {
        return exports.buildAnomalyDetectionBand(data, evaluationPeriods, minutes);
    }

    return exports.buildThresholdMessage(data, evaluationPeriods, minutes);
}

// 이상 지표 중 Band를 벗어나는 경우
exports.buildAnomalyDetectionBand = (data, evaluationPeriods, minutes) => {
    const metrics = data.Trigger.Metrics;
    const metric = metrics.find(metric => metric.Id === 'm1').MetricStat.Metric.MetricName;
    const expression = metrics.find(metric => metric.Id === 'ad1').Expression;
    const width = expression.split(',')[1].replace(')', '').trim();

    return `${evaluationPeriods * minutes} 분 동안 ${evaluationPeriods} 회 ${metric} 지표가 범위(약 ${width}배)를 벗어났습니다.`;
}

// 이상 지표 중 Threshold 벗어나는 경우 
exports.buildThresholdMessage = (data, evaluationPeriods, minutes) => {
    const trigger = data.Trigger;
    const threshold = trigger.Threshold;
    const metric = trigger.MetricName;
    const operator = comparisonOperator[trigger.ComparisonOperator];

    return `${evaluationPeriods * minutes} 분 동안 ${evaluationPeriods} 회 ${metric} ${operator} ${threshold}`;
}

// 타임존 UTC -> KST
exports.toYyyymmddhhmmss = (timeString) => {

    if(!timeString){
        return '';
    }

    const kstDate = new Date(new Date(timeString).getTime() + 32400000);

    function pad2(n) { return n < 10 ? '0' + n : n }

    return kstDate.getFullYear().toString()
        + '-'+ pad2(kstDate.getMonth() + 1)
        + '-'+ pad2(kstDate.getDate())
        + ' '+ pad2(kstDate.getHours())
        + ':'+ pad2(kstDate.getMinutes())
        + ':'+ pad2(kstDate.getSeconds());
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

- 구성에서 환경 변수 선택 후 webhookurl 추가
- Lambda 테스트

```java
{
  "Records": [
    {
      "EventSource": "aws:sns",
      "EventVersion": "1.0",
      "EventSubscriptionArn": "arn:aws:sns:ap-northeast-2:981604548033:alarm-topic:test",
      "Sns": {
        "Type": "Notification",
        "MessageId": "test",
        "TopicArn": "arn:aws:sns:ap-northeast-2:123123:test-alarm-topic",
        "Subject": "ALARM: \"RDS-CPUUtilization-high\" in Asia Pacific (Seoul)",
        "Message": "{\"AlarmName\":\"TEST!!!\",\"AlarmDescription\":\"EC2 CPU 알람 (10% 이상 시)\",\"AlarmArn\":\"arn:aws:cloudwatch:ap-northeast-2:123123:alarm:ant-man-live-ALB-RequestCount-high\",\"AWSAccountId\":\"683308520328\",\"NewStateValue\":\"ALARM\",\"NewStateReason\":\"Threshold Crossed: 1 datapoint (10.0) was greater than or equal to the threshold (1.0).\",\"StateChangeTime\":\"2021-07-14T23:20:50.708+0000\",\"Region\":\"Asia Pacific (Seoul)\",\"OldStateValue\":\"OK\",\"Trigger\":{\"MetricName\":\"CPUUtilization\",\"Namespace\":\"AWS/EC2\",\"StatisticType\":\"Statistic\",\"Statistic\":\"MAXIMUM\",\"Unit\":null,\"Dimensions\":[{\"value\":\"i-0e3e982bf1c7f0910\",\"name\":\"EngineName\"}],\"Period\":300,\"EvaluationPeriods\":1,\"ComparisonOperator\":\"GreaterThanOrEqualToThreshold\",\"Threshold\":1.0}}",
        "Timestamp": "2021-06-07T10:51:39.536Z",
        "SignatureVersion": "1",
        "MessageAttributes": {}
      }
    }
  ]
}
```

### 3. Lambda와 SNS 연동

- 함수 개요에서 트리거 추가 후 SNS 주제 설정

### 4. SNS와 CloudWatch 경보 연동

- 경보 생성 후 EC2 검색
- EC2 선택 후 웹서버의 이름이나 인스턴스 ID 검색 후 CPUUtilization 찾은 후 선택
- 추가 구성에서 경보를 알릴 데이터 포인트 3/3 선택
`정상 → 경보` , `경보 → 정상`  생성 (장애가 해소되었는지 알 수 없음)
---
## 결과 확인
- 박세진 노션 : [PSJ REPOSITORY](https://psjrepository.notion.site/DAY-25-2543d86ddbdc80568b83fef67c889168)