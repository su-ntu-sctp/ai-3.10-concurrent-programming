# Java Multithreading - Complete Lesson

## Lesson Overview

This lesson introduces multithreading in Java, from basic thread creation to modern concurrency patterns. You'll learn to create and manage threads, handle synchronization issues, use thread pools with ExecutorService, work with CompletableFuture for asynchronous programming, and explore Java 21's virtual threads.

**Prerequisites:** Basic Java knowledge (classes, interfaces, lambda expressions)

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Understand** the difference between processes, threads, concurrency, and parallelism
2. **Create** threads using both the Thread class and Runnable interface
3. **Identify** and prevent race conditions using synchronization
4. **Implement** thread pooling using ExecutorService for better resource management
5. **Apply** CompletableFuture for asynchronous programming with return values and exception handling
6. **Recognize** when to use virtual threads for scalable concurrent applications

---

## Part 1: Basics of Multithreading

Consider 5 philosophers seated around a table, each with a plate of spaghetti. There is a fork between each pair of adjacent diners, and a bowl of spaghetti in the center of the table. The diners do nothing but alternate between thinking and eating. To eat, a diner needs both forks on either side of him. If a fork is unavailable, the diner simply waits. When a diner has both forks, he eats for a while, then puts down the forks and continues to think.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*SdI7qwsNa05IkyiMXlAmPg.png" width="200">

This is the famous dining philosophers problem in computer science. It is a classic multithreading problem. The problem is to design a solution where the philosophers can eat without causing deadlock.

For more info on the Dining Philosophers problem : https://medium.com/science-journal/the-dining-philosophers-problem-fded861c37ed

As you can imagine, building an actual multithreaded application is a very complex task. It requires a deep understanding of the Java language and its concurrency mechanisms, as well as the ability to design algorithms that can be executed concurrently.

This is an introductory lesson to multithreading which is meant to be a starting point for learners to explore further on their own.

<video width="200" loop autoplay muted>
    <source src="https://preview.redd.it/t50uybvfyj771.gif?format=mp4&s=205be9c462550432ea8e84049e0fbf326510c0ee" type="video/mp4">
</video>

### CPUs, Cores, Processes and Threads

A typical system has a single CPU, also known as a processor. Most modern CPUs have multiple cores.

On a Mac, you can check the number of cores by going to `System Settings` > `General` > `About` > `System Report`.

In Windows, you can check the number of cores by going to `Task Manager` > `Performance` > `CPU`.

<img src="https://spectrum.ieee.org/media-library/image.jpg?id=25560174&width=980" width="350">

A running application is known as a process. A thread is a unit of execution within a process. A process can have multiple threads. Each thread contains the instructions that are executed by the processor.

<img src="https://miro.medium.com/v2/resize:fit:1200/0*OrJ6SWAS2ATYskoI.png" width="300">

### Concurrency vs Parallelism

Concurrency and parallelism are two related but distinct concepts. Concurrency is the ability of an application to execute multiple tasks by interleaving their executions. Parallelism is the ability of an application to execute multiple tasks simultaneously.

<img src="https://www.baeldung.com/wp-content/uploads/sites/4/2022/01/vs-1024x462-1.png" width="450">

Source: https://www.baeldung.com/cs/concurrency-vs-parallelism

With concurrency, our application is doing more than one thing at a time, though not necessarily simultaneously. For example, a single processor can switch between tasks to give the illusion of parallelism. This is known as time slicing.

In Java, when working with threads, we are actually working with concurrency. This is because Java threads are mapped to operating system threads, which are scheduled by the operating system.

We do not actually have direct control over the execution of the threads - it is all dependent on the operating system. Because of this, we cannot guarantee the order of execution of the threads.

This also implies that parallelism is not guaranteed in Java. Java enables concurrency, but whether the threads are executed in parallel depends on the operating system and the number of cores available.

---

## Part 2: Threads

### Life Cycle of a Thread


| STATE         | REASON                                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| New           | A thread is created but `start()` has not been called                                                                    |
| Runnable      | After calling the `start()` method, thread is entering a running state                                                   |
| Running       | Thread is actively execute codes                                                                                         |
| Blocked       | The state waiting for a specific resource to be unblocked                                                                |
| Waiting       | When the thread calls `wait()`, `join()`, or `sleep()`. Used when waiting for a specific condition before resume running |
| Timed Waiting | Similar to `Waiting` state except it is timed                                                                            |
| Terminated    | When a thread completed an execution                                                                                     |

### Why use Threads?

Create a `LearnThreads.java` and code along.

First, let's see the problem with single-threaded applications.

Add a static method to simulate a long delay in our application.

```java
public static void simulateLongDelay(int milliseconds, String message) {
    // Get the current time in milliseconds.
    long startTime = System.currentTimeMillis();

    // Loop until the specified number of milliseconds have passed.
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

The `main` method will only print `Hello World!` after 5 seconds. This is the problem with single-threaded applications. If a task takes a long time to complete, the entire application will be blocked until the task is completed.

### Creating Threads

We can solve the previous problem by creating a new thread to run the time-intensive task. This will allow the main thread to continue executing other tasks while the time-intensive task is running.

There are two main ways to create a thread in Java:

1. By extending the `Thread` class
2. By implementing the `Runnable` interface (preferred)

#### Extending the `Thread` class

Let's create a class that extends the `Thread` class.

The `Thread` class has a `run` method that we can override. The `run` method contains the code that will be executed by the thread.

In this case, let's put the time intensve task here.

```java
class MyFirstThread extends Thread {

  @Override
  public void run() {
    LearnThreads.simulateLongDelay(5000, "Intensive task completed with Thread subclass.");
  }

}
```

And in the `main` method, we can create an instance of the `MyFirstThread` class and call the `start` method to start the thread.

```java
public class LearnThreads {

  public static void main(String[] args) {

    MyFirstThread myFirstThread = new MyFirstThread();
    myFirstThread.start();
  }

}
```

Now, the time intensive task will run in a separate thread and the main thread will continue executing the rest of the code.

To see the thread names, we can use the `Thread.currentThread().getName()` method.

```java
System.out.println(Thread.currentThread().getName() + ": Hello World");
```

And in the `run` method, we can print the thread name as well.

```java
LearnThreads.simulateLongDelay(5000, Thread.currentThread().getName() + ": Completed Time Intensive Task.");
```

Run the code again to see the output and observe the thread names.

You may have noticed that we are calling the `start` method instead of the `run` method, even though we overrode the `run` method.

The `start` method actually starts the thread and calls the `run` method for us. If we call the `run` method directly, the thread will not be started.

Try calling the `run` method directly and see what happens.

```java
myFirstThread.run();
```

#### Implementing the `Runnable` interface

The second way to create a thread is by implementing the `Runnable` interface.

```java
class MyFirstRunnable implements Runnable {

  @Override
  public void run() {
    LearnThreads.simulateLongDelay(5000,
        Thread.currentThread().getName() + ": Completed Time Intensive Task using Runnable.");
  }
}
```

We can pass a `Runnable` instance to the `Thread` constructor and call the `start` method to start the thread.

```java
Thread runnableThread = new Thread(new MyFirstRunnable());
runnableThread.start();
```

Note that the order of the output may be different each time you run the code. This is because the threads are running concurrently and the order of execution is not guaranteed. This is scheduled by the operating system.

### 🧑‍💻 Activity

Create another method in the `LearnThreads` class that calculates a big number.

```java
  public static void calculateBigNumber() {
    long result = 0;
    for (long i = 0; i < 1000000000; i++) {
      result += i;
    }
    System.out.println("Result: " + result);
  }
```

Create threads using both the `Thread` class and the `Runnable` interface to run the `calculateBigNumber` method.

### Using an Anonymous Class

Instead of creating a separate class, we can also use an anonymous class.

```java
Thread anonymousRunnableThread = new Thread(new Runnable() {

  @Override
  public void run() {
    LearnThreads.simulateLongDelay(5000,
        Thread.currentThread().getName() + ": Completed Time Intensive Task using Anonymous Class.");
  }
});
anonymousRunnableThread.start();
```

This syntax means we are implementing the `Runnable` interface and overriding the `run` method at the same time, and then passing an instance of the anonymous class to the `Thread` constructor.

### Using a Lambda Expression

From Java 8 onwards, we can also use a lambda expression instead of an anonymous class, which is a concise way to create a thread.

The `Thread` constructor takes in a `Runnable` object. The `Runnable` interface is what we call a functional interface (an interface with only 1 abstract method).
We can use a lambda expression to create an instance of the `Runnable` interface.

| Without Lambda Expression                                                                                                                       | With Lambda Expression                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `Runnable runnable = new Runnable() {`<br>`  @Override`<br>`  public void run() {`<br>`    System.out.println("Hello World");`<br>`  }`<br>`};` | `Runnable runnable = () -> {`<br>`  System.out.println("Hello World");`<br>`};` |

Since we pass a `Runnable` object to the `Thread` constructor, we can use a lambda expression to create the `Runnable` object directly.

| Without Lambda Expression               | With Lambda Expression                                                                  |
| --------------------------------------- | --------------------------------------------------------------------------------------- |
| `Thread thread = new Thread(runnable);` | `Thread thread = new Thread(() -> {`<br>`  System.out.println("Hello World");`<br>`});` |

Let's use a lambda expression to create a thread.

```java
Thread lambdaRunnableThread = new Thread(() -> {
  simulateLongDelay(5000,
      Thread.currentThread().getName() + ": Completed Time Intensive Task using Lambda Expression.");
});
lambdaRunnableThread.start();
```

Note that you can also start a thread directly without assigning it to a variable.

```java
new Thread(() -> {
  System.out.println("Simple Thread using Lambda Expression.");
}).start();
```

Generally, when creating threads, the `Runnable` method is preferred over extending the `Thread` class because:

- we can extend other classes if needed (instead of the `Thread` class)
- we can implement multiple interfaces (instead of just the `Runnable` interface)
- the same instance of the `Runnable` class can be passed to multiple threads
- it allows us to use lambda expressions, which is a more concise syntax
- many Java methods accept `Runnable` instances

### Sleep and Interrupt

A thread can be put to sleep for a specified number of milliseconds using the `sleep` method. This is useful when we want to delay the execution of a thread. For example, we can use the `sleep` method to simulate a delay in a game or wait for a database connection to be established.

```java
Thread sleepyThread = new Thread(() -> {
    try {
      System.out.println( Thread.currentThread().getName() + ": sleepyThread is going to sleep.");
    Thread.sleep(8000);
    System.out.println( Thread.currentThread().getName() + ": sleepyThread is awake.");
    } catch (InterruptedException e) {
    System.out.println(Thread.currentThread().getName() + ": sleepyThread was interrupted.");
    }
});
sleepyThread.start();
```

Run the code and observe the output. The thread will sleep for 8 seconds before printing the message.

The `sleep` method throws an `InterruptedException` if the thread is interrupted while sleeping. This is useful when we want wake up a thread from sleep.

```java
sleepyThread.interrupt();
```

### Race Condition and Synchronisation

When multiple threads access the same resource, it could lead to unpredictable results.

Let's see a simple example. Create a `MultipleThreads.java` and code along.

We will add a simple `BankAccount` class for this example.

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
    System.out
        .println("🟢 Deposited: $" + amount + ", Current Balance: $" + balance);
  }

  public void withdraw(double amount) {
    balance -= amount;
    System.out
        .println("🔴 Withdrawn: $" + amount + ", Current Balance: $" + balance);
  }
}
```

Then in our `main`.

```java
// Instantiate a new bank account with $1000.
BankAccount account = new BankAccount(1000);
System.out.println("Initial Balance: $" + account.getBalance());
```

Let's define 2 Runnables to deposit and withdraw money from the bank account. We can use the lambda expression syntax to create the Runnables.

```java
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

And instantiate 4 threads to run the Runnables. As mentioned earlier, the advantage of using the `Runnable` interface is that we can pass the same instance to multiple threads. With this, we can reuse the behaviour of the `depositRunnable` and `withdrawRunnable` in multiple threads.

```java
Thread depositThread1 = new Thread(depositRunnable);
Thread depositThread2 = new Thread(depositRunnable);
Thread withdrawThread1 = new Thread(withdrawRunnable);
Thread withdrawThread2 = new Thread(withdrawRunnable);

```

And start the threads.

```java
depositThread1.start();
depositThread2.start();
withdrawThread1.start();
withdrawThread2.start();
```

Now, run the code and observe the balances as money is deposited and withdrawn from the bank account.

Does the balance match the expected value?

Due to the threads interleaving, it creates unpredictable results because multiple threads can access and modify the account balance concurrently. This is known as a race condition.

To fix this problem, we need to ensure that only one thread can access the `deposit` or `withdraw` at any one time. This prevents concurrent access to the `balance` variable. This is known as synchronisation and we can achieve this with the `synchronized` keyword.

```java
public synchronized void deposit(double amount) {}
public synchronized void withdraw(double amount) {}
```

Now run the code again. The balance should match the expected value.

---

## Part 3: Multithreading using ExecutorService

Java provides the `ExecutorService` class to manage threads. This is a higher level abstraction that allows us to create and manage threads more easily.

Why use `ExecutorService`?

- Thread reuse and resource management - it manages a pool of threads and reuses them
- Thread pooling and load balancing - we can define the size of the thread pool and it will automatically manage the number of active threads based on available resources. This prevents the application from spawning too many threads and crashing the system.

In simple terms, instead of having to pass our Runnables to the Thread constructor, we can pass them to the `ExecutorService` instead. And the `ExecutorService` will manage the threads for us.

Create a `LearnExecutors.java` in the `lesson_sample_code` folder and code along.

First we have to create an `ExecutorService` instance. We pass in the number of threads we want in the thread pool. In this case, we want 5 threads. This means that the `ExecutorService` will manage a pool of 5 threads for us.

Note that selecting the number of threads is a tradeoff between performance and resource usage. If we have too many threads, it will consume more resources. If we have too few threads, it will affect performance. Because if there are too few threads, some threads will have to wait for other threads to finish before they can start.

```java
ExecutorService executorService = Executors.newFixedThreadPool(5);
```

Create two `Runnable`s.

```java
Runnable printLettersRunnable = () -> {
  System.out.println("This thread would loop through letters A to E");
  String[] letters = { "A", "B", "C", "D", "E" };
  for (String letter : letters) {
    System.out.println("Current letter: " + letter);
    try {
      Thread.sleep(1000);
    } catch (Exception e) {
      System.out.println("Interrupted Thread");
    }
  }
};

Runnable printSquaresRunnable = () -> {
  System.out.println("This thread would print the squares of 1 to 5");
  for (int i = 1; i <= 5; i++) {
    System.out.println("Current number: " + i + ", Squared value: " + (i * i));
    try {
      Thread.sleep(1000);
    } catch (Exception e) {
      System.out.println("Interrupted Thread");
    }
  }
};
```

Then we can pass our Runnables to the `ExecutorService`. Instead of calling the `start` method manually, we call the `execute` method.

```java
executorService.execute(printLettersRunnable);
executorService.execute(printSquaresRunnable);
executorService.execute(printLettersRunnable);
```

Finally, we have to shut down the `ExecutorService` when we are done using it.

```java
executorService.shutdown();
```

Add the thread names using `Thread.currentThread().getName()` to see the output again.

Now, try changing the pool size to 2 and observe the output.

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
```

---

## Part 4: CompletableFuture instead of Runnable interface

`CompletableFuture` is a powerful class introduced in Java 8 that represents a future result of an asynchronous computation. It is a modern and more powerful approach to handling asynchronous programming and multithreading in Java, compared to using `Thread` or `Runnable` directly.

Create a file `LearnCompletableFuture.java` and code along.

### Problem with Runnable/Thread Approach

1. No return values - `Runnable` and `Thread` do not return a value, making it difficult to handle results from asynchronous tasks.

```java
// With Runnable - can't easily get results back
Runnable task = () -> {
  long result = calculateBigNumber();
  // How do you get this result back to the main thread?
};
Thread thread = new Thread(task);
thread.start();
// No easy way to get the result!
```

2. Complex Exception Handling - Handling exceptions in `Runnable` or `Thread` can be cumbersome, as you need to catch exceptions within the `run` method.

```java
// With threads - exceptions are hard to handle
Thread thread = new Thread(() -> {
  try {
      calculateBigNumber(); // throws exception
  } catch (Exception e) {
      // Exception is trapped here, main thread doesn't know
  }
});
thread.start();
// How do you handle exceptions in the main thread?
```

### Using CompletableFuture

First add these static methods to `LearnCompletableFuture.java`:

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

public static void simulateLongDelay(int milliseconds, String message) {
  long startTime = System.currentTimeMillis();

  while (System.currentTimeMillis() - startTime < milliseconds) {
    // Do nothing.
  }
  System.out.println(message);
}
```

#### runAsync() for Methods that Do Not Return a Value

For methods that do no return a value, we can use `CompletableFuture.runAsync()`. `runAync()` is a static method that runs a task asynchronously in a separate thread. It takes a `Runnable` as an argument, which is the task to be executed.

```java
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
  calculateBigNumber1();
});
```

This method runs the task asynchronously and returns a `CompletableFuture<Void>`, which means it does not return any value.

If you run the code now, there will be no output because the main thread will finish before the `CompletableFuture` completes. To wait for the `CompletableFuture` to complete, we can use the `join()` method. The `join()` method blocks the current thread until the `CompletableFuture` completes.

```java
future1.join();
```

#### supplyAsync() for Methods that Return a Value

For methods that return a value, we can use `CompletableFuture.supplyAsync()`. This method takes a `Supplier` as an argument, which is a functional interface that returns a value. Since it is a functional interface, we can use a lambda expression to create an instance of the `Supplier`, just like we did with `Runnable`.

```java
CompletableFuture<Long> future2 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber2();
});
```

This method runs the task asynchronously and returns a `CompletableFuture<Long>`, which means it returns a value of type `Long`. To consume the result, we can use the `thenAccept()` method, which takes a `Consumer` as an argument. The `Consumer` is a functional interface that accepts a value and does not return anything. Again, we can use a lambda expression to create an instance of the `Consumer`, just like we did with `Runnable`.

```java
future2.thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ": Result2: " + result);
});
```

To wait for more than one `CompletableFuture` to complete, we can use the `allOf()` method, and then call `join()`.

```java
CompletableFuture.allOf(future1, future2).join();
```

Alternatively, we can also chain the `thenAccept()` method to the `CompletableFuture` returned by `supplyAsync()`.

```java
CompletableFuture<Void> future3 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber2();
}).thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ": calculateBigNumber2 result: " + result);
});
```

This will run the `calculateBigNumber2` method asynchronously and print the result when it completes.

#### Exception Handling with exceptionally()

When using `CompletableFuture`, we can handle exceptions more gracefully using the `exceptionally()` method. This method allows us to specify a fallback action if the `CompletableFuture` completes exceptionally.

```java
CompletableFuture<Void> future4 = CompletableFuture.supplyAsync(() -> {
  return calculateBigNumber3();
}).thenAccept(result -> {
  System.out.println(Thread.currentThread().getName() + ":Result: " + result);
}).exceptionally(ex -> {
  System.out.println(Thread.currentThread().getName() + ": 🚨 Exception occurred: " + ex.getMessage());
  return null; // Return a fallback result or null for void futures
});
```

This will catch any exceptions thrown by the `calculateBigNumber3` method and print an error message.

Join all the futures to wait for their completion.

```java
CompletableFuture.allOf(future1, future2, future3, future4).join();
```

> Tip #1: When your use case is asynchronous programming, use `CompletableFuture` over `Runnable` interface

> Tip #2: If you are designing a series of runnable processes, it is acceptable to implement `Runnable` interface and then decide how to execute them (using `ExecutorService`, `CompletableFuture`, or others)

---

## Part 5: Introduction to Virtual Threads (Java 21)

### What are Virtual Threads?

Virtual threads are a lightweight alternative to traditional platform threads introduced in Java 21 as part of Project Loom. They are designed to handle massive numbers of concurrent tasks with minimal resource overhead.

**Key Differences:**

| Platform Threads                          | Virtual Threads                              |
| ----------------------------------------- | -------------------------------------------- |
| Heavyweight (managed by OS)               | Lightweight (managed by JVM)                 |
| Limited by system resources (~thousands)  | Can create millions of threads               |
| 1:1 mapping with OS threads               | Many-to-few mapping with OS threads          |
| Expensive to create and context switch    | Cheap to create and switch                   |

### Why Use Virtual Threads?

Virtual threads are ideal for:
- I/O-bound applications (network calls, database queries, file operations)
- Applications that need to handle many concurrent requests (web servers, microservices)
- Simplifying concurrent code without callbacks or reactive programming

**Note:** Virtual threads are NOT faster for CPU-intensive tasks. They excel at handling many concurrent I/O operations.

### Creating Virtual Threads

Create a file `LearnVirtualThreads.java` and code along.

#### Method 1: Using Thread.startVirtualThread()

The simplest way to create and start a virtual thread:

```java
public class LearnVirtualThreads {

  public static void main(String[] args) throws InterruptedException {

    // Create and start a virtual thread
    Thread virtualThread = Thread.startVirtualThread(() -> {
      System.out.println("Hello from virtual thread: " + Thread.currentThread());
      try {
        Thread.sleep(1000);
        System.out.println("Virtual thread woke up!");
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    });

    // Wait for the virtual thread to complete
    virtualThread.join();
    System.out.println("Main thread finished");
  }
}
```

Notice the thread name includes "VirtualThread" in the output!

#### Method 2: Using Thread.ofVirtual()

For more control over virtual thread creation:

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

#### Method 3: Using Virtual Thread Executor

For handling multiple tasks, you can use an executor that creates virtual threads:

```java
// Create an executor that uses virtual threads
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// Submit multiple tasks
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

// Shutdown the executor
executor.shutdown();
executor.awaitTermination(5, TimeUnit.SECONDS);
System.out.println("All tasks completed");
```

### Comparing Platform Threads vs Virtual Threads

Let's see the difference in creating many threads:

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

Try creating 10,000 platform threads using a regular thread pool - you'll likely run into memory issues or poor performance!

### When to Use Virtual Threads

✅ **Good Use Cases:**
- Web applications handling many requests
- Database queries with connection pooling
- Microservices making multiple API calls
- File I/O operations
- Network operations

❌ **Not Recommended For:**
- CPU-intensive calculations
- Short-lived tasks (overhead not worth it)
- When you need guaranteed thread-local storage

### Important Notes

1. **Don't Pool Virtual Threads**: Unlike platform threads, you don't need to pool virtual threads. Create them on-demand as they're cheap to create.

2. **Blocking is OK**: With virtual threads, blocking operations (like `Thread.sleep()` or I/O) don't waste OS threads. The JVM efficiently manages them.

3. **Backward Compatible**: Virtual threads implement the same `Thread` API, so existing code works with minimal changes.

4. **Avoid Synchronized**: While virtual threads work with `synchronized`, it can pin them to platform threads. Prefer `ReentrantLock` or other `java.util.concurrent` primitives for better performance.

### 🧑‍💻 Activity

Try modifying the `LearnExecutors.java` code from Part 3 to use virtual threads instead of a fixed thread pool. Compare the behavior!

**Hint:** Replace `Executors.newFixedThreadPool(5)` with `Executors.newVirtualThreadPerTaskExecutor()`


---

END