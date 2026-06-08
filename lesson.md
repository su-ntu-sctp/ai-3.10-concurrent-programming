# Lesson: Java Multithreading

## Lesson Overview

This lesson introduces Java's concurrency model from the ground up, building toward the modern patterns you will encounter in production Java codebases. Rather than going through every API in depth, the goal is to give you a clear mental model of how Java handles concurrent work, why certain problems occur, and which tools to reach for depending on the situation.

**Prerequisites:** Basic Java knowledge (classes, interfaces, lambda expressions)

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Create** and manage threads using `Runnable` and lambda expressions
2. **Identify** race conditions and apply appropriate synchronization techniques including `ReentrantLock` and `ConcurrentHashMap`
3. **Use** `ExecutorService` and `CompletableFuture` for efficient concurrent programming
4. **Recognise** when to use virtual threads (Java 21) for scalable I/O-bound applications

---

## Part 1: Threads & Synchronization

### What is a Thread?

A running application is called a **process**. Every process has at least one thread — the **main thread** — which is the sequence of instructions the JVM executes when you run your program. A **thread** is the smallest unit of execution within a process: it has its own call stack and program counter, but shares the process's memory (heap) with all other threads in the same application.

When you create additional threads, the operating system scheduler decides when each one runs. You do not control the exact order — you can influence it, but never guarantee it. This is the fundamental reason concurrency is hard: **shared state + unpredictable scheduling = bugs that are difficult to reproduce.**

### Thread Lifecycle

A thread moves through a set of states during its lifetime. Understanding these helps you reason about what your threads are doing at any point.

| State | Description |
|---|---|
| **New** | Thread created but `start()` not yet called |
| **Runnable / Running** | After `start()` — ready to run or actively executing on a CPU core |
| **Waiting / Sleeping** | Paused — waiting for a condition, a lock, or a timed sleep to expire |
| **Terminated** | Execution has finished — the thread cannot be restarted |

---

### Creating Threads

In modern Java, the standard way to define a thread's work is with a `Runnable` lambda expression. `Runnable` is a functional interface — it has exactly one method (`run()`) — which means you can express a thread's task as a lambda, keeping the code concise and readable.

Create a `LearnThreads.java` and code along.

```java
// Define the task as a lambda
Runnable task = () -> {
    System.out.println(Thread.currentThread().getName() + ": working...");
};

// Pass it to a Thread and start it
Thread thread = new Thread(task);
thread.start();

// Or inline — common for simple one-off threads
new Thread(() -> System.out.println("Hello from thread")).start();
```

> **Important:** Always call `start()`, not `run()`. Calling `start()` is what actually creates a new OS thread and invokes `run()` on it. Calling `run()` directly just executes the method on the current thread — no new thread is created.

> **Note:** You may see older code that creates threads by extending the `Thread` class and overriding `run()`. This approach is considered legacy — it ties your task to the threading mechanism, prevents you from extending another class, and cannot be reused across different execution strategies. Implement `Runnable` (or use a lambda) and keep your task separate from how it is executed.

---

### Sleep and Interrupt

A thread can pause its own execution by calling `Thread.sleep(milliseconds)`. This is useful for polling, rate-limiting, or simulating delays in tests. Sleep throws `InterruptedException` — a checked exception you must handle — which fires if another thread calls `interrupt()` on the sleeping thread.

> **Note:** The following code goes inside the `main` method of `LearnThreads.java`.

> **Tip:** To insert an emoji on Windows press **Windows key + .** (period) to open the emoji picker.

```java
Thread t = new Thread(() -> {
    try {
        System.out.println("Going to sleep...");
        Thread.sleep(5000);
        System.out.println("Awake!");
    } catch (InterruptedException e) {
        System.out.println("Thread was interrupted early.");
    }
});

t.start();
t.interrupt(); // Wake it up early
```

---

### Race Conditions

A **race condition** occurs when two or more threads read and write the same shared data concurrently and the final result depends on the unpredictable order in which the threads are scheduled. The name is apt: the threads are literally racing to update the same variable, and whoever gets there last wins — which may not be what you intended.

Create a `RaceDemo.java` and code along.

Add a `BankAccount` class:

```java
class BankAccount {
    private double balance;

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        balance += amount;
        System.out.println("🟢 Deposited: $" + amount + ", Current Balance: $" + balance);
    }

    public void withdraw(double amount) {
        balance -= amount;
        System.out.println("🔴 Withdrawn: $" + amount + ", Current Balance: $" + balance);
    }
}
```

Now run four threads against the same account — two depositing, two withdrawing:

```java
BankAccount account = new BankAccount(1000);

Runnable depositTask  = () -> { for (int i = 0; i < 5; i++) account.deposit(100); };
Runnable withdrawTask = () -> { for (int i = 0; i < 5; i++) account.withdraw(200); };

new Thread(depositTask).start();
new Thread(depositTask).start();
new Thread(withdrawTask).start();
new Thread(withdrawTask).start();
```

Run this several times. You will see the final balance is inconsistent. The problem is that `balance += amount` is **not atomic** — it compiles to three steps: read balance, add amount, write balance back. Two threads can read the same value simultaneously, both add their amount, and both write back — one of the writes is lost.

---

### Synchronization

To fix a race condition, you need to ensure that only one thread at a time can execute the **critical section** — the code that reads and writes shared state. Java provides several mechanisms to do this.

#### The `synchronized` keyword

Adding `synchronized` to a method acquires the object's intrinsic lock when a thread enters the method, and releases it on exit. Any other thread that tries to enter a `synchronized` method on the same object must wait.

```java
public synchronized void deposit(double amount) {
    balance += amount;
    System.out.println("🟢 Deposited: $" + amount + ", Current Balance: $" + balance);
}

public synchronized void withdraw(double amount) {
    balance -= amount;
    System.out.println("🔴 Withdrawn: $" + amount + ", Current Balance: $" + balance);
}
```

Run the code again. The balance is now consistent every time. This is the simplest fix and works well for straightforward cases.

#### `ReentrantLock` — what you will see in production

`synchronized` is simple but inflexible. `ReentrantLock` from `java.util.concurrent.locks` gives you more control: you can attempt to acquire a lock without blocking forever (`tryLock()`), set a timeout, or lock and unlock in different methods. The lock/unlock pattern is always placed inside a `try/finally` block to guarantee the lock is released even if an exception is thrown.

Add the `BankAccountV2` class to `RaceDemo.java` and run the same four threads against it in `main` to verify the balance is consistent.

```java
import java.util.concurrent.locks.ReentrantLock;

class BankAccountV2 {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public BankAccountV2(double balance) {
        this.balance = balance;
    }

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        lock.lock();
        try {
            balance += amount;
            System.out.println("🟢 Deposited: $" + amount + ", Balance: $" + balance);
        } finally {
            lock.unlock(); // Always unlock in finally
        }
    }

    public void withdraw(double amount) {
        lock.lock();
        try {
            balance -= amount;
            System.out.println("🔴 Withdrawn: $" + amount + ", Balance: $" + balance);
        } finally {
            lock.unlock();
        }
    }
}
```

In `main`, run the same four threads as before but using `BankAccountV2`:

```java
BankAccountV2 account2 = new BankAccountV2(1000);

Runnable depositTask2  = () -> { for (int i = 0; i < 5; i++) account2.deposit(100); };
Runnable withdrawTask2 = () -> { for (int i = 0; i < 5; i++) account2.withdraw(200); };

new Thread(depositTask2).start();
new Thread(depositTask2).start();
new Thread(withdrawTask2).start();
new Thread(withdrawTask2).start();
```

#### `ConcurrentHashMap` — concurrent collections

Locking individual methods works for custom classes, but when you need a shared map across threads, reach for the concurrent collections in `java.util.concurrent`. The most commonly used is `ConcurrentHashMap` — a thread-safe drop-in replacement for `HashMap`. It achieves thread safety without locking the entire map: it partitions the map into segments and only locks the relevant segment during a write, making it far more performant than wrapping a `HashMap` in `synchronized`.

Add the following to `main` in `RaceDemo.java`:

```java
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// Thread-safe increment — no manual locking needed
// Call merge twice on "hello" to demonstrate the increment
wordCount.merge("hello", 1, Integer::sum);
wordCount.merge("hello", 1, Integer::sum);

// Only inserts if the key is not already present
wordCount.computeIfAbsent("world", k -> 0);

System.out.println(wordCount); // {hello=2, world=0}
```

**Quick reference — which tool to use?**

| Tool | Use when |
|---|---|
| `synchronized` | Simple cases, fine-grained locking on a single object |
| `ReentrantLock` | Need `tryLock()`, timeouts, or unlock in a different method |
| `ConcurrentHashMap` | Shared map across threads — always prefer over synchronizing a `HashMap` |

---

### 🧑‍💻 Activity **(10 minutes)**

Add a `calculateBigNumber()` method to `LearnThreads.java`:

```java
public static long calculateBigNumber() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) {
        result += i;
    }
    return result;
}
```

1. Run this method in two threads concurrently using lambda expressions.
2. Add a shared `ConcurrentHashMap<String, Long>` that stores each thread's result keyed by thread name.
3. Print the map after both threads complete.

---

## Part 2: Multithreading Using ExecutorService

### The Problem with Creating Threads Manually

Creating a raw `Thread` object is not free. The JVM must ask the operating system to allocate a native OS thread, which consumes roughly 512KB to 1MB of memory per thread and takes time to set up and tear down. If your application handles thousands of concurrent requests and spawns a new thread for each one, you will exhaust system memory long before your application logic becomes the bottleneck.

The solution is a **thread pool** — a fixed set of pre-created threads that stay alive and pick up tasks from a queue as they come in. Instead of creating and destroying threads repeatedly, the pool recycles them.

### ExecutorService

`ExecutorService` is the main interface for working with thread pools in Java. The `Executors` factory class provides several ready-made pool configurations.

Create a `LearnExecutors.java` and code along. All the following code goes inside `main`.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class LearnExecutors {
    public static void main(String[] args) {

        // Fixed pool of 5 threads — at most 5 tasks run simultaneously
        // Extra tasks queue up and wait for a thread to become free
        ExecutorService executorService = Executors.newFixedThreadPool(5);

        Runnable printLettersRunnable = () -> {
            System.out.println(Thread.currentThread().getName() + ": Looping through letters A to E");
            String[] letters = { "A", "B", "C", "D", "E" };
            for (String letter : letters) {
                System.out.println(Thread.currentThread().getName() + ": " + letter);
                try { Thread.sleep(1000); } catch (InterruptedException e) { }
            }
        };

        Runnable printSquaresRunnable = () -> {
            System.out.println(Thread.currentThread().getName() + ": Printing squares of 1 to 5");
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + i + "^2 = " + (i * i));
                try { Thread.sleep(1000); } catch (InterruptedException e) { }
            }
        };

        // Submit tasks — ExecutorService calls start() for you
        executorService.execute(printLettersRunnable);
        executorService.execute(printSquaresRunnable);
        executorService.execute(printLettersRunnable);

        // Always shut down — without this the thread pool keeps the JVM alive
        executorService.shutdown();
    }
}
```

Now try changing the pool size to 2 and observe how the output changes.

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
```

With only 2 threads, the third task has to wait until one of the first two finishes — you'll see a clear queuing effect in the output.

> **Pool sizing:** A common starting point for I/O-bound work is `number of cores × 2`. For CPU-bound work, match the number of available cores (`Runtime.getRuntime().availableProcessors()`). In practice, pool sizing is workload-specific and benefits from load testing. If you are on Java 21, virtual threads (Part 4) often remove the need to think about pool sizing at all for I/O-bound tasks.

---

### 🧑‍💻 Activity **(5 minutes)**

Change the pool size from 5 to 2 and run the code again. Observe which tasks run simultaneously and when the third task starts.

---

## Part 3: CompletableFuture

### Limitations of Runnable

`Runnable` works well for fire-and-forget tasks, but it has two significant limitations that make it unsuitable for the majority of real-world async work:

- **No return value.** `Runnable.run()` returns void. If your async task produces a result, there is no built-in mechanism to get it back to the calling code.
- **No exception propagation.** Exceptions thrown inside a thread's `run()` method are trapped there. The calling thread has no way to know something went wrong unless you build your own error-passing mechanism.

`CompletableFuture`, introduced in Java 8, solves both problems. It represents the **eventual result** of an asynchronous computation — a value that does not exist yet but will at some point in the future. You can chain operations onto it, handle errors declarably, and compose multiple futures together.

Create a `LearnCompletableFuture.java` and add these helper methods:

```java
// Method that does not return a value
public static void calculateBigNumber1() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) result += i;
    System.out.println(Thread.currentThread().getName() + ": Result: " + result);
}

// Method that returns a value
public static long calculateBigNumber2() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) result += i;
    return result;
}

// Method that throws an exception
public static long calculateBigNumber3() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) result += i;
    throw new RuntimeException("Something went wrong in calculateBigNumber3");
}
```

---

### `runAsync()` — For Tasks That Do Not Return a Value

Use `CompletableFuture.runAsync()` when your task does not need to return anything. It takes a `Runnable` and runs it asynchronously. By default it runs on the **common ForkJoinPool** — Java's shared pool of worker threads that backs most async operations in the JVM. You will see it in stack traces labelled as `ForkJoinPool.commonPool-worker-N`.

```java
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
    calculateBigNumber1();
});

// join() blocks the current thread until future1 completes
// Without this, main() may exit before the async task finishes
future1.join();
```

---

### `supplyAsync()` — For Tasks That Return a Value

Use `CompletableFuture.supplyAsync()` when your task returns a result. It takes a `Supplier` (a functional interface that returns a value).

**Default thread pool — ForkJoinPool:** When you do not specify a thread pool, `supplyAsync()` (and `runAsync()`) automatically uses `ForkJoinPool.commonPool()` — a shared pool the JVM creates at startup, sized to `number of CPU cores - 1`. You never need to create or manage it. You will see it in stack traces labelled as `ForkJoinPool.commonPool-worker-N`.

If you want to use your own pool instead — for example to isolate database calls from HTTP calls — pass an `ExecutorService` as the second argument:

```java
ExecutorService myPool = Executors.newFixedThreadPool(4);

CompletableFuture<Long> future = CompletableFuture.supplyAsync(() -> {
    return calculateBigNumber2();
}, myPool); // uses your pool, not ForkJoinPool

myPool.shutdown();
```

For most use cases the default `ForkJoinPool` is fine. Now back to the standard pattern — chain `thenAccept()` directly onto `supplyAsync()` and call `join()` on the entire chain, not just the `supplyAsync()` stage. Otherwise `thenAccept()` may not have executed by the time `join()` returns.

```java
CompletableFuture<Void> future2 = CompletableFuture.supplyAsync(() -> {
    return calculateBigNumber2(); // returns Long
}).thenAccept(result -> {
    System.out.println(Thread.currentThread().getName() + ": Result = " + result);
});

future2.join(); // waits for the entire chain including thenAccept()
```

> **Important:** Always call `join()` on the **last stage** of the chain, not the first. Calling `join()` on `supplyAsync()` alone only waits for the computation — `thenAccept()` is a separate stage that nobody is waiting for, so it may never print.

---

### `thenApply()` — Transform the Result

While `thenAccept()` consumes a result (returning void), `thenApply()` **transforms** a result and returns a new `CompletableFuture` with the transformed value. This is the method you will use most often when chaining async operations — each step takes the output of the previous step and produces new output.

**Chain order:** Always use `thenApply()` for intermediate transformation steps and `thenAccept()` as the final step to consume the result. `thenAccept()` returns `CompletableFuture<Void>` — nothing meaningful can be chained after it.

```
supplyAsync() → thenApply() → thenApply() → thenAccept()
```

```java
CompletableFuture<Void> future3 = CompletableFuture.supplyAsync(() -> {
    return calculateBigNumber2(); // returns Long
})
.thenApply(result -> {
    return "Formatted result: " + result; // Long -> String
})
.thenApply(formatted -> {
    return formatted.toUpperCase(); // String -> String
})
.thenAccept(finalResult -> {
    System.out.println(Thread.currentThread().getName() + ": " + finalResult); // consume and print
});

future3.join();
```

If you want to retrieve the final value rather than just print it, end with `thenApply()` instead of `thenAccept()` and call `join()` to extract it:

```java
String result = CompletableFuture.supplyAsync(() -> calculateBigNumber2())
    .thenApply(r -> "Result: " + r)
    .join(); // extracts the final String value

System.out.println(result);
```

| Method | Returns | Use when |
|---|---|---|
| `thenApply(fn)` | `CompletableFuture<T>` | Transform the result — chain continues |
| `thenAccept(fn)` | `CompletableFuture<Void>` | Consume the result — last step, no further chaining |
| `thenRun(fn)` | `CompletableFuture<Void>` | Run a follow-up action that ignores the result |

---

### `exceptionally()` — Handling Errors

When an exception is thrown inside any stage of a `CompletableFuture` chain, the exception propagates down and skips all `thenApply` and `thenAccept` stages until it reaches an `exceptionally()` handler. This gives you a clean, centralised place to handle errors without wrapping every stage in try/catch.

```java
CompletableFuture<Void> future4 = CompletableFuture.supplyAsync(() -> {
    return calculateBigNumber3(); // throws RuntimeException
})
.thenApply(result -> "Result: " + result)   // skipped due to exception
.thenAccept(System.out::println)            // skipped due to exception
.exceptionally(ex -> {
    System.out.println("Caught: " + ex.getMessage());
    return null;
});

future4.join();
```

---

### `allOf()` — Waiting for Multiple Futures

`CompletableFuture.allOf()` takes multiple futures and returns a new future that completes when all of them complete. A common pattern is to collect the results after all futures are done:

```java
CompletableFuture<Long> f1 = CompletableFuture.supplyAsync(() -> calculateBigNumber2());
CompletableFuture<Long> f2 = CompletableFuture.supplyAsync(() -> calculateBigNumber2());
CompletableFuture<Long> f3 = CompletableFuture.supplyAsync(() -> calculateBigNumber2());

// Wait for all three to complete
CompletableFuture.allOf(f1, f2, f3).join();

// Now safe to call join() on each — they are already done, no blocking
long total = f1.join() + f2.join() + f3.join();
System.out.println("Combined total: " + total);
```

---

### 🧑‍💻 Activity **(10 minutes)**

1. Write three `supplyAsync()` tasks, each returning a `Long` result.
2. Chain a `thenApply()` on each to format the result as a `String`.
3. Use `allOf()` to wait for all three, then collect and print each result.
4. Add an `exceptionally()` handler to one of the tasks by intentionally throwing an exception inside the supplier.

---

## Part 4: Introduction to Virtual Threads (Java 21)

### The Problem with Platform Threads at Scale

Traditional Java threads — called **platform threads** — are mapped 1-to-1 with operating system threads. Each one consumes roughly 512KB to 1MB of memory. In a typical web server handling 10,000 concurrent HTTP requests, that means 10,000 OS threads — roughly 10GB of memory just for thread stacks, before you have processed a single byte of actual data.

The historical workaround has been to use thread pools and non-blocking / reactive programming frameworks (like Project Reactor or RxJava). These work, but they force you to write code in a fundamentally different style — callbacks, reactive chains, and operators — that is difficult to read, debug, and maintain.

### What are Virtual Threads?

Virtual threads, introduced as a production feature in Java 21 as part of **Project Loom**, are a lightweight alternative managed entirely by the JVM rather than the operating system. They are extremely cheap to create — you can create tens of thousands or millions of them — and the JVM automatically multiplexes them onto a small number of OS threads.

The key insight is this: most threads in a typical web application spend the vast majority of their time **waiting** — waiting for a database response, a network call, a file read. Platform threads waste an OS thread during that waiting time. Virtual threads do not — when a virtual thread blocks on I/O, the JVM parks it and reassigns the underlying OS thread to another virtual thread that has work to do.

| | Platform Threads | Virtual Threads |
|---|---|---|
| **Managed by** | Operating system | JVM (Project Loom) |
| **Memory per thread** | ~512KB–1MB | A few KB |
| **Max practical count** | ~Thousands | Millions |
| **Best for** | CPU-bound tasks | I/O-bound tasks |
| **Code style** | Same as always | Same as always — no reactive rewrites needed |

> **Important:** Virtual threads are **not** faster for CPU-intensive work. If your threads are always busy doing computation (number crunching, image processing), a fixed platform thread pool sized to your CPU cores remains the right choice. Virtual threads shine specifically in I/O-bound scenarios where threads spend time waiting.

---

### Creating Virtual Threads

Create a `LearnVirtualThreads.java` and code along.

#### Method 1: `Thread.startVirtualThread()`

The simplest way — creates and starts a virtual thread in one call.

```java
Thread virtualThread = Thread.startVirtualThread(() -> {
    System.out.println("Running in: " + Thread.currentThread());
    try {
        Thread.sleep(1000);
        System.out.println("Virtual thread woke up!");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

virtualThread.join();
System.out.println("Main thread finished");
```

Notice the thread name includes `VirtualThread` in the output — this confirms it is a virtual thread, not a platform thread.

#### Method 2: `Thread.ofVirtual()`

For more control, such as setting a custom thread name.

```java
Thread virtualThread2 = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> {
        System.out.println("Running in: " + Thread.currentThread().getName());
        try {
            Thread.sleep(500);
            System.out.println("Task completed!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });

virtualThread2.join();
```

#### Method 3: Virtual Thread Executor (recommended for many tasks)

For running many tasks concurrently, use `Executors.newVirtualThreadPerTaskExecutor()`. Unlike `newFixedThreadPool`, there is no pool size limit — the JVM creates a new virtual thread per task and manages everything automatically.

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10; i++) {
        int taskNumber = i;
        executor.submit(() -> {
            System.out.println("Task " + taskNumber + " running on: " + Thread.currentThread());
            try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println("Task " + taskNumber + " completed");
        });
    }
} // try-with-resources auto-shuts down the executor
```

---

### Seeing the Scalability Advantage

Try creating 10,000 virtual threads — something that would exhaust memory or degrade severely with platform threads:

```java
long startTime = System.currentTimeMillis();

try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        executor.submit(() -> {
            try { Thread.sleep(100); } catch (InterruptedException e) { e.printStackTrace(); }
        });
    }
}

long elapsed = System.currentTimeMillis() - startTime;
System.out.println("10,000 virtual threads completed in " + elapsed + "ms");
```

---

### Virtual Threads in Spring Boot

If you are working in Spring Boot 3.2 or later, enabling virtual threads for all HTTP request handling requires a single configuration line:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

Spring Boot replaces the default Tomcat thread pool with virtual threads, meaning each incoming request gets its own virtual thread. No code changes, no reactive rewrites — the same blocking-style code you already write now scales to handle far more concurrent requests.

---

### 🧑‍💻 Activity **(5 minutes)**

Go back to your `LearnExecutors.java` from Part 2. Replace `Executors.newFixedThreadPool(5)` with `Executors.newVirtualThreadPerTaskExecutor()` and observe the difference in thread names in the output.

```java
// Before
ExecutorService executorService = Executors.newFixedThreadPool(5);

// After
ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor();
```

---

## Summary

| Concept | Key API | Use when |
|---|---|---|
| Raw threads | `new Thread(runnable).start()` | Simple one-off background tasks |
| Synchronization | `synchronized` / `ReentrantLock` | Protecting shared mutable state |
| Concurrent collections | `ConcurrentHashMap` | Shared data structures across threads |
| Thread pools | `ExecutorService` / `Executors` | Managing many reusable platform threads |
| Async with results | `CompletableFuture.supplyAsync()` | Async computation that returns a value |
| Chaining | `thenApply()` / `thenAccept()` | Composing async steps in a pipeline |
| Error handling | `exceptionally()` | Centralised async error handling |
| Parallel wait | `CompletableFuture.allOf()` | Waiting for multiple async tasks |
| Scalable I/O | Virtual threads (Java 21) | High-concurrency I/O-bound workloads |

---

END