---
title: "AWS SQS의 Queue 유형별 특성과 활용 방안"
description: "SQS의 Standard Queue와 FIFO Queue의 주요 특성, 차이점, 그리고 각각의 활용 사례를 상세히 알아봅니다. 시스템 설계 시 고려해야 할 중요한 특성들을 중심으로 설명합니다."
date: 2023-05-06T12:55:31+09:00
lastmod: 2024-12-30T16:37:00+09:00
tags: [TIL, AWS, SQS]
---

Amazon SQS(Simple Queue Service)는 두 가지 유형의 큐를 제공하고 있습니다. 각각의 특성과 사용 사례에 대해 자세히 알아보겠습니다.

## Standard Queue
Standard Queue는 무제한에 가까운 처리량(throughput)을 제공하는 것이 특징입니다. 다음과 같은 특성을 가지고 있습니다:

**메시지 전달 특성**
- At-least-once delivery: 메시지가 최소 한 번 이상 전달됨을 보장합니다. 때로는 동일한 메시지가 여러 번 전달될 수 있습니다.
- Best-Effort Ordering: 메시지 순서가 보장되지 않을 수 있으며, 상황에 따라 전송된 순서와 다르게 수신될 수 있습니다.

**주요 고려사항**
- 애플리케이션은 반드시 멱등성(idempotent)을 보장해야 합니다. 즉, 동일한 메시지가 여러 번 처리되더라도 시스템에 영향을 주지 않도록 설계되어야 합니다.
- 높은 처리량이 요구되는 시스템에 적합합니다.

## FIFO Queue
FIFO(First-In-First-Out) Queue는 메시지의 순서와 정확성을 보장하는 것이 특징입니다.

**메시지 전달 특성**
- Exactly-Once Processing: 메시지가 정확히 한 번만 처리됨을 보장합니다.
- First-In-First-Out Delivery: 메시지가 전송된 순서대로 정확하게 수신됨을 보장합니다.

**주요 사용 사례**
- 이벤트의 순서가 중요한 비즈니스 프로세스에 적합합니다.
- 정확한 순서 처리가 필요한 금융 거래나 주문 처리 시스템에 활용됩니다.

---

참고 사이트:
* https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-types.html