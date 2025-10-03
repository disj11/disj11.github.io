---
title: "Redis DEL 과 UNLINK 의 차이"
description: "Redis DEL 명령어와 UNLINK 명령어의 차이와, lazyfree-lazy-user-del 옵션에 대해 설명합니다."
date: 2023-10-20T20:57:17+09:00
lastmod: 2024-12-26T19:30:10+09:00
url: "/redis-del-command/"
tags: ["redis"]
---

Redis는 인기 있는 인메모리 데이터 구조 저장소로, 키 삭제를 위한 여러 방법을 제공합니다. 이 글에서는 `DEL`과 `UNLINK` 명령어의 차이점, 그리고 `lazyfree-lazy-user-del` 설정 옵션에 대해 살펴보며, Redis 키 삭제 전략을 최적화하는 데 도움을 드리고자 합니다.

## DEL 명령어

Redis의 `DEL` 명령어는 데이터베이스에서 키를 제거하는 기본적인 작업입니다. 한 번의 작업으로 여러 키를 삭제할 수 있으며, 시간 복잡도는 O(N)입니다. 여기서 N은 제거할 키의 수입니다.[^1]

### 시간 복잡도 분석

- 단순 키 타입(예: Strings): 키당 O(1)
- 복잡한 데이터 구조(예: Lists, Sets, Hashes): 키당 O(M), M은 데이터 구조 내 요소의 수

따라서, N개의 키를 삭제하는 총 시간 복잡도는 최악의 경우 O(N*M)로 표현될 수 있습니다.

### 성능 영향

Redis는 주로 단일 스레드로 작동하기 때문에, 큰 키나 여러 복잡한 데이터 구조를 삭제하는 것은 서버를 차단하여 전체 성능에 영향을 줄 수 있습니다. 이러한 차단 동작은 특히 고처리량 환경에서 문제가 될 수 있습니다.

## UNLINK 명령어

Redis 4.0에서 도입된 `UNLINK` 명령어는 키 삭제를 위한 비차단 대안을 제공합니다.[^2][^3]

### 주요 특징

1. **일정한 시간 복잡도**: 키 크기나 데이터 구조 복잡성에 관계없이 키당 O(1)
2. **비동기 삭제**: 키를 키스페이스에서 즉시 제거하고 실제 메모리 회수는 나중에 예약
3. **비차단**: 메모리가 해제되는 동안 Redis가 다른 명령을 계속 처리할 수 있음

### 내부 메커니즘

`UNLINK` 명령어는 두 단계로 작동합니다:

1. 키스페이스에서 키를 즉시 연결 해제합니다(O(1) 작업).
2. 실제 메모리 회수를 백그라운드 스레드에서 비동기적으로 수행하도록 예약합니다.

### 작은 키에 대한 최적화

모든 `UNLINK` 작업이 비동기적으로 처리되는 것은 아닙니다. 매우 작은 키의 경우, 비동기 삭제를 예약하는 오버헤드가 즉시 삭제 비용을 초과할 수 있습니다. Redis는 내부적으로 삭제 비용을 계산하고, 비용이 특정 임계값을 초과하는 경우에만 비동기적으로 처리합니다.

lazyfree.c 기반 간소화된 의사 코드[^4]

```c
if (deletion_cost > LAZYFREE_THRESHOLD) {
    schedule_async_deletion(key);
} else {
    delete_immediately(key);
}
```

## lazyfree-lazy-user-del 설정

기존 코드에서 `DEL`을 `UNLINK`로 변경하는 것이 실용적이지 않은 상황을 위해, Redis 6.0에서는 `lazyfree-lazy-user-del` 설정 옵션을 도입했습니다.

### 사용법

Redis 설정에서 `yes`로 설정하면:[^5]

```
lazyfree-lazy-user-del yes
```

이 옵션은 모든 `DEL` 명령어를 내부적으로 `UNLINK`로 처리하게 하여, 코드 변경 없이 비차단 삭제의 이점을 제공합니다.

## 결론

`DEL`과 `UNLINK`의 차이를 이해하는 것은 특히 큰 키나 고처리량 시나리오를 다룰 때 Redis 성능을 최적화하는 데 중요합니다. `DEL`은 작은 키와 단순한 데이터 구조에 적합한 반면, `UNLINK`는 더 큰 데이터 세트와 복잡한 구조에 상당한 이점을 제공합니다. `lazyfree-lazy-user-del` 옵션은 광범위한 코드 수정 없이 이러한 이점을 활용할 수 있는 편리한 방법을 제공합니다.

사용 사례를 신중히 고려하고 적절한 삭제 전략을 구현함으로써 Redis 기반 애플리케이션의 성능과 응답성을 크게 향상시킬 수 있습니다.

[^1]: https://redis.io/docs/latest/commands/del/
[^2]: https://redis.io/docs/latest/commands/unlink/
[^3]: http://redisgate.kr/redis/command/unlink.php
[^4]: https://github.com/redis/redis/blob/unstable/src/lazyfree.c
[^5]: https://redis.io/docs/latest/operate/oss_and_stack/management/config-file/