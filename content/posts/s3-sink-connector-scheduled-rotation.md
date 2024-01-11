---
title: "S3 Sink Connector Scheduled Rotation"
description: ""
date: 2024-01-10T16:42:15+09:00
url: "/s3-sink-connector-scheduled-rotation/"
tags: [kafka]
---

S3 Sink Connector 에는 로테이션 일정을 설정할 수 있는 두 가지 속성이 있다.
이번 포스트에서는 이 두 속성이 어떻게 다른지 비교하여 알아볼 것이다.

* `rotate.interval.ms`
* `rotate.schedule.interval.ms`

먼저 자세한 내용을 알아보기 전에 대표적인 차이를 표로 비교해보았다.

|                       | rotate.schedule.interval.ms | rotate.interval.ms                                                                    |
|-----------------------|-----------------------------|---------------------------------------------------------------------------------------|
| timestamp             | 시스템 시간                      | timstamp extractor 를 통해 설정 (Kafka Record Time, Record Field, or Wall Clock extractor) |
| 지속적인 데이터 스트림          | 필요하지 않음                     | 필요함                                                                                   |
| Exactly-once delivery | 보장되지 않음                     | 경우에 따라 보장됨                                                                            |

## 지속적인 데이터 유입

`rotate.schedule.interval.ms` 을 사용하기 위해서는 `timezone` 속성을 먼저 설정해야한다.
`rotate.schedule.interval.ms` 을 사용하는 경우 파일의 타임스탬프는 시스템의 시간으로 시작된다.
설정된 시간이 지나면 파일을 플러시하고 S3에 업로드 한 다음 해당 파일의 레코드 오프렛을 커밋한다.
이 속성을 사용하는 경우 **지속적인 데이터 유입**이 필요하지 않다.
이 속성의 값을 3,000 으로 설정했다고 가정해보자.

| time          | Offset | Description   |
|---------------|--------|---------------|
| 1706713200000 | 100    | consume topic |
| 1706713201000 | 101    | consume topic |
| 1706713202000 | 102    | consume topic |
| 1706713203000 | n/a    | flush 시작      |

3번째 데이터가 들어온 이후 데이터가 더 이상 들어오지 않는다 하여도 파일이 플러시된다.

`rotate.interval.ms` 을 사용하는 경우 파일의 타임스탬프는 파일에 첫번째 기록된 레코드의 타임스탬프로 시작된다.
레코드 타임스탬프는 `timestamp.extractor`를 통해 결정된다.
후속으로 들어온 레코드의 타임스탬프가 파일의 타임스탬프 범위를 벗어나게되면 파일을 플러시하고 S3에 업로드 한 다음 해당 파일의 레코드 오프셋을 커밋한다.
이러한 이유로 이 속성은 **지속적인 데이터 유입**이 필요하다.
이 속성의 값을 3,000 으로 설정했다고 가정해보자.

| time          | Offset | Description                    |
|---------------|--------|--------------------------------|
| 1706713200000 | 100    | consume topic                  |
| 1706713201000 | 101    | consume topic                  |
| 1706713202000 | 102    | consume topic                  |
| 1706713204000 | 103    | rotate.interval.ms가 초과되어 flush |
| 1706713205000 | 104    | consume topic                  |

위 표에서 4번째 데이터가 들어오면 레코드의 타임스탬프가 설정한 범위를 벗어났기 때문에 flush 가 발생할 것이다.
이후 5번째 데이터가 들어오고 더 이상 데이터가 들어오지 않는다면 파일이 S3 에 업로드되지 않고 파일이 계속 오픈 상태로 남을 수 있으므로 주의가 필요하다.

## Exactly-once delivery

Exactly-once delivery 를 보장하기 위해서는 기본적으로 `rotate.interval.ms` 속성을 사용해야 한다.
이 속성을 사용하면서 Default Kafka Partitioner 또는 Field Partitioner 를 사용하는 경우 Exactly-once delivery 가 보장된다.
만약 Time Based Partitioner 를 사용한다면 Timestamp Extractor 에 따라 Exactly-once delivery 가 보장될 수도, 그렇지 않을 수도 있다.

Time Based Partitioner 를 사용하면서 Exactly-once delivery 를 보장받기 위해서는 `timestamp.extractor=Record` 또는 `timestamp.extractor=RecordField` 를 사용해야한다.
Exactly-once delivery 를 보장받기 위한 방법이 복잡하기 때문에 [Exactly-once delivery on top of eventual consistency](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html#s3-exactly-once) 에 다음과 같은 이미지를 제공하고 있다.

![connect-s3-eos.png](https://docs.confluent.io/kafka-connectors/s3-sink/current/_images/connect-s3-eos.png)

이미지에 나온 Exactly-once delivery 을 위한 조건을 모두 만족시킬 때 Exactly-once delivery 를 보장받을 수 있다.

위에서 설명한 파티셔너에 대해 궁금하다면 [Partitioning Records into S3 Objects](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html#partitioning-records-into-s3-objects) 문서를 참고한다.

## 참고

* [Amazon S3 Sink Connector for Confluent Platform](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html)
* [Amazon S3 Sink Connector for Confluent Cloud](https://docs.confluent.io/cloud/current/connectors/cc-s3-sink.html#)
* [Confluent S3 Sink Connector EOS](https://www.declarativesystems.com/2023/08/18/confluent-s3-sink-connector-eos.html)