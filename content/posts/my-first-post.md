---
title: "Kafka Idempotency в микросервисах"
date: 2026-02-16
draft: false
tags: ["kafka", "java"]
categories: ["backend"]
---

## Проблема

При повторной доставке события в Kafka возможна двойная обработка.

## Решение

```java
@KafkaListener
public void handle(Event event) {
    if (repository.exists(event.id())) {
        return;
    }
    repository.save(event);
}
```
