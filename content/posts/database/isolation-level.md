---
title: Isolation level (격리 수준)
date: 2019-11-16T16:53:00+09:00
menu:
  sidebar:
    name: Isolation level (격리 수준)
    identifier: isolation-level
    parent: database
    weight: 10
---

## 개요

Isolation은 트랜잭션에서 일관성이 없는 데이터를 표시하는 방법입니다. isolation level(격리 수준)이 낮을수록 많은 사용자가 동일한 데이터에 동시에 접근할 수 있는 기능이 향상됩니다. 그러나 이는
잘 못된 데이터를 읽거나 데이터의 업데이트가 손실되는 등과 같은 증상을 유발할 수 있습니다.

반대로 isolation level이 높을수록 문제가 발생할 확률은 줄어들지만, 동일한 데이터에 동시에 접근할 수 있는 기능은 떨어집니다. 따라서 데이터베이스 시스템은 보통 네 종류의 isolation level을
정의하고, 알맞은 격리 수준을 선택할 수 있도록 하였습니다.

## 용어 정리

- Dirty Read  
  커밋되지 않은 데이터를 읽을 수 있을 때 발생합니다.

|Transaction 1|Transaction 2|
|---|---|
|`SELECT age FROM users WHERE id = 1;`||
|위 쿼리로 20의 값이 읽어짐||
||`UPDATE users SET age = 21 WHERE id = 1;`|
||나이를 21로 바꾸고 커밋은 하지 않음|
|`SELECT age FROM users WHERE id = 1;`||
|다시 나이를 읽으면 21살로 읽어짐||
||`ROLLBACK;`|
||Dirty Read 발생|

- Non-Repeatable Read  
  Transaction1 이 진행되는 동안 두 번 이상의 조회가 발생하고, 그 사이에 Transaction2 에서 데이터의 값을 변경하면 발생합니다.

|Transaction 1|Transaction 2|
|---|---|
|`SELECT * FROM users WHERE id = 1;`||
||`UPDATE users SET age = 21 WHERE id = 1;`<br>`COMMIT;`|
||나이를 21로 바꾸고 커밋|
|`SELECT * FROM users WHERE id = 1;`<br>`COMMIT;`||
|트랜잭션 진행중 조회 결과가 다르게 나옴||

- Phantom Read  
  트랜잭션 과정에서, 다른 트랜잭션에서 읽은 레코드에 새로운 행이 추가되거나 제거될 때 발생합니다.

|Transaction 1|Transaction 2|
|---|---|
|`SELECT * FROM users WHERE age BETWEEN 10 AND 30;`||
|10 ~ 30살 사이의 사용자 검색||
||`INSERT INTO users(id,name,age) VALUES ( 3, 'Bob', 27 );`<br>`COMMIT;`|
||27살 사용자 추가|
|`SELECT * FROM users WHERE age BETWEEN 10 AND 30;`<br>`COMMIT;`||
|Transaction2 에서 추가된 사용자가 같이 나옴.||

## Isolation Level

1. READ UNCOMMITTED  
   가장 낮은 레벨의 격리 수준으로 커밋하지 않은 데이터를 다른 트랜잭션에서 조회할 수 있습니다.

> **WARNING**: Dirty Read, Non-Repeatable Read, Phantom Read 가 발생할 수 있습니다.

2. READ COMMITTED 커밋 완료된 데이터만 다른 트랜잭션에서 조회할 수 있습니다.

> **WARNING**: Non-Repeatable Read, Phantom Read 가 발생할 수 있습니다.

3. REPEATABLE READ 트랜잭션 내에서 한번 조회한 데이터는 항상 같은 값으로 조회되는 격리 수준입니다.

> **WARNING**: Phantom Read 가 발생할 수 있습니다.

4. SERIALIZABLE 가장 높은 레벨의 격리 수준으로 위에서 알아본 Dirty Read, Non-Repeatable Read, Phantom Read 이 모두 발생하지 않습니다.

> **WARNING**: 이 수준의 격리 레벨은 동시성 처리 성능이 현저히 떨어집니다.

|Isolation Level|Dirty reads|Non-repeatable reads|Phantoms|
|:---|:---:|:---:|:---:|
|READ COMMITTED|V|V|V|
|READ COMMITTED||V|V|
|REPEATABLE READ|||V|
|SERIALIZABLE||||