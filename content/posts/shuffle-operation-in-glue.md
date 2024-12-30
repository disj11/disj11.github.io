---
title: "Spark Shuffle과 Spill 현상 이해하기"
description: "3시간 걸리던 Glue Job을 17분으로 단축한 경험을 통해 Spark의 Shuffle 메커니즘과 Spill 현상을 이해하고, 효과적인 성능 최적화 방안을 알아봅니다."
date: 2023-05-10T18:58:24+09:00
lastmod: 2024-12-30T16:34:00+09:00
tags: [TIL, AWS, Glue]
---

AWS Glue workflow 사용 중 3시간 정도 걸리는 Glue Job이 발견되었습니다. Worker의 수를 10개에서 30개로 증가시키니 17분으로 드라마틱하게 단축되어, 이러한 현상이 발생한 원인을 분석해보았습니다.

## Spark Shuffle 이해하기

Spark에는 Shuffle Partition이라는 개념이 존재합니다. `join`, `groupBy`, `repartition` 등의 연산을 수행할 때 이 Shuffle Partition이 사용되며, 이는 Spark의 성능에 가장 큰 영향을 미치는 요소입니다.

**Shuffle Partition 특징**
- 기본값으로 200개의 파티션이 설정됩니다.
- 데이터 크기에 비해 파티션 수가 적으면 각 파티션의 부하가 증가합니다.
- 반대로 파티션 수가 너무 많으면 관리 오버헤드가 증가합니다.

## Shuffle Spill 현상

Shuffle 작업 중 가장 주의해야 할 현상이 바로 Shuffle Spill입니다. 이는 연산에 필요한 메모리가 부족할 때 발생하며, 다음과 같은 과정으로 진행됩니다:

1. 메모리 부족 상황 발생
2. 데이터를 직렬화하여 디스크에 임시 저장
3. 필요할 때 다시 역직렬화하여 메모리로 로드
4. 연산 재개

이러한 Shuffle Spill이 발생하면 다음과 같은 문제가 발생할 수 있습니다:
- Task 지연 및 에러 발생 가능성 증가
- 디스크 I/O로 인한 성능 저하
- 직렬화/역직렬화 과정에서의 추가적인 리소스 소모

## 성능 개선 방안

**Worker 수 증가의 효과**

Worker 수를 증가시켰을 때 성능이 크게 향상된 이유는 다음과 같습니다:
- 더 많은 Worker가 동시에 데이터를 처리하여 병렬 처리 능력이 향상됩니다.
- 각 Worker가 처리해야 하는 데이터양이 감소하여 Shuffle Spill 발생 가능성이 줄어듭니다.

**최적화를 위한 설정**

1. 메모리 관리
    - `spark.executor.memory`: Executor의 메모리 크기 설정
    - `spark.executor.memoryOverhead`: 추가 메모리 버퍼 설정

2. 파티션 관리
    - `spark.sql.shuffle.partitions`: Shuffle Partition 수 조정
    - 파티션 크기를 100MB~200MB 사이로 유지하는 것이 권장됩니다.

## AWS Glue의 S3 Shuffle

AWS Glue 3.0부터는 S3 기반 Shuffle 기능을 제공하여 다음과 같은 이점이 있습니다:
- 대규모 Shuffle 작업에서도 안정적인 성능 제공
- Worker 노드의 로컬 디스크 제한 극복
- 작업 실패 시에도 중간 Shuffle 데이터 보존

이러한 최적화 방법들을 적절히 활용하면 Spark 작업의 성능을 크게 향상시킬 수 있습니다. 특히 대용량 데이터를 처리할 때는 Shuffle Spill 현상을 주의 깊게 모니터링하고, 적절한 Worker 수와 메모리 설정을 통해 성능을 최적화하는 것이 중요합니다.

---

참고 사이트:

* https://aws.amazon.com/ko/blogs/big-data/introducing-amazon-s3-shuffle-in-aws-glue/
* https://tech.kakao.com/2021/10/08/spark-shuffle-partition/