---
title: "Isolation Level 에 대해 알아보자"
description: "데이터베이스의 격리 수준(Isolation Level)에 대하여"
date: 2021-09-28T20:30:02+09:00
tags: [database]
---

## 개요

Isolation level 은 트랜잭션에서 일관성이 없는 데이터를 어느 수준까지 허용할 것인지를 정의한다. isolation level 이 낮을수록 많은 사용자가 동일한 데이터에 동시에 접근할 수 있는 성능이 향상되지만, 이는 잘 못된 데이터를 읽거나 데이터의 업데이트가 손실되는 등과 같은 증상을 유발할 수 있다.

반대로 isolation level 이 높을수록 문제가 발생할 확률은 줄어들지만, 동일한 데이터에 동시에 접근할 수 있는 성능은 떨어진다. 따라서 데이터베이스 시스템은 보통 네 종류의 isolation level을 정의하고, 알맞은 격리 수준을 선택할 수 있도록 하였다.

## 용어 설명

격리 수준에 대해 알아보기 전에 `Dirty Read`, `Non-repeatable Read`, `Phantom Read` 가 무엇인지 알아야한다. 

### Dirty Read

커밋되지 않은 데이터를 다른 트랜잭션에서 읽을 수 있도록 허용할 때 발생한다.

| Transaction 1                         | Transaction 2                             |
|---------------------------------------|-------------------------------------------|
| `SELECT age FROM users WHERE id = 1;` |                                           |
|                                       | `UPDATE users SET age = 21 WHERE id = 1;` |
| `SELECT age FROM users WHERE id = 1;` |                                           |
|                                       | `ROLLBACK;`                               |

### Non-repeatable Read

한 트랜잭션 내에서 같은 쿼리를 두 번 수행할 때, 그 사이에 다른 트랜잭션이 값을 수정 또는 삭제함으로써 첫 번째와 두 번째 조회의 결과가 다르게 나타나는 현상을 말한다.

| Transaction 1                         | Transaction 2                             |
|---------------------------------------|-------------------------------------------|
| `SELECT age FROM users WHERE id = 1;` |                                           |
|                                       | `UPDATE users SET age = 21 WHERE id = 1;` |
|                                       | `COMMIT;`                                 |
| `SELECT age FROM users WHERE id = 1;` |                                           |
| `COMMIT;`                             |                                           |

### Phantom Read

한 트랜잭션 내에서 일정 범위의 레코드를 두 번 이상 읽을 때, 첫 번째 쿼리에서는 없었던 레코드가 이후의 쿼리에서 나타나는 현상을 말한다.

| Transaction 1                           | Transaction 2                                      |
|-----------------------------------------|----------------------------------------------------|
| `SELECT age FROM users WHERE age < 20`  |                                                    |
|                                         | `INSERT INTO users(name, age) VALUES ('홍길동', 10);` |
|                                         | `COMMIT;`                                          |
| `SELECT age FROM users WHERE age < 20;` |                                                    |
| `COMMIT;`                               |                                                    |

## Isolation level

앞서 설명한대로 데이터베이스에서는 보통 네 종류의 isolation level을 제공한다.

### Read Uncommitted

커밋되지 않은 데이터를 다른 트랜잭션이 읽는 것을 허용한다. `Dirty Read`, `Non-repeatable Read`, `Phantom Read` 현상이 발생한다.

### Read Committed

커밋된 데이터만 다른 트랜잭션이 읽는 것을 허용한다. `Non-repeatable Read` 와 `Phantom Read` 현상이 발생한다. 대부분의 DBMS가 기본모드로 채택하고 있는 모드이다.

### Repeatable Read

선행 트랜잭션이 읽은 데이터의 트랜잭션이 종료될 때까지 후행 트랜잭션은 이 데이터를 갱신하거나 삭제할 수 없게 한다. 같은 데이터를 여러 번 조회하여도 일관성 있는 결과를 보장한다. `Phantom Read` 현상이 발생한다.

### Serializable Read

선행 트랜잭션이 읽은 데이터를 후행 트랜잭션이 갱신하거나 삭제하지 못하게 할 뿐만 아니라, 새로운 레코드를 삽입하는 것도 막는다. 완벽한 읽기 일관성 모드를 제공한다.

