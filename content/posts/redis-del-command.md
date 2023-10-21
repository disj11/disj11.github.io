---
title: "Redis Del Command"
description: "Redis DEL 명령어와 UNLINK 명령어의 차이와 lazyfree-lazy-user-del 옵션에 대한 설명"
date: 2023-10-20T20:57:17+09:00
url: "/redis-del-command/"
tags: ["redis", "TIL"]
---

## DEL

Redis 의 `DEL` 명령어는 한번에 여러 key 를 삭제할 수 있으며 Time complexity 가 O(N) 이다. 여기에서의 N 은 제거할 키의 수이다.
만약 제거할 키의 데이터 타입이 `Strings` 이 아닌 경우 (e.g. list, set, hash...), 키에 포함된 모든 항목을 삭제해야한다. 때문에 개별 키를 삭제하는데에 O(M) 만큼의 시간이 걸린다.
결국 키를 제거하는데 총 O(N*M) 이 걸리게 된다. Redis 는 싱글 스레드로 작동하기 때문에 크기가 큰 key 를 삭제하는 경우 문제가 발생할 수 있다.

참고: [DEL](https://redis.io/commands/del/)

## UNLINK

위와 같은 이유로 Redis 에서 key 를 제거할 때에는 `UNLINK` 를 사용하는 것이 좋다.
`UNLINK` 는 key 의 크기와 상관 없이 각 키에 대해 O(1) 의 Time complexity 를 갖는다.
이는 키에 포함된 모든 항목을 삭제하는 `DEL` 과 다르게 key 의 연결만을 끊기 때문이다.
실제 메모리 회수를 위한 제거(O(N))는 다른 스레드에서 비동기적으로 이루어진다.

> 모든 UNLINK 명령어가 비동기로 실행되지는 않는다. 키의 크기가 너무 작은 경우는 별도 스레드에서 처리하는 것이 더 비용이 클 수 있기 때문이다.
> 이런 이유로 UNLINK 명령 실행시 내부적으로 삭제를 위한 비용을 계산하고, 그 비용이 큰 경우에만 비동기로 실행된다.
> 이에 대한 코드는 [lzayfree.c](https://github.com/redis/redis/blob/unstable/src/lazyfree.c#L170) 에서 확인할 수 있다.

참고: [Redis - UNLINK](https://redis.io/commands/unlink/), [Redis Gate - UNLINK](http://redisgate.kr/redis/command/unlink.php)

## lazyfree-lazy-user-del

`DEL` 호출을 `UNLINK` 호출로 모두 변경하는 것이 쉽지 않은 경우 `lazyfree-lazy-user-del` 옵션을 사용할 수 있다.
`lazyfree-lazy-user-del` 옵션을 `yes` 로 설정하면 `DEL` 명령어 사용시 내부적으로 `UNLINK` 로 작동한다. 이 옵션은 Redis 6.0 부터 사용 가능하다.

참고: [Configuration example](https://redis.io/docs/management/config-file/) 페이지에서 `lazyfree-lazy-user-del` 로 검색하여 확인