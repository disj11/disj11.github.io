---
title: "Sns Queue Types"
description: ""
date: 2023-05-06T12:55:31+09:00
tags: [AWS, SQS]
---

SQS 의 Queue type 에는 `Standard` 와 `FIFO` 가 있다.

Standard queues 는 At-least-once delivery, Best-Effort Ordering 으로 작동한다.
At-least-once delivery 는 적어도 한번 메시지가 전달된다는 의미로 같은 메시지가 경우에 따라 두 번 이상 전달될 수 있다.
Best-Effort Ordering 은 경우에 따라 메시지의 순서가 보장되지 않는 것을 의미한다.
이러한 특성으로 Standard queues 를 사용하는 어플리케이션은 멱등성(idempotent) 을 보장해야 한다.
높은 처리량이 필요한 어플리케이션에 주로 사용한다.

FIFO 는 Exactly-Once Processing, First-In-First-Out Delivery 로 작동한다.
메시지는 정확히 한 번 처리되며 메시지를 보내고 받는 순서가 보장된다.
이벤트 순서가 중요한 어플리케이션에 주로 사용한다.