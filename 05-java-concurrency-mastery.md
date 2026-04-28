# 🧵 Complete Java Concurrency Mastery Guide

This guide breaks down foundational and advanced Java Concurrency concepts, explaining what they are, when to use them in real-world scenarios, and providing concise code examples.

---

## 🟢 BASIC CONCEPTS

### 1. Application, Process, and Thread
* **What it is:** An **Application** is the static code. A **Process** is a running instance of the app with its own isolated memory space. A **Thread** is the smallest unit of execution *within* a process. Threads share the process's memory heap.
* **When to use:** Use processes for isolation (e.g., Microservices). Use threads for concurrent tasks within a single service (e.g., Tomcat creating a thread per HTTP request).

### 2. How to Create a Thread
* **What it is:** Threads are created by implementing `Runnable` or extending the `Thread` class.
* **When to use:** Purely foundational. In modern Java, you should almost *never* manually create threads. Use `ExecutorService` or Virtual Threads instead.
* **Example:**
  ```java
  Thread thread = new Thread(() -> System.out.println("Running"));
  thread.start(); // Always call start(), not run()
  ```

### 3. How to Join a Thread
* **What it is:** The `join()` method forces the current thread to wait until the target thread finishes.
* **When to use:** When your main thread cannot proceed until a background task finishes.
* **Example:**
  ```java
  Thread worker = new Thread(() -> doWork());
  worker.start();
  worker.join(); // Main thread blocks here until worker completes
  ```

### 4. How to Interrupt a Thread
* **What it is:** `interrupt()` sets an interrupt flag on a thread. If the thread is blocked (sleeping, waiting), it throws an `InterruptedException`.
* **When to use:** To politely request a background task (like a file download) to cancel itself.
* **Example:**
  ```java
  Thread t = new Thread(() -> {
      while (!Thread.currentThread().isInterrupted()) { /* process */ }
  });
  t.start();
  t.interrupt(); // Requests the thread to stop
  ```

### 5. Threads Synchronization
* **What it is:** The `synchronized` keyword ensures only one thread can execute a block of code at a time using an intrinsic lock (monitor).
* **When to use:** To protect shared, mutable state and prevent Race Conditions.
* **Example:**
  ```java
  public synchronized void increment() { 
      this.count++; // Thread-safe
  }
  ```

### 6. Happens-before Relationship
* **What it is:** A guarantee by the Java Memory Model. If action A *happens-before* action B, the memory changes made by A are guaranteed to be visible to B.
* **When to use:** Conceptual. It dictates how you write thread-safe code (e.g., unlocking a monitor *happens-before* another thread locks that same monitor).

### 7. Locks Granularity
* **What it is:** **Coarse-grained** locks the whole object (`synchronized(this)`). **Fine-grained** locks only specific parts of the object.
* **When to use:** Use fine-grained locks to increase throughput, allowing non-conflicting threads to work simultaneously.

### 8. Volatile Variables
* **What it is:** The `volatile` keyword ensures reads and writes go straight to main memory, bypassing CPU caches. It guarantees visibility, but **not atomicity** (e.g., `count++` is not safe).
* **When to use:** For state flags where one thread writes and others check.
* **Example:**
  ```java
  private volatile boolean isRunning = true;
  ```

### 9. Atomic Variables
* **What it is:** Classes like `AtomicInteger` that use CPU-level Compare-And-Swap (CAS) instructions for thread-safe operations without locking.
* **When to use:** Counters, ID generators, and metrics aggregators.
* **Example:**
  ```java
  AtomicInteger counter = new AtomicInteger(0);
  counter.incrementAndGet(); // Thread-safe, no synchronized block needed
  ```

### 10. Wait/Notify and the Producer-Consumer Pattern
* **What it is:** Methods used for inter-thread communication. Must be called from inside a `synchronized` block.
* **When to use:** When threads need to coordinate (e.g., Consumer waits if queue is empty, Producer notifies when item added). *Note: Prefer `BlockingQueue` in modern Java.*
* **Example:**
  ```java
  synchronized(lock) {
      while(queue.isEmpty()) lock.wait(); // Release lock and wait
      Object item = queue.poll();
  }
  ```

### 11. ThreadLocal Variables
* **What it is:** Variables that provide thread-local copies. Each thread accessing the variable gets its own independent instance.
* **When to use:** Storing User Context (Security Principals), Transaction IDs, or isolating non-thread-safe objects like `SimpleDateFormat`.
* **Example:**
  ```java
  ThreadLocal<String> userId = new ThreadLocal<>();
  userId.set("user-123");
  ```

### 12. Race Condition
* **What it is:** A bug where two or more threads attempt to change shared data at the exact same time, leading to unpredictable results.
* **When to use (Fix):** Prevent this using `synchronized`, `ReentrantLock`, or `Atomic` classes.

### 13. Deadlock
* **What it is:** Thread A holds Lock 1 and waits for Lock 2. Thread B holds Lock 2 and waits for Lock 1. Both are stuck forever.
* **When to use (Fix):** Always acquire multiple locks in the exact same predefined order across all threads.

### 14. Starvation
* **What it is:** A thread is perpetually denied access to resources because "greedy" or higher-priority threads keep taking the lock.
* **When to use (Fix):** Use "Fair" locks (e.g., `new ReentrantLock(true)`) which give access to the longest-waiting thread.

### 15. Livelock
* **What it is:** Threads constantly change state in response to each other to avoid a deadlock, but make no actual progress (like two people stepping aside for each other in a hallway infinitely).
* **When to use (Fix):** Add random backoff delays before retrying the operation.

---

## 🔴 ADVANCED CONCEPTS

### 1. Lock and ReentrantLock
* **What it is:** An explicit, advanced locking mechanism. `Reentrant` means a thread can acquire the same lock multiple times without deadlocking itself.
* **When to use:** When you need advanced features `synchronized` lacks: `tryLock()` (don't wait forever), interruptible locks, or fairness.
* **Example:**
  ```java
  Lock lock = new ReentrantLock();
  if (lock.tryLock()) {
      try { /* Critical section */ } 
      finally { lock.unlock(); } // MUST be in finally block
  }
  ```

### 2. Read-Write Lock
* **What it is:** Allows multiple threads to read simultaneously, but only one thread can write (blocking all readers).
* **When to use:** Data structures that are read *frequently* but updated *rarely* (e.g., an in-memory cache).
* **Example:**
  ```java
  ReadWriteLock rwLock = new ReentrantReadWriteLock();
  rwLock.readLock().lock(); // Multiple threads can do this
  rwLock.writeLock().lock(); // Exclusive access
  ```

### 3. Condition Variables
* **What it is:** The explicit lock equivalent of `wait()/notify()`. You can have multiple conditions bound to a single Lock.
* **When to use:** Bounded buffers. You can have a `notFull` condition (producers wait on this) and a `notEmpty` condition (consumers wait on this).
* **Example:**
  ```java
  Condition notEmpty = lock.newCondition();
  notEmpty.await();  // Like wait()
  notEmpty.signal(); // Like notify()
  ```

### 4. Semaphore
* **What it is:** Maintains a set of "permits". `acquire()` takes a permit (blocks if 0). `release()` returns a permit.
* **When to use:** Limiting concurrent access to a restricted resource (e.g., restricting DB connections or API rate limiting).
* **Example:**
  ```java
  Semaphore sem = new Semaphore(5); // Only 5 threads allowed
  sem.acquire();
  // ... use resource ...
  sem.release();
  ```

### 5. CyclicBarrier and Parallel Sum
* **What it is:** Forces a set number of threads to wait at a "barrier" until all arrive. Once all arrive, they proceed together. It **can be reused**.
* **When to use:** Parallel divide-and-conquer. E.g., computing sub-sums of an array in 4 threads, waiting at the barrier, and then combining the results.

### 6. CountDownLatch and Merge-Sort Algorithm
* **What it is:** A counter initialized to N. Threads `await()` until other threads `countDown()` the latch to zero. It **cannot be reused**.
* **When to use:** Waiting for N independent services to initialize before starting the main app, or a parent thread waiting for N workers to finish.
* **Example:**
  ```java
  CountDownLatch latch = new CountDownLatch(3);
  latch.countDown(); // Worker finished
  latch.await();     // Main thread waits until counter is 0
  ```

### 7. Exchanger
* **What it is:** A synchronization point where exactly two threads can safely swap objects with each other.
* **When to use:** Genetic algorithms, or strict producer-consumer where one thread fills a buffer and swaps it with an empty buffer from the consumer.

### 8. Phaser
* **What it is:** A highly flexible, reusable barrier. Unlike CyclicBarrier, the number of registered parties can change dynamically during execution.
* **When to use:** Complex, multi-phase parallel tasks where worker threads can arrive, complete phases, or deregister dynamically.

### 9. CopyOnWrite Collections
* **What it is:** E.g., `CopyOnWriteArrayList`. Mutative operations (add/set) create a brand new copy of the underlying array. Iterations read the old copy lock-free.
* **When to use:** Event Listener lists or read-heavy caches where iterations heavily outnumber modifications.

### 10. NonBlocking Queues
* **What it is:** E.g., `ConcurrentLinkedQueue`. Uses lock-free CAS algorithms instead of physical locks.
* **When to use:** Highly concurrent environments where threads should never be blocked or put to sleep, maximizing throughput.

### 11. Blocking_Queues
* **What it is:** E.g., `ArrayBlockingQueue`. `take()` blocks if the queue is empty; `put()` blocks if it is full.
* **When to use:** The absolute standard for the Producer-Consumer pattern and Thread Pool work queues.

### 12. ConcurrentMap
* **What it is:** E.g., `ConcurrentHashMap`. Thread-safe map that uses lock striping (Java 7) or CAS + node-level locking (Java 8+) to allow highly concurrent reads and writes.
* **When to use:** Multi-threaded caching. (Never use legacy `Hashtable` or `Collections.synchronizedMap()`).

### 13. Map-Reduce Algorithm
* **What it is:** Paradigm for processing huge datasets: split data (Map), process in parallel, and aggregate results (Reduce).
* **When to use:** Distributed big-data processing (Hadoop), or locally via Java Parallel Streams.

### 14. Executors
* **What it is:** A framework that decouples task submission from thread management.
* **When to use:** Any time you need asynchronous execution. Never write `new Thread()`; use `Executors`.

### 15. Scheduled Tasks
* **What it is:** `ScheduledExecutorService`. Replaces the legacy `Timer` class.
* **When to use:** Running background cleanup jobs, polling an external API every 5 minutes.
* **Example:**
  ```java
  executor.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES);
  ```

### 16. ThreadPoolExecutor and ThreadsFactory
* **What it is:** The highly configurable engine behind ExecutorService. `ThreadFactory` allows custom naming of threads.
* **When to use:** When standard `Executors.newFixedThreadPool()` isn't enough. Use this to configure custom Core/Max pool sizes, Keep-Alive times, custom Rejection Policies, and meaningful thread names for easier log debugging.

### 17. Fork-Join Pool
* **What it is:** A specialized thread pool for recursive, divide-and-conquer tasks. Uses "work-stealing" (idle threads steal work from the queues of busy threads).
* **When to use:** Recursive algorithms (Merge Sort), and it powers `parallelStream()` under the hood.

### 18. CompletableFuture
* **What it is:** Java's implementation of Promises. Allows chaining non-blocking, asynchronous callbacks.
* **When to use:** Composing multiple async I/O calls. (e.g., fetch user data asynchronously `thenApply` fetch user orders `thenCombine` results).
* **Example:**
  ```java
  CompletableFuture.supplyAsync(() -> fetchUser())
                   .thenAccept(user -> save(user));
  ```

### 19. Parallel Streams
* **What it is:** Calling `collection.parallelStream()`. Distributes operations across the common ForkJoinPool.
* **When to use:** CPU-heavy processing on massive collections. **Warning:** Never use for blocking I/O (like DB calls), as it will freeze the shared global common pool.

### 20. Spinlock and Busy Wait
* **What it is:** A thread loops continuously checking a condition (`while(!ready) {}`) instead of yielding the CPU via `wait()`.
* **When to use:** ONLY in ultra-low latency systems (High-Frequency Trading) where the microsecond cost of an OS context switch is worse than wasting CPU cycles.

### 21. Lockfree and Waitfree Algorithms
* **What it is:** Algorithms built on CAS without locks. **Lock-free** ensures system-wide progress. **Wait-free** ensures every thread completes in finite steps (no starvation).
* **When to use:** Implementing high-performance data structures. (Java does this for you inside `ConcurrentLinkedQueue` and `AtomicInteger`).

### 22. Throughput and Latency in Concurrent Applications
* **What it is:** **Throughput** = tasks completed per second. **Latency** = time to complete one task.
* **When to use:** Trade-off analysis. Heavy locking reduces throughput. Fine-grained locking increases throughput but might increase latency slightly due to lock management overhead.

### 23. Profiling
* **What it is:** Running tools (VisualVM, YourKit, Java Flight Recorder) to see exactly where an application spends its time or memory.
* **When to use:** Finding performance bottlenecks, deadlocks, or tracking down memory leaks.

### 24. Microbenchmarks with JMH
* **What it is:** Java Microbenchmark Harness. A framework to accurately measure code execution time.
* **When to use:** To avoid JVM pitfalls like Just-In-Time (JIT) compilation optimizations, Dead Code Elimination, and Warmup phases when testing if algorithm A is faster than algorithm B.
