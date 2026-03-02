---
title: "Java Concurrency: вопросы, которые действительно проверяют уровень"
date: 2026-02-18
tags: ["java", "concurrency", "multithreading", "interview"]
draft: false
---

## Почему тема многопоточности — индикатор senior-уровня

Многопоточность — одна из самых сложных тем в backend-разработке.  
Она проявляется не только в `Thread`, но и в:

- высоконагруженных сервисах
- обработке Kafka
- транзакционной логике
- работе с БД
- асинхронных REST вызовах
- пуле соединений

На собеседованиях по Java concurrency редко спрашивают "что такое поток".  
Проверяют понимание **memory model, синхронизации, race condition и практических trade-off’ов**.

Разберём ключевые вопросы, которые чаще всего встречаются.

---

# 1. Thread vs Process

**Процесс** — независимый экземпляр программы с собственной памятью.  
**Поток** — легковесная единица выполнения внутри процесса.

| Процесс | Поток |
|----------|--------|
| Своя память | Общая память |
| Дорогой в создании | Лёгкий |
| IPC сложнее | Коммуникация проще |

В Java мы управляем потоками, а не процессами.

---

# 2. Способы создания потока

### 1️⃣ Наследование от Thread

```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Hello");
    }
}
❌ Плохая практика — ограничивает наследование.

2️⃣ Реализация Runnable
class MyTask implements Runnable {
    public void run() {
        System.out.println("Hello");
    }
}
✔ Предпочтительный способ.

3️⃣ ExecutorService (правильный production-подход)
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> doWork());
✔ Контроль пула
✔ Управление жизненным циклом
✔ Лучший вариант для backend

3. Runnable vs Callable
Runnable	Callable
Нет return	Возвращает результат
Нет checked exception	Может бросать exception
Используется чаще	Используется для async-логики
Callable<String> task = () -> "result";
Future<String> future = executor.submit(task);
4. Что такое Race Condition
Race condition возникает, когда несколько потоков изменяют общее состояние без синхронизации.

count++;
Это НЕ атомарная операция.

Она состоит из:

read

increment

write

Решения:

synchronized

Lock

AtomicInteger

Иммутабельность

5. synchronized — что он реально делает
synchronized:

блокирует монитор объекта

гарантирует visibility (happens-before)

обеспечивает mutual exclusion

public synchronized void increment() {
    count++;
}
Важно понимать:

это блокировка уровня объекта

возможны deadlock’и

есть накладные расходы

6. volatile — когда он нужен
volatile:

гарантирует visibility

НЕ гарантирует атомарность

Подходит для:

флагов остановки

read-heavy конфигурации

private volatile boolean running = true;
7. Deadlock
Deadlock — ситуация, когда потоки ждут друг друга бесконечно.

Пример:

Thread A: lock1 -> lock2
Thread B: lock2 -> lock1
Условия deadlock:

Mutual exclusion

Hold and wait

No preemption

Circular wait

Как избегать
единый порядок захвата локов

tryLock с таймаутом

минимизация вложенных блокировок

8. Difference: synchronized vs ReentrantLock
synchronized	ReentrantLock
Проще	Гибче
Нет tryLock	Есть tryLock
Нет fairness	Можно включить fairness
Автоматический unlock	Нужно вручную
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
Использовать ReentrantLock стоит там, где нужна гибкость.

9. Concurrent Collections
Обычные коллекции НЕ потокобезопасны.

❌ ArrayList
❌ HashMap

✔ ConcurrentHashMap
✔ CopyOnWriteArrayList
✔ BlockingQueue

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
ConcurrentHashMap не блокирует всю таблицу — только сегменты.

10. Thread Pool — зачем он нужен
Создание потоков дорого:

память

системные ресурсы

контекстные переключения

Thread pool:

переиспользует потоки

ограничивает параллелизм

защищает систему от перегрузки

Типы:

Fixed

Cached

Single

Scheduled

В backend почти всегда используется ThreadPoolExecutor напрямую или через Spring.

11. Future и CompletableFuture
Future — базовый уровень
Future<String> f = executor.submit(task);
f.get();
Блокирующий вызов.

CompletableFuture — современный подход
CompletableFuture.supplyAsync(this::callService)
    .thenApply(this::transform)
    .thenAccept(System.out::println);
✔ Композиция
✔ Асинхронность
✔ Нет callback hell

12. ExecutorService shutdown
Частая ошибка — забыть закрыть executor.

executor.shutdown();
или

executor.shutdownNow();
Иначе — утечки потоков.

13. Memory Model (коротко, но важно)
Java Memory Model определяет:

visibility

ordering

happens-before

Ключевые механизмы:

volatile

synchronized

final

Lock

atomic classes

Без понимания JMM невозможно писать корректный concurrent-код.

Практический вывод
Concurrency — это не про "запустить поток".

Это про:

управление состоянием

предсказуемость поведения

контроль параллелизма

баланс throughput vs latency

избежание subtle багов

В реальном backend-е чаще всего используются:

ThreadPoolExecutor

CompletableFuture

ConcurrentHashMap

@Async (Spring)

Kafka consumer concurrency

Optimistic locking

Что реально спрашивают на senior-собеседовании
Объясни happens-before

Когда использовать volatile?

Почему double-checked locking опасен?

Как работает ConcurrentHashMap?

Чем отличается synchronized от Lock?

Как избежать deadlock?

Что произойдёт при shutdownNow()?

Как ограничить concurrency в сервисе?

Итог
Если вы понимаете:

как работает memory model

где возможен race condition

как управлять пулом потоков

и какие инструменты использовать в разных сценариях

— вы действительно понимаете многопоточность, а не просто синтаксис.