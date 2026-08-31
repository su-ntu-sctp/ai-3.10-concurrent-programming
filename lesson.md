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

A thread is created, becomes ready to run, may pause while waiting for something, and finally terminates — and once terminated it cannot be restarted.

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

To fix a race condition, you need to ensure that only one thread at a time can execute the **critical section** — the code that reads and writes shared state.

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

#### `ReentrantLock` — the manual alternative

Both `synchronized` and `ReentrantLock` do the same job: let only one thread into the critical section at a time. The difference is **how the lock is turned on and off**. `synchronized` locks and unlocks **automatically** — it always locks the whole method, and that is your only option. `ReentrantLock` makes **you** do it manually, so you decide exactly where the lock starts and exactly where it ends. You could lock just a few lines in the middle of a long method, or lock in one method and unlock in another.

Comment out the two `synchronized` methods in `BankAccount` and add the lock versions underneath, so you can see both side by side. The `main` method stays exactly the same.

```java
import java.util.concurrent.locks.ReentrantLock;

class BankAccount {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public double getBalance() {
        return balance;
    }

    // ----- Version 1: synchronized (commented out) -----
    // public synchronized void deposit(double amount) {
    //     balance += amount;
    //     System.out.println("🟢 Deposited: $" + amount + ", Current Balance: $" + balance);
    // }

    // public synchronized void withdraw(double amount) {
    //     balance -= amount;
    //     System.out.println("🔴 Withdrawn: $" + amount + ", Current Balance: $" + balance);
    // }

    // ----- Version 2: ReentrantLock -----
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

The output is identical to the `synchronized` version — both fix the race condition the same way. What changes is control over the lock, not behaviour.

> **Why `finally` matters:** if an exception is thrown and `unlock()` never runs, the lock is held forever and every other thread waits forever — a frozen application. `synchronized` releases automatically, which is its main safety advantage.

#### `ConcurrentHashMap` — concurrent collections

A normal `HashMap` is **not safe** when several threads write to it at the same time — you get race conditions, just like the bank account. You could wrap a `HashMap` in `synchronized`, but that locks the **whole map**, so only one thread can touch it at a time — slow.

`ConcurrentHashMap` is a **thread-safe HashMap**. Its trick is that it does **not** lock the whole map on a write — it only locks the small part you are touching. So many threads can safely write to different parts at the same time. It is safe *and* fast, and it is a drop-in replacement for `HashMap`.

```java
import java.util.concurrent.ConcurrentHashMap;

// Thread-safe map — many threads can put/get at the same time safely
ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();

scores.put("Alice", 10);
scores.put("Bob", 20);

// Reading and updating works just like a normal HashMap
System.out.println("Alice's score: " + scores.get("Alice"));

scores.put("Alice", 15); // update Alice's score

System.out.println(scores); // {Alice=15, Bob=20}
```

You use it exactly like a normal `HashMap` (`put`, `get`, and so on) — the only difference is that it is safe when many threads use it at once.

**Quick reference — which tool to use?**

| Tool | Use when |
|---|---|
| `synchronized` | Simple cases — automatic locking on a single object |
| `ReentrantLock` | You need to control exactly where the lock starts and ends |
| `ConcurrentHashMap` | Shared map across threads — always prefer over synchronizing a `HashMap` |

---

### Other Concurrent Collections

`ConcurrentHashMap` is the one you will use most, but every common collection has a thread-safe version in the same `java.util.concurrent` package. `CopyOnWriteArrayList` is a thread-safe list, best when you read often and write rarely. `CopyOnWriteArraySet` and `ConcurrentHashMap.newKeySet()` are thread-safe sets. `ConcurrentLinkedQueue` and `LinkedBlockingQueue` are thread-safe queues — a blocking queue is what sits inside every `ExecutorService`, holding tasks that are waiting for a free thread. `ConcurrentSkipListMap` is the thread-safe version of `TreeMap` when you need keys kept in sorted order. You do not need to memorise these; the rule is what matters — if a collection is shared across threads, use the concurrent version instead of the plain one.

> **Important:** Making a collection thread-safe protects the collection itself — adding, removing, reading. It does not protect the objects inside it. If two threads take the same `Customer` out of a thread-safe list and both change its balance, that is still a race condition.
---

### 🧑‍💻 Activity **(10 minutes)**

You will build a shared counter that several threads increment at the same time, and see why synchronization matters.

1. Create a `Counter` class with a private `int count` field and an `increment()` method that does `count++`. Add a `getCount()` method.
2. In `main`, create one shared `Counter`. Start **two threads**, each running a lambda that calls `increment()` 10,000 times in a loop.
3. Use `join()` to wait for both threads to finish, then print the final count. Run it a few times — you will see a value **less than 20,000** (often around 17,000–18,000) because `count++` is not atomic (a race condition). Notice the number is different on every run.
4. Now fix it: make `increment()` a `synchronized` method (or protect `count++` with a `ReentrantLock`). Run again — the result should be exactly **20,000** every time, with no variation at all.

> **Note:** `join()` throws the checked exception `InterruptedException`, so your `main` method needs `throws InterruptedException` (or wrap the `join()` calls in a try/catch).

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

**Watch what happens when the pool shrinks.** Change the pool size from 5 to 2 and run it again:

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
```

With only 2 threads, the third task cannot start immediately — it **queues** and waits until one of the first two finishes. Look at the thread names in the output: you will only ever see two of them working at a time, and one of them picks up the third task once it is free. The pool size caps how many tasks run at once; extra tasks are not lost, they simply wait their turn.

---

## Part 3: CompletableFuture

### Limitations of Runnable

`Runnable` works well for fire-and-forget tasks, but it has two significant limitations that make it unsuitable for the majority of real-world async work:

- **No return value.** `Runnable.run()` returns void. If your async task produces a result, there is no built-in mechanism to get it back to the calling code.
- **No exception propagation.** Exceptions thrown inside a thread's `run()` method are trapped there. The calling thread has no way to know something went wrong unless you build your own error-passing mechanism.

`CompletableFuture`, introduced in Java 8, solves both problems. It represents the **eventual result** of an asynchronous computation — a value that does not exist yet but will at some point in the future.

**What "asynchronous" means:** normally your program waits at each slow line before moving on. Asynchronous means **start the work in the background and carry on** — two things happen at once. Later, when you actually need the result, you call `join()` to wait for it.

Create a `LearnCompletableFuture.java` and add these helper methods:

```java
// Method that returns a value
public static long calculateBigNumber() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) result += i;
    return result;
}

// Method that throws an exception
public static long calculateBigNumberError() {
    long result = 0;
    for (long i = 0; i < 1_000_000_000; i++) result += i;
    throw new RuntimeException("Something went wrong");
}
```

---

### `supplyAsync()`, `thenApply()`, `thenAccept()` and `exceptionally()`

`CompletableFuture.supplyAsync()` runs a task in the background and returns a result. By default it runs on the **common ForkJoinPool** — a shared pool of worker threads the JVM creates at startup. You never need to create or manage it.

From there you chain steps onto the result:

- **`thenApply()`** takes the value, **transforms** it, and passes a new value along — the chain continues.
- **`thenAccept()`** takes the value, **consumes** it (for example prints it), and returns nothing — the chain ends.
- **`exceptionally()`** catches any error thrown anywhere in the chain.

Think of it as a factory line: `thenApply` is a station that changes the product and passes it on; `thenAccept` is the last station that packs it away.

```java
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
    return calculateBigNumber();          // produces a Long
})
.thenApply(result -> {
    return "Result: " + result;           // transforms Long -> String
})
.thenAccept(finalResult -> {
    System.out.println(finalResult);      // consumes and prints — chain ends
})
.exceptionally(ex -> {
    System.out.println("Caught: " + ex.getMessage());
    return null;
});

future.join(); // waits for the entire chain to finish
```

> **Important:** Always call `join()` on the **last stage** of the chain, not the first. Calling `join()` on `supplyAsync()` alone only waits for the computation — the later stages are separate and may never run.

> **Why is this `CompletableFuture<Void>` and not `<Long>`?** The type is decided by the **last step**. `supplyAsync()` produces a `Long`, but `thenAccept()` consumes that value and returns nothing — so the whole chain ends up as `<Void>`.

> **Why `return null` in `exceptionally()`?** It must return a value of the same type as the chain. Here the chain is `<Void>`, so `null` is the only thing it can return.

| Method | Returns | Use when |
|---|---|---|
| `thenApply(fn)` | `CompletableFuture<T>` | Transform the result — chain continues |
| `thenAccept(fn)` | `CompletableFuture<Void>` | Consume the result — last step, no further chaining |
| `exceptionally(fn)` | `CompletableFuture<T>` | Catch any error raised earlier in the chain |

---

### 🧑‍💻 Activity **(5 minutes)**

See what happens when a stage in the chain fails.

1. In the chain you just wrote, change the supplier to call `calculateBigNumberError()` instead of `calculateBigNumber()`.
2. Run it again.

You will see the error message printed instead of the result. Notice what happened: the exception **skipped** both `thenApply()` and `thenAccept()` entirely and went straight to `exceptionally()`. That is the whole point — one place to handle errors, instead of a try/catch around every stage.

> **Note:** The message will read `java.util.concurrent.CompletionException: java.lang.RuntimeException: Something went wrong`. `CompletableFuture` wraps the original exception, so the text is longer than what was thrown. Nothing is broken.

---

## Part 4: Introduction to Virtual Threads (Java 21)

### The Problem with Platform Threads at Scale

Traditional Java threads — called **platform threads** — are mapped 1-to-1 with operating system threads. This does not mean two threads are created: it means one Java `Thread` object is permanently glued to one real OS thread. Each one consumes roughly 512KB to 1MB of memory. In a typical web server handling 10,000 concurrent HTTP requests, that means 10,000 OS threads — roughly 10GB of memory just for thread stacks, before you have processed a single byte of actual data.

The historical workaround has been to use thread pools and non-blocking / reactive programming frameworks (like Project Reactor or RxJava). These work, but they force you to write code in a fundamentally different style — callbacks, reactive chains, and operators — that is difficult to read, debug, and maintain.

### What are Virtual Threads?

Virtual threads, introduced as a production feature in Java 21 as part of **Project Loom**, are a lightweight alternative managed entirely by the JVM rather than the operating system. They are extremely cheap to create — you can create tens of thousands or millions of them — and the JVM automatically multiplexes them onto a small number of OS threads.

The key insight is this: most threads in a typical web application spend the vast majority of their time **waiting** — waiting for a database response, a network call, a file read.

Here is the crucial difference. A **platform thread keeps its OS thread even while it is waiting** — locked, idle, and wasted. A **virtual thread gives its OS thread back** when it starts waiting, so another virtual thread with real work can borrow it. When the waiting is over, the virtual thread is rescheduled onto whichever OS thread is free. That reuse is why a handful of OS threads can serve millions of virtual threads.

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
public static void main(String[] args) throws InterruptedException {

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
}
```

Notice the output includes `VirtualThread` — this confirms it is a virtual thread, not a platform thread.

> **Note:** `join()` throws the checked exception `InterruptedException`, so `main` declares `throws InterruptedException`. This is a `join()` rule, not a virtual-thread rule — it applies to platform threads too.

> **Also available:** `Thread.ofVirtual().name("my-thread").start(...)` if you need to give a virtual thread a custom name — virtual threads have no name by default.

#### Method 2: Virtual Thread Executor (recommended for many tasks)

For running many tasks concurrently, use `Executors.newVirtualThreadPerTaskExecutor()`. Unlike `newFixedThreadPool`, there is no pool size limit — the executor creates a **new virtual thread for every task you submit** and manages everything automatically.

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
} // try-with-resources auto-shuts down the executor AND waits for all tasks to finish
```

All 10 tasks run at the same time, so the whole thing finishes in about a second rather than ten.

> **The `try (...)` is essential.** Virtual threads do **not** keep the JVM alive. With a plain `shutdown()`, `main` reaches its end, the JVM exits, and the tasks die before printing anything — you get no output at all. Closing the executor at the end of the try block automatically **waits** for all tasks to finish first.

> **Print `Thread.currentThread()`, not `.getName()`.** Virtual threads have empty names by default, so `getName()` prints a blank. Printing the thread object shows the full `VirtualThread[#21]/...` label.

---

### Virtual Threads in Spring Boot

In a real Spring Boot application you rarely create threads yourself. The embedded server (Tomcat) keeps its own thread pool and hands **one thread to each incoming HTTP request**, which carries that request down through your controller, service, and repository layers and back. You write ordinary blocking code; the framework does the threading.

Enabling virtual threads for all HTTP request handling requires a single configuration line:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

Spring Boot replaces the default Tomcat thread pool with virtual threads, meaning each incoming request gets its own virtual thread. No code changes, no reactive rewrites — the same blocking-style code you already write now scales to handle far more concurrent requests.

`ExecutorService` and `CompletableFuture` are different: those you **do** write yourself, when a single request needs several slow operations done in parallel (for example calling three external services at once instead of one after another).

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
| Scalable I/O | Virtual threads (Java 21) | High-concurrency I/O-bound workloads |

---

END