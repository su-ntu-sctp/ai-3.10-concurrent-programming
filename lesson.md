# Lesson: Java Multithreading

## Lesson Overview

This lesson introduces multithreading in Java, from basic thread creation to modern concurrency patterns. You'll learn to create and manage threads, handle synchronization issues, use thread pools with ExecutorService, work with CompletableFuture for asynchronous programming, and explore Java 21's virtual threads.

**Prerequisites:** Basic Java knowledge (classes, interfaces, lambda expressions)

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Create** and manage threads using `Thread`, `Runnable`, and lambda expressions
2. **Identify** race conditions and apply synchronization to prevent them
3. **Use** `ExecutorService` and `CompletableFuture` for efficient concurrent programming
4. **Recognise** when to use virtual threads (Java 21) for scalable applications

---

## Part 1: Basics of Multithreading

Consider 5 philosophers seated around a table, each with a plate of spaghetti. There is a fork between each pair of adjacent diners. To eat, a diner needs both forks on either side of them. If a fork is unavailable, the diner simply waits. This is the famous **Dining Philosophers Problem** — a classic illustration of the challenges in multithreaded programming, where multiple processes compete for shared resources.

For more info: https://medium.com/science-journal/the-dining-philosophers-problem-fded861c37ed

As you can imagine, building an actual multithreaded application is a complex task. It requires a good understanding of Java's concurrency mechanisms and the ability to design algorithms that can run concurrently without causing problems. This lesson is an introduction — a starting point for you to explore further.

### CPUs, Cores, Processes and Threads

A typical system has a single CPU (also known as a processor). Most modern CPUs have multiple cores, which means they can execute multiple tasks simultaneously at the hardware level.

On a Mac, you can check the number of cores by going to `System Settings` > `General` > `About` > `System Report`. In Windows, go to `Task Manager` > `Performance` > `CPU`.

<img src="https://spectrum.ieee.org/media-library/image.jpg?id=25560174&width=980" width="350">

A running application is known as a **process**. A **thread** is a unit of execution within a process. A process can have multiple threads, each carrying out different parts of the work at the same time. Each thread contains the instructions that are executed by the processor.

<img src="https://miro.medium.com/v2/resize:fit:1200/0*OrJ6SWAS2ATYskoI.png" width="300">

### Concurrency vs Parallelism

Concurrency and parallelism are two related but distinct concepts that are often confused.

**Concurrency** is the ability of an application to execute multiple tasks by interleaving their execution. It does not necessarily mean tasks run at exactly the same time — instead, the processor switches between tasks so quickly that it gives the illusion of simultaneous execution. This is known as **time slicing**.

**Parallelism** is the ability of an application to execute multiple tasks truly simultaneously, which requires multiple cores.

<img src="https://www.baeldung.com/wp-content/uploads/sites/4/2022/01/vs-1024x462-1.png" width="450">

Source: https://www.baeldung.com/cs/concurrency-vs-parallelism

In Java, when working with threads, we are actually working with concurrency. Java threads are mapped to operating system threads, which are scheduled by the operating system. This means we do not have direct control over the order of execution — it is managed for us. Parallelism is possible in Java but depends on the number of cores available and the operating system scheduler.

---

## Part 2: Threads

### Thread Lifecycle

A thread goes through several states during its lifetime. The key states to understand are:

| State | Description |
|---|---|
| **New** | A thread has been created but `start()` has not yet been called |
| **Runnable / Running** | After calling `start()`, the thread is ready to run or actively executing |
| **Waiting / Sleeping** | The thread is paused — either waiting for a condition or sleeping for a set time |
| **Terminated** | The thread has finished executing |

Understanding these states helps you reason about what your threads are doing at any point in time.

### Why Use Threads?

Create a `LearnThreads.java` and code along.

First, let's see the problem with single-threaded applications. Add a static method to simulate a long-running task.

```java
public static void simulateLongDelay(int milliseconds, String message) {
    long startTime = System.currentTimeMillis();
    while (System.currentTimeMillis() - startTime < milliseconds) {
        // Do nothing.
    }
    System.out.println(message);
}
```

Next, run the following code in the `main` method.

```java
System.out.println("Application Started");
simulateLongDelay(5000, "Completed Time Intensive Task");
System.out.println("Hello World!");
```

The program will only print `Hello World!` after 5 seconds. This is the problem with single-threaded applications — if one task takes a long time, everything else is blocked until it finishes. Threads solve this by letting you run tasks concurrently so the rest of the application can keep going.

### Creating Threads

There are two main ways to create a thread in Java:

1. By extending the `Thread` class
2. By implementing the `Runnable` interface (preferred)

#### Extending the `Thread` Class

Create a class that extends `Thread` and override the `run` method with the task you want to execute.

```java
class MyFirstThread extends Thread {

  @Override
  public void run() {
    LearnThreads.simulateLongDelay(5000, "Intensive task completed with Thread subclass.");
  }
}
```

In `main`, create an instance and call `start()` to begin the thread.

```java
MyFirstThread myFirstThread = new MyFirstThread();
myFirstThread.start();
System.out.println(Thread.currentThread().getName() + ": Hello World");
```

Notice that `Hello World` prints immediately, without waiting for the 5-second task. The time-intensive task is now running in a separate thread.

To see thread names, use `Thread.currentThread().getName()` inside the `run` method as well:

```java
LearnThreads.simulateLongDelay(5000, Thread.currentThread().getName() + ": Completed Time Intensive Task.");
```

You may have noticed that we call `start()` rather than `run()` directly. The `start()` method is what actually creates a new thread and calls `run()` for us. If you call `run()` directly, no new thread is created — it just runs on the main thread like a regular method call. Try it and observe the difference.

```java
myFirstThread.run(); // No new thread — runs on main thread
```

#### Implementing the `Runnable` Interface

The second and more flexible way to create a thread is by implementing the `Runnable` interface. A `Runnable` represents a task — you pass it to a `Thread` which then executes it.

```java
class MyFirstRunnable implements Runnable {

  @Override
  public void run() {
    LearnThreads.simulateLongDelay(5000,
        Thread.currentThread().getName() + ": Completed Time Intensive Task using Runnable.");
  }
}
```

Pass the `Runnable` to the `Thread` constructor and call `start()`.

```java
Thread runnableThread = new Thread(new MyFirstRunnable());
runnableThread.start();
```

Note that the order of output may differ each time you run the code. This is because threads run concurrently and the operating system decides when to schedule each one — we cannot guarantee execution order.

#### Using a Lambda Expression

Since `Runnable` is a functional interface (it has exactly one abstract method), we can use a lambda expression to create a thread more concisely. This is the most common style you will see in modern Java code.

| Without Lambda | With Lambda |
|---|---|
| `new Thread(new Runnable() {`<br>`  @Override`<br>`  public void run() {`<br>`    System.out.println("Hello");`<br>`  }`<br>`})` | `new Thread(() -> {`<br>`  System.out.println("Hello");`<br>`})` |

```java
Thread lambdaRunnableThread = new Thread(() -> {
  simulateLongDelay(5000,
      Thread.currentThread().getName() + ": Completed Time Intensive Task using Lambda.");
});
lambdaRunnableThread.start();
```

You can also start a thread directly without storing it in a variable.

```java
new Thread(() -> {
  System.out.println("Simple Thread using Lambda Expression.");
}).start();
```

**Why prefer `Runnable` over extending `Thread`?**
- You can still extend other classes if needed (Java only allows single inheritance)
- The same `Runnable` instance can be passed to multiple threads
- Lambda expressions make the code concise and readable
- Many Java APIs and frameworks accept `Runnable` directly

### Sleep and Interrupt

A thread can be paused using `Thread.sleep()`. This is useful when you want to delay execution — for example, simulating a network delay, polling at intervals, or waiting for a resource.

```java
Thread sleepyThread = new Thread(() -> {
    try {
      System.out.println(Thread.currentThread().getName() + ": sleepyThread is going to sleep.");
      Thread.sleep(8000);
      System.out.println(Thread.currentThread().getName() + ": sleepyThread is awake.");
    } catch (InterruptedException e) {
      System.out.println(Thread.currentThread().getName() + ": sleepyThread was interrupted.");
    }
});
sleepyThread.start();
```

`Thread.sleep()` throws an `InterruptedException` if the thread is interrupted while sleeping. To wake a sleeping thread early, call `interrupt()` on it.

```java
sleepyThread.interrupt();
```

### Race Condition and Synchronisation

When multiple threads access and modify the same shared resource, the results can become unpredictable. This is known as a **race condition** — because the threads are literally "racing" to read and write the same data.

Let's see a concrete example. Create a `MultipleThreads.java` and code along.

Add a `BankAccount` class.

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

In `main`, create a bank account and define deposit and withdraw runnables.

```java
BankAccount account = new BankAccount(1000);
System.out.println("Initial Balance: $" + account.getBalance());

Runnable depositRunnable = () -> {
  for (int i = 0; i < 5; i++) {
    account.deposit(100);
  }
};

Runnable withdrawRunnable = () -> {
  for (int i = 0; i < 5; i++) {
    account.withdraw(200);
  }
};
```

One advantage of using `Runnable` is that you can pass the same instance to multiple threads, reusing the same behaviour across different threads without duplicating code.

```java
Thread depositThread1 = new Thread(depositRunnable);
Thread depositThread2 = new Thread(depositRunnable);
Thread withdrawThread1 = new Thread(withdrawRunnable);
Thread withdrawThread2 = new Thread(withdrawRunnable);

depositThread1.start();
depositThread2.start();
withdrawThread1.start();
withdrawThread2.start();
```

Run the code several times and observe the balance. You'll likely see inconsistent results — the final balance may not match what you'd expect mathematically. This is the race condition in action: two threads read the balance at the same time, both apply their change, and one of them overwrites the other's work.

To fix this, we need to ensure that only one thread can access `deposit` or `withdraw` at a time. We do this with the `synchronized` keyword, which locks the method so that while one thread is executing it, all others must wait.

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

Run the code again. The balance should now be consistent and correct every time.

### 🧑‍💻 Activity **(10 minutes)**

Add a `calculateBigNumber()` method to `LearnThreads.java`.

```java
public static void calculateBigNumber() {
  long result = 0;
  for (long i = 0; i < 1000000000; i++) {
    result += i;
  }
  System.out.println("Result: " + result);
}
```

Create two threads to run this method concurrently — one using the `Thread` class and one using the `Runnable` interface with a lambda expression. Observe the output.

---

## Part 3: Multithreading Using ExecutorService

Creating threads manually works fine for simple cases, but it becomes problematic at scale. Each time you create a new `Thread` object, the JVM and operating system have to allocate resources for it. If your application spawns hundreds of threads, this can exhaust system memory and degrade performance.

Java's `ExecutorService` solves this with a **thread pool** — a fixed set of reusable threads that are kept alive and assigned tasks as they come in. Instead of creating and destroying threads repeatedly, the pool recycles them, which is much more efficient.

Create a `LearnExecutors.java` and code along.

Create an `ExecutorService` with a fixed pool of 5 threads.

```java
ExecutorService executorService = Executors.newFixedThreadPool(5);
```

This means at most 5 tasks will run at the same time. If you submit more than 5 tasks, the extras wait in a queue until a thread becomes free. Choosing the right pool size is a trade-off: too few threads means tasks queue up; too many means wasted resources.

Create two `Runnable` tasks.

```java
Runnable printLettersRunnable = () -> {
  System.out.println(Thread.currentThread().getName() + ": This thread would loop through letters A to E");
  String[] letters = { "A", "B", "C", "D", "E" };
  for (String letter : letters) {
    System.out.println(Thread.currentThread().getName() + ": Current letter: " + letter);
    try {
      Thread.sleep(1000);
    } catch (Exception e) {
      System.out.println("Interrupted Thread");
    }
  }
};

Runnable printSquaresRunnable = () -> {
  System.out.println(Thread.currentThread().getName() + ": This thread would print the squares of 1 to 5");
  for (int i = 1; i <= 5; i++) {
    System.out.println(Thread.currentThread().getName() + ": Current number: " + i + ", Squared value: " + (i * i));
    try {
      Thread.sleep(1000);
    } catch (Exception e) {
      System.out.println("Interrupted Thread");
    }
  }
};
```

Submit the tasks using `execute()`. Notice we do not call `start()` — the `ExecutorService` manages that for us.

```java
executorService.execute(printLettersRunnable);
executorService.execute(printSquaresRunnable);
executorService.execute(printLettersRunnable);
```

Always shut down the `ExecutorService` when you are done. Without this, the thread pool threads keep running and the application will not terminate.

```java
executorService.shutdown();
```

Now try changing the pool size to 2 and observe how the output changes.

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
```

With only 2 threads, the third task has to wait until one of the first two finishes — you'll see a clear queuing effect in the output.

---

## Part 4: CompletableFuture

`CompletableFuture` was introduced in Java 8 and represents a more modern, powerful approach to asynchronous programming. It addresses two key limitations of `Runnable` and `Thread`.

**Problem 1 — No return values:** `Runnable` cannot return a result. If you run a calculation in a thread, there is no built-in way to get the result back to the calling code.

```java
// With Runnable - can't easily get results back
Runnable task = () -> {
  long result = calculateBigNumber2();
  // How do you get this result back to the main thread?
};
```

**Problem 2 — Complex exception handling:** Exceptions thrown inside a thread are trapped there. The main thread has no easy way to know something went wrong.

```java
// With threads - exceptions are hard to handle
Thread thread = new Thread(() -> {
  try {
    calculateBigNumber(); // throws exception
  } catch (Exception e) {
    // Exception is trapped here — main thread doesn't know
  }
});
```

`CompletableFuture` solves both problems elegantly.

Create a `LearnCompletableFuture.java` and add the following helper methods.

```java
// Method that does not return a value
public static void calculateBigNumber1() {
  long result = 0;
  for (long i = 0; i < 1_000_000_000; i++) {
    result += i;
  }
  System.out.println(Thread.currentThread().getName() + ": Result: " + result);
}

// Method that returns a value
public static long calculateBigNumber2() {
  long result = 0;
  for (long i = 0; i < 1_000_000_000; i++) {
    result += i;
  }
  return result;
}

// Method that throws an exception
public static long calculateBigNumber3() {
  long result = 0;
  for (long i = 0; i < 1_000_000_000; i++) {
    result += i;
  }
  throw new RuntimeException("Error in calculateBigNumber3");
}
```

#### runAsync() — For Tasks That Do Not Return a Value

Use `CompletableFuture.runAsync()` when your task does not need to return anything. It takes a `Runnable` and runs it asynchronously in a separate thread.

```java
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
  calculateBigNumber1();
});
```

If you run this now, the main thread might finish before the async task completes and you'll see no output. Call `join()` to block the main thread until the future completes.

```java
future1.join();
```

#### supplyAsync() — For Tasks That Return a Value

Use `CompletableFuture.supplyAsync()` when your task returns a result. It takes a `Supplier` (a functional interface that returns a value).

```java
CompletableFuture<Long> future2 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber2();
});
```

To consume the result when it is ready, chain `.thenAccept()` which takes a `Consumer`.

```java
future2.thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ": Result2: " + result);
});
```

You can also chain both calls together without storing intermediate futures.

```java
CompletableFuture<Void> future3 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber2();
}).thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ": calculateBigNumber2 result: " + result);
});
```

#### exceptionally() — Handling Exceptions

When using `CompletableFuture`, you can handle exceptions gracefully using `exceptionally()`. This method lets you define a fallback action if the future completes with an error, rather than crashing silently.

```java
CompletableFuture<Void> future4 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber3();
}).thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ": Result: " + result);
}).exceptionally(ex -> {
  System.out.println(Thread.currentThread().getName() + ": 🚨 Exception occurred: " + ex.getMessage());
  return null;
});
```

To wait for multiple futures to all complete, use `allOf()` followed by `join()`.

```java
CompletableFuture.allOf(future1, future2, future3, future4).join();
```

> **Tip #1:** When your use case is asynchronous programming with results or error handling, prefer `CompletableFuture` over raw `Thread` or `Runnable`.

> **Tip #2:** If you are designing a series of runnable processes, it is fine to implement `Runnable` and then decide how to execute them — via `ExecutorService`, `CompletableFuture`, or other mechanisms.

---

## Part 5: Introduction to Virtual Threads (Java 21)

### What are Virtual Threads?

Virtual threads are a lightweight alternative to traditional platform threads, introduced in Java 21 as part of **Project Loom**. They are designed to handle a massive number of concurrent tasks with minimal resource overhead.

The key problem that virtual threads solve is this: traditional platform threads are expensive. Each one is mapped 1-to-1 to an OS thread, which consumes significant memory (typically ~1MB per thread) and takes time to create and context-switch. This means most applications are limited to a few thousand threads before running into performance issues.

Virtual threads, on the other hand, are managed by the JVM rather than the OS. They are extremely cheap to create — you can run tens of thousands or even millions of them. The JVM multiplexes many virtual threads onto a small number of OS threads automatically.

| Platform Threads | Virtual Threads |
|---|---|
| Heavyweight (managed by OS) | Lightweight (managed by JVM) |
| Limited by system resources (~thousands) | Can create millions |
| 1:1 mapping with OS threads | Many-to-few mapping with OS threads |
| Expensive to create and context-switch | Cheap to create and switch |

**Important note:** Virtual threads are NOT faster for CPU-intensive tasks. They excel at handling many concurrent I/O operations — like network calls, database queries, and file reads — where threads spend most of their time waiting.

Create a `LearnVirtualThreads.java` and code along.

### Creating Virtual Threads

#### Method 1: Using Thread.startVirtualThread()

The simplest way to create and start a virtual thread.

```java
Thread virtualThread = Thread.startVirtualThread(() -> {
  System.out.println("Hello from virtual thread: " + Thread.currentThread());
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

Notice the thread name includes `VirtualThread` in the output — this confirms it is a virtual thread.

#### Method 2: Using Thread.ofVirtual()

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

#### Method 3: Using a Virtual Thread Executor

For running many tasks, use an executor that creates a new virtual thread per task. Unlike `newFixedThreadPool`, there is no pool size limit — the JVM handles everything.

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

for (int i = 0; i < 10; i++) {
  int taskNumber = i;
  executor.submit(() -> {
    System.out.println("Task " + taskNumber + " running on: " + Thread.currentThread());
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Task " + taskNumber + " completed");
  });
}

executor.shutdown();
executor.awaitTermination(5, TimeUnit.SECONDS);
System.out.println("All tasks completed");
```

### Seeing the Scalability Advantage

Let's demonstrate how virtual threads scale. This code creates 10,000 virtual threads — try doing that with a fixed thread pool and you'll likely run out of memory or see severe performance degradation.

```java
public static void demonstrateScalability() {
  System.out.println("\n=== Creating 10,000 Virtual Threads ===");
  long startTime = System.currentTimeMillis();

  try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10000; i++) {
      executor.submit(() -> {
        try {
          Thread.sleep(100);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      });
    }
  } // Auto-shutdown on close

  long endTime = System.currentTimeMillis();
  System.out.println("Time taken: " + (endTime - startTime) + "ms");
  System.out.println("Successfully created and ran 10,000 virtual threads!");
}
```

### When to Use Virtual Threads

Virtual threads are ideal for **I/O-bound** workloads where threads spend most of their time waiting — network calls, database queries, file operations, or handling web requests in a server. They are not recommended for CPU-intensive tasks (like heavy calculations) where threads are always busy — in those cases, a fixed thread pool sized to your CPU cores is a better fit.

Unlike platform threads, you do not need to pool virtual threads. Since they are so cheap to create, you can simply create a new one per task.

### 🧑‍💻 Activity **(5 minutes)**

Go back to your `LearnExecutors.java` from Part 3. Replace `Executors.newFixedThreadPool(5)` with `Executors.newVirtualThreadPerTaskExecutor()` and observe the difference in behavior and thread names in the output.

```java
// Before
ExecutorService executorService = Executors.newFixedThreadPool(5);

// After
ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor();
```

---

END