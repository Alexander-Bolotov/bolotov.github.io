---
title: "wait(), notify() и notifyAll() в Java: низкоуровневая координация потоков"
date: 2026-03-02
lastmod: 2026-03-02
description: "Подробный разбор механизмов wait(), notify() и notifyAll() в Java: монитор, жизненный цикл потока, happens-before гарантии и типичные ошибки."
tags: ["java", "multithreading", "concurrency", "synchronization"]
categories: ["Java"]
draft: false
toc: true
---

# wait(), notify() и notifyAll() в Java

В Java существуют низкоуровневые механизмы межпоточной координации — методы `wait()`, `notify()` и `notifyAll()` класса `Object`. Они позволяют организовать взаимодействие потоков через монитор объекта и лежат в основе более высокоуровневых абстракций (`Lock`, `Condition`, `BlockingQueue`, `CountDownLatch` и др.).

В статье разберём:

- как работает монитор в Java;
- что происходит при вызове `wait()`;
- кого пробуждает `notify()`;
- когда нужен `notifyAll()`;
- типичные ошибки и best practices.

---

## 1. Монитор и intrinsic lock

Каждый объект в Java связан с монитором (monitor lock).

Когда поток входит в `synchronized`-блок:

```java
synchronized (lock) {
    // критическая секция
}
```

он захватывает монитор объекта `lock`. Пока монитор удерживается, другие потоки не могут войти в синхронизированные блоки по этому же объекту.

Это называется **intrinsic lock** или **monitor lock**.

---

## 2. Метод wait()

### Что делает wait()?

Когда поток вызывает:

```java
lock.wait();
```

происходит следующее:

1. Поток освобождает монитор объекта `lock`.
2. Переходит в состояние `WAITING`.
3. Ожидает вызова `notify()` или `notifyAll()` на том же объекте.
4. После пробуждения пытается повторно захватить монитор.
5. Только после успешного захвата выполнение продолжается.

⚠️ Важно: `wait()` можно вызывать **только внутри synchronized-блока**, иначе будет `IllegalMonitorStateException`.

---

### Пример: Producer / Consumer

```java
class Message {
    private String text;
    private boolean hasMessage = false;

    public synchronized void produce(String message) throws InterruptedException {
        while (hasMessage) {
            wait();
        }
        this.text = message;
        hasMessage = true;
        notify();
    }

    public synchronized String consume() throws InterruptedException {
        while (!hasMessage) {
            wait();
        }
        hasMessage = false;
        notify();
        return text;
    }
}
```

Обрати внимание:

- используется `while`, а не `if`;
- после изменения состояния вызывается `notify()`.

---

## 3. Почему обязательно while?

Использование `while` — критически важно.

Причины:

1. Возможны **spurious wakeups** (ложные пробуждения).
2. Может проснуться поток, для которого условие всё ещё ложно.
3. Между пробуждением и повторным захватом монитора состояние могло измениться.

Правильный шаблон:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
}
```

Это называется **guarded block pattern**.

---

## 4. Метод notify()

```java
lock.notify();
```

Что происходит:

- Пробуждается **один случайный поток** из wait-set данного объекта.
- Он переходит в состояние `BLOCKED`.
- Реально продолжит выполнение только после освобождения монитора.
- `notify()` **не освобождает монитор немедленно**.

### Кого именно пробуждает notify()?

Любой поток, ожидающий на этом объекте.  
Порядок выбора не гарантируется.

Если несколько потоков ждут разные условия — `notify()` может пробудить «не тот» поток.

---

## 5. Метод notifyAll()

```java
lock.notifyAll();
```

- Пробуждает **все** ожидающие потоки.
- Каждый из них повторно проверяет условие.
- Только те, для кого условие истинно, продолжат работу.

### Когда использовать notifyAll()

- Когда есть несколько разных условий ожидания;
- Когда логика сложная;
- Когда важнее корректность, чем микропроизводительность.

В production-коде чаще безопаснее использовать `notifyAll()`.

---

## 6. Жизненный цикл потока

При использовании wait/notify поток проходит состояния:

```
RUNNABLE → WAITING → BLOCKED → RUNNABLE
```

1. Поток выполняется (`RUNNABLE`).
2. Вызывает `wait()` → `WAITING`.
3. Кто-то вызывает `notify()`.
4. Поток становится `BLOCKED` (ждёт монитор).
5. После захвата монитора снова `RUNNABLE`.

---

## 7. wait(timeout)

Можно указать таймаут:

```java
lock.wait(1000);
```

Поток выйдет из ожидания:

- либо по `notify()`;
- либо по истечении 1000 мс.

---

## 8. InterruptedException

Метод `wait()` выбрасывает `InterruptedException`.

Правильная обработка:

```java
try {
    lock.wait();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Важно восстанавливать флаг прерывания.

---

## 9. Типичные ошибки

### ❌ Вызов wait() вне synchronized

Приведёт к `IllegalMonitorStateException`.

### ❌ Использование if вместо while

Логическая ошибка при конкурентном доступе.

### ❌ Использование notify() при сложной логике

Может привести к зависанию системы.

### ❌ Разные объекты для wait и notify

Ожидание и уведомление должны происходить на **одном и том же объекте**.

---

## 10. Когда лучше не использовать wait/notify

Это низкоуровневый механизм.

Современные альтернативы:

- `ReentrantLock` + `Condition`
- `BlockingQueue`
- `CountDownLatch`
- `CyclicBarrier`
- `Semaphore`
- `CompletableFuture`
- `ExecutorService`

Например:

```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>();
```

Во многих случаях это проще и безопаснее.

---

## 11. Сравнение с Lock + Condition

| wait/notify | Lock + Condition |
|-------------|------------------|
| Привязаны к Object | Несколько условий |
| Один wait-set | Несколько Condition |
| Менее гибко | Более управляемо |
| Низкий уровень | Современный API |

---

## 12. Happens-before гарантии

Согласно Java Memory Model:

- Выход из `synchronized` → happens-before следующего входа.
- `notify()` → happens-before завершение `wait()`.

Это гарантирует корректную видимость изменений между потоками.

---

# Вывод

`wait()`, `notify()` и `notifyAll()` — фундаментальный механизм межпоточной координации в Java.

Ключевые правила:

- Вызывать только внутри `synchronized`;
- Использовать `while`;
- Предпочитать `notifyAll()` при сложной логике;
- В новых проектах чаще использовать `java.util.concurrent`.

Понимание этих механизмов необходимо для:

- собеседований;
- чтения legacy-кода;
- глубокого понимания Java Memory Model;
- диагностики race condition и deadlock.