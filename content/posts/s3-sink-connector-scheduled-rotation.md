---
title: "S3 Sink Connector Scheduled Rotation"
description: ""
date: 2024-01-10T16:42:15+09:00
url: "/s3-sink-connector-scheduled-rotation/"
tags: [kafka]
---

S3 Sink Connector 에는 로테이션 일정을 설정할 수 있는 두 가지 속성이 있다.
`rotate.schedule.interval.ms` 와 `rotate.interval.ms` 속성이다.
이 두 속성이 어떻게 다른지 알아보자.

## rotate.schedule.interval.ms

각 파일의 타임스탬프는 첫 번째 레코드가 파일에 기록되는 **시스템 시간** 부터 시작된다.
파일의 타임스탬프부터 설정한 시간이 지난 이후에 레코드가 처리되면 파일이 flush 되고 S3에 업로드되며,
파일에 기록된 레코드의 오프셋이 커밋된다.

이 속성은 지속적인 데이터 스트림이 필요하지 않다.
`rotate.schedule.interval.ms` 가 3000 이라고 가정해보자.
이 경우 아래 표와 같이 3 개의 레코드가 유입된 이후에 더 이상 레코드가 유입되지 않아도 파일이 flush 되며 S3 에 업로드된다.

| Wall clock time | Message time  | Offset | Description   |
|-----------------|---------------|--------|---------------|
| 1706713200000   | 1706713200000 | 100    | consume topic |
| 1706713201000   | 1706713201000 | 101    | consume topic |
| 1706713202000   | 1706713202000 | 102    | consume topic |
| 1706713203000   | n/a           | n/a    | flush 시작      |

중요한 점은 `rotate.schedule.interval.ms` 속성을 사용하면 Exactly-once delivery 을 보장하지 않는는 것이다.
`rotate.schedule.interval.ms` 가 3000 이라고 가정하고 아래의 표를 보자.

| Wall clock time | Message time  | Offset | Description       |
|-----------------|---------------|--------|-------------------|
| 1706713200000   | 1706713200000 | 100    | consume topic     |
| ...             | ...           | 9000   | consume topic     |
| 1706713203000   | n/a           | n/a    | flush 시작          |
| 1706713203001   | 1706713203001 | 9001   | consume topic     |
| 1706713203002   | 1706713203001 | 9002   | consume topic     |
| 1706713203010   | n/a           | n/a    | S3 업로드 완료, 오프셋 커밋 |
| 1706713203011   | 1706713203011 | 9003   | consume topic     |
| 1706713203012   | 1706713203012 | 9004   | consume topic     |

이 경우 9001 - 9002 레코드는 연속되는 두 개의 파일에 모두 기록된다.
이런 경우 외에도 커넥터 재시작과 같이 동일한 동작을 유발할 수 있는 다른 조건도 존재 할 수 있다.

## rotate.interval.ms

이 속성도 `rotate.schedule.interval.ms` 와 비슷하지만,
각 파일의 타임스탬프는 첫 번째 레코드가 파일에 기록되는 **레코드의 시간** 부터 시작된다.
레코드의 시간은 Kafka Record Time, Record Field, Wall Clock extractor 등을 사용하여 지정할 수 있다.
중요한 점으로 이 속성을 사용하면 지속적인 데이터 유입이 필요하다는 것이다.
`rotate.interval.ms` 가 3000 이라고 가정하고 아래의 표를 보자.

| Wall clock time | Message time  | Offset | Description                    |
|-----------------|---------------|--------|--------------------------------|
| 1706713200000   | 1706713200000 | 100    | consume topic                  |
| 1706713201000   | 1706713201000 | 101    | consume topic                  |
| 1706713202000   | 1706713202000 | 102    | consume topic                  |
| 1706713203500   | 1706713203500 | 103    | rotate.interval.ms가 초과되어 flush |
| 1706713204000   | 1706713204000 | 104    | consume topic                  |

이 경우 104번 레코드 이후 메시지가 유입되지 않는다면 104 번 레코드는 S3에 업로드 될 수 없는 상태로 남아있다.

이 속성을 사용하면서 S3 Connector 가 기본적으로 제공하는 default pertitioner 와 field partitoner 를 사용하는 경우 Exactly-once delivery 를 보장한다.
하지만 TimeBasedPartitioner 를 사용한다면 TimestampExtractor 에 따라 Exactly-once delivery 를 보장하지 않을 수도 있다.
Exactly-once delivery 보장을 위해서는 `timestamp.extractor=Record` 또는 `timestamp.extractor=RecordField` 를 사용해야 한다.
([관련 문서](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html#s3-exactly-onc]))

![connect-s3-eos.png](https://docs.confluent.io/kafka-connectors/s3-sink/current/_images/connect-s3-eos.png)

## 참고

* [Amazon S3 Sink Connector Configuration Properties](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html)
* [Exactly-once delivery on top of eventual consistency](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html#s3-exactly-once)
* [Confluent S3 Sink Connector EOS](https://www.declarativesystems.com/2023/08/18/confluent-s3-sink-connector-eos.html)