---
title: "Shuffle Operation in Glue"
description: ""
date: 2023-05-10T18:58:24+09:00
tags: [AWS, Glue]
---

Glue workflow 사용중 3시간 정도 걸리는 Glue Job 이 발견되었다.
Worker 의 수를 10 -> 30 으로 올리니 17분 정도로 드라마틱하게 단축되어 왜 이런 현상이 발생하였는지 찾아보았다.

Spark 에는 Shuffle Partition 이란 게 존재한다.
`join`, `groupBy` 등의 연산을 수행할 때 이 Shuffle Partition 이 사용된다.
이 Shuffle Partition 은 Spark의 성능에 가장 큰 영향을 미치는 Partition 이다.
연산에 쓰이는 메모리가 부족할 때 Shuffle Spill (데이터를 직렬화 하고 스토리지에 저장, 데이터 처리 이후에 역직렬화 후 연산 재개) 이 발생한다.
Shuffle Spill 이 일어나면, Task 가 지연되고 에러가 발생할 수 있다. 이 문제를 해결하기 위해서는 Core 당 메모리를 늘려야한다.

참고 사이트:

* https://aws.amazon.com/ko/blogs/big-data/introducing-amazon-s3-shuffle-in-aws-glue/
* https://tech.kakao.com/2021/10/08/spark-shuffle-partition/