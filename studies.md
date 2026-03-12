# Self Studies: Java Multithreading

## Overview

In this lesson, you will be introduced to multithreading in Java — one of the more conceptually challenging topics in the course. The self-study materials below are designed to help you build a mental model of how threads work before the lesson, so that the code-along sessions feel familiar rather than overwhelming.

**Estimated Prep Time:** 60–80 minutes

---

## Task 1: Understand Threads and Concurrency

Before writing any multithreaded code, it helps to understand what a thread actually is and why concurrency matters. This video gives you a solid visual introduction to processes, threads, and how they interact — concepts you will apply directly in the lesson.

**Watch:** Introduction to Multithreading and Concurrency in Java
🎬 VIDEO URL NEEDED https://www.youtube.com/watch?v=r_MbozD32eo


**Then read:** Lesson 3.10 — Part 1 (CPUs, Cores, Processes and Threads) and Part 2 up to "Creating Threads"

**Guiding Questions:**
- What is the difference between a process and a thread?
- What is the difference between concurrency and parallelism?
- Why does the order of thread execution vary each time you run the program?

---

## Task 2: CompletableFuture and Asynchronous Programming

`CompletableFuture` is the most modern and widely used approach covered in this lesson. It builds on lambdas (which you already know) and introduces the idea of chaining asynchronous operations. Watching a short video on this before the lesson will make the code-along much easier to follow.

**Watch:** Java CompletableFuture Explained
🎬 VIDEO URL NEEDED
https://www.youtube.com/watch?v=GJ5Tx43q6KM


**Then read:** Lesson 3.10 — Part 4 (CompletableFuture)

**Guiding Questions:**
- What problem does `CompletableFuture` solve that `Runnable` cannot?
- What is the difference between `runAsync()` and `supplyAsync()`?
- How does `exceptionally()` help with error handling in async code?

---

## Task 3: Read — Virtual Threads (Java 21)

Virtual threads are a relatively new feature and there are not yet many beginner-friendly video tutorials available. Reading the lesson section directly is the best preparation here.

**Read:** Lesson 3.10 — Part 5 (Introduction to Virtual Threads)

**Guiding Questions:**
- What is the key difference between a platform thread and a virtual thread?
- Why are virtual threads better suited for I/O-bound tasks than CPU-bound tasks?
- Why do you not need to pool virtual threads the way you pool platform threads?

---

## Active Engagement Strategies

- As you watch the videos, pause and try to sketch out a simple diagram of what is happening — for example, two threads interleaving on a single CPU
- After reading Part 2, try writing a thread using all three approaches (Thread class, Runnable, lambda) from memory before the class
- If something is unclear, note down your question and bring it to the lesson — multithreading has many "why does this happen?" moments that are best discussed live

---

## Additional Reading Material

- [Java Concurrency — Oracle Docs](https://docs.oracle.com/javase/tutorial/essential/concurrency/)
- [Virtual Threads in Java 21 — Baeldung](https://www.baeldung.com/java-virtual-thread-vs-thread)
- [CompletableFuture Guide — Baeldung](https://www.baeldung.com/java-completablefuture)