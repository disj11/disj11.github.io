---
title: "S3 Sink Connector Scheduled Rotation"
description: "S3 Sink Connector에서 파일 로테이션을 설정하는 두 가지 주요 속성, rotate.interval.ms와 rotate.schedule.interval.ms의 차이점과 동작 방식을 비교합니다. 지속적인 데이터 유입 여부, Exactly-once delivery 보장 조건 등 꼭 알아야 할 내용을 알아봅니다."
date: 2024-01-10T16:42:15+09:00
lastmod: 2024-12-29T10:00:00+09:00
url: "/s3-sink-connector-scheduled-rotation/"
tags: [kafka]
---

S3 Sink Connector에는 **파일 로테이션**을 설정할 수 있는 두 가지 주요 속성이 있습니다. 이번 포스트에서는 이 두 속성의 차이를 비교하고, 각각의 동작 방식을 자세히 살펴보겠습니다.

- `rotate.interval.ms`
- `rotate.schedule.interval.ms`

먼저 두 속성의 주요 차이를 표로 정리하였습니다.

|                       | **rotate.schedule.interval.ms**       | **rotate.interval.ms**                                                               |
|-----------------------|---------------------------------------|-------------------------------------------------------------------------------------|
| **기준 시간**           | 시스템 시간 기준                             | `timestamp.extractor`를 통해 설정 (Kafka Record Time, Record Field, Wall Clock 등)      |
| **지속적인 데이터 스트림 필요 여부** | 필요하지 않음                                | 필요함                                                                                 |
| **Exactly-once 보장 여부** | 보장되지 않음                                | 경우에 따라 보장됨                                                                      |

---

## **지속적인 데이터 유입**

### `rotate.schedule.interval.ms`

- 이 속성은 시스템 시간을 기준으로 일정 간격마다 파일을 플러시하고 S3에 업로드합니다.
- **지속적인 데이터 유입이 필요하지 않으며**, 설정된 시간이 지나면 자동으로 파일이 커밋됩니다.
- 사용하기 위해서는 반드시 `timezone` 속성을 설정해야 합니다.

예를 들어, `rotate.schedule.interval.ms` 값을 3000ms로 설정한 경우를 살펴보겠습니다.

| 시간 (time)  | 오프셋 (Offset) | 설명 (Description) |
|--------------|-----------------|--------------------|
| 1706713200000 | 100             | 토픽 데이터 수신       |
| 1706713201000 | 101             | 토픽 데이터 수신       |
| 1706713202000 | 102             | 토픽 데이터 수신       |
| 1706713203000 | n/a             | 파일 플러시 시작       |

위 예시에서, 데이터가 더 이상 들어오지 않더라도 지정된 시간(3000ms)이 지나면 파일이 플러시되고 업로드됩니다.

### `rotate.interval.ms`

- 이 속성은 첫 번째 레코드의 타임스탬프를 기준으로 파일의 타임스탬프 범위를 계산합니다.
- 이후 레코드의 타임스탬프가 범위를 초과하면 파일을 플러시하고 S3에 업로드합니다.
- **지속적인 데이터 유입이 필요**하며, 데이터가 중단되면 파일이 업로드되지 않고 열려 있는 상태로 남을 수 있습니다.

예를 들어, `rotate.interval.ms` 값을 3000ms로 설정한 경우를 살펴보겠습니다.

| 시간 (time)  | 오프셋 (Offset) | 설명 (Description)                        |
|--------------|-----------------|-------------------------------------------|
| 1706713200000 | 100             | 토픽 데이터 수신                           |
| 1706713201000 | 101             | 토픽 데이터 수신                           |
| 1706713202000 | 102             | 토픽 데이터 수신                           |
| 1706713204000 | 103             | 타임스탬프 범위 초과로 파일 플러시 및 업로드 시작 |
| 1706713205000 | 104             | 토픽 데이터 수신                           |

위 예시에서, 후속 레코드(Offset: 104)가 없으면 파일이 업로드되지 않을 가능성이 있으므로 주의가 필요합니다.

---

## **Exactly-once Delivery**

Exactly-once delivery를 보장하려면 기본적으로 `rotate.interval.ms`를 사용하는 것이 권장됩니다. 추가로 다양한 조건을 충족해야만 정확히 한 번만 데이터를 전달할 수 있습니다.

Confluent 문서에서는 아래와 같은 이미지를 제공하여 Exactly-once delivery 조건을 설명합니다:

![connect-s3-eos.png](https://docs.confluent.io/kafka-connectors/s3-sink/current/_images/connect-s3-eos.png)

---

## **결론**

두 속성은 서로 다른 목적과 환경에서 사용됩니다:
- **`rotate.schedule.interval.ms`**: 데이터 유입이 불규칙하거나 적은 경우 적합.
- **`rotate.interval.ms`**: 지속적인 데이터 유입이 보장되고 Exactly-once delivery가 필요한 경우 적합.

참고자료:
- [Amazon S3 Sink Connector for Confluent Platform](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html)
- [Amazon S3 Sink Connector for Confluent Cloud](https://docs.confluent.io/cloud/current/connectors/cc-s3-sink.html#)
- [Partitioning Records into S3 Objects](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html#partitioning-records-into-s3-objects)
- [Confluent S3 Sink Connector EOS](https://www.declarativesystems.com/2023/08/18/confluent-s3-sink-connector-eos.html)
