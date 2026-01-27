# Java Concurrency Interview Guide - Complete SDE-2 Preparation

> **Target Audience:** Amateur to Intermediate Java Developers preparing for Backend SDE-2 interviews.
>
> This guide covers concurrency concepts from fundamentals to advanced topics with simple examples, visual diagrams, and common interview problems.

---

## Table of Contents

1. [Fundamentals: Concurrency vs Parallelism](#1-fundamentals-concurrency-vs-parallelism)
2. [Threads Basics](#2-threads-basics)
3. [Thread Lifecycle & States](#3-thread-lifecycle--states)
4. [Race Conditions & Thread Safety](#4-race-conditions--thread-safety)
5. [synchronized Keyword](#5-synchronized-keyword)
6. [volatile Keyword](#6-volatile-keyword)
7. [wait(), notify(), notifyAll()](#7-wait-notify-notifyall)
8. [Locks & ReentrantLock](#8-locks--reentrantlock)
9. [ReadWriteLock](#9-readwritelock)
10. [Atomic Variables](#10-atomic-variables)
11. [Thread Pools & ExecutorService](#11-thread-pools--executorservice)
12. [Future & Callable](#12-future--callable)
13. [CompletableFuture](#13-completablefuture)
14. [BlockingQueue](#14-blockingqueue)
15. [Concurrent Collections](#15-concurrent-collections)
16. [CountDownLatch & CyclicBarrier](#16-countdownlatch--cyclicbarrier)
17. [Semaphore](#17-semaphore)
18. [Deadlock, Livelock & Starvation](#18-deadlock-livelock--starvation)
19. [Classic Concurrency Problems](#19-classic-concurrency-problems)
20. [Interview Coding Problems](#20-interview-coding-problems)
21. [Best Practices & Common Pitfalls](#21-best-practices--common-pitfalls)
22. [Quick Revision Cheat Sheet](#22-quick-revision-cheat-sheet)

---

## 1. Fundamentals: Concurrency vs Parallelism

### Q: What is the difference between Concurrency and Parallelism?

| Aspect | Concurrency | Parallelism |
|--------|-------------|-------------|
| **Definition** | Managing multiple tasks by switching between them | Executing multiple tasks simultaneously |
| **CPU Cores** | Can work on single core | Requires multiple cores |
| **Goal** | Deal with many things at once | Do many things at once |
| **Example** | A chef switching between multiple dishes | Multiple chefs cooking different dishes |

```
CONCURRENCY (Single Core - Time Slicing):
┌──────────────────────────────────────────────────────┐
│  Task A  │  Task B  │  Task A  │  Task B  │  Task A  │
└──────────────────────────────────────────────────────┘
                    TIME ──────►

PARALLELISM (Multiple Cores):
Core 1: ┌────────────────────────────────┐
        │           Task A               │
        └────────────────────────────────┘
Core 2: ┌────────────────────────────────┐
        │           Task B               │
        └────────────────────────────────┘
                    TIME ──────►
```

### Q: Why do we need Concurrency in Java?

1. **Better Resource Utilization** - Keep CPU busy while waiting for I/O
2. **Responsiveness** - UI remains responsive while background tasks run
3. **Throughput** - Handle multiple requests simultaneously (web servers)
4. **Performance** - Utilize multi-core processors effectively

---

## 2. Threads Basics

### Q: What is a Thread?

A thread is the smallest unit of execution within a process. A process can have multiple threads sharing the same memory space.

```
┌─────────────────────────────────────────────────────┐
│                      PROCESS                         │
│  ┌─────────────────────────────────────────────┐    │
│  │              SHARED MEMORY                   │    │
│  │    (Heap, Static Variables, Code)           │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│  │ Thread 1│   │ Thread 2│   │ Thread 3│           │
│  │ (Stack) │   │ (Stack) │   │ (Stack) │           │
│  └─────────┘   └─────────┘   └─────────┘           │
└─────────────────────────────────────────────────────┘
```

### Q: How to create a Thread in Java? (Two Ways)

#### Method 1: Extending Thread class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}

// Usage
MyThread thread = new MyThread();
thread.start();  // NOT thread.run()!
```

#### Method 2: Implementing Runnable interface (PREFERRED)

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}

// Usage
Thread thread = new Thread(new MyRunnable());
thread.start();

// Or using Lambda (Java 8+)
Thread thread2 = new Thread(() -> {
    System.out.println("Lambda thread running!");
});
thread2.start();
```

### Q: Why is Runnable preferred over extending Thread?

| Extending Thread | Implementing Runnable |
|-----------------|----------------------|
| Cannot extend another class (Java single inheritance) | Can extend another class |
| Creates tight coupling | Loose coupling, better design |
| Each object creates new thread | Same Runnable can be shared by multiple threads |
| Not reusable | More reusable |

### Q: What is the difference between `start()` and `run()`?

```java
Thread t = new Thread(() -> System.out.println("In thread: " + Thread.currentThread().getName()));

t.run();   // Executes in MAIN thread (no new thread created!)
t.start(); // Creates NEW thread and executes run() in that thread
```

```
t.run():                           t.start():
┌────────────────────┐            ┌────────────────────┐
│     Main Thread    │            │     Main Thread    │
│         │          │            │         │          │
│    ┌────▼────┐     │            │    ┌────▼────┐     │
│    │  run()  │     │            │    │ start() │     │
│    └────┬────┘     │            │    └────┬────┘     │
│         │          │            │         │          │
│    (continues)     │            │   Creates new      │──► ┌─────────────┐
│                    │            │      thread        │    │ New Thread  │
└────────────────────┘            └────────────────────┘    │   run()     │
                                                            └─────────────┘
```

---

## 3. Thread Lifecycle & States

### Q: What are the different states of a Thread?

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    THREAD LIFECYCLE                      │
                    └─────────────────────────────────────────────────────────┘

    ┌─────────┐     start()      ┌──────────┐
    │   NEW   │ ───────────────► │ RUNNABLE │◄──────────────────────────────┐
    └─────────┘                  └────┬─────┘                               │
         │                            │                                      │
         │                  ┌─────────┴─────────┐                           │
    (Thread                 │                   │                           │
     created)        Scheduler              Scheduler                       │
                     picks it              preempts it                      │
                         │                     │                            │
                    ┌────▼────┐          ┌─────▼──────┐                     │
                    │ RUNNING │◄────────►│   READY    │                     │
                    └────┬────┘          └────────────┘                     │
                         │                                                   │
         ┌───────────────┼───────────────────────────────┐                  │
         │               │                               │                  │
         ▼               ▼                               ▼                  │
   ┌───────────┐  ┌─────────────┐              ┌──────────────┐            │
   │  BLOCKED  │  │   WAITING   │              │ TIMED_WAITING│            │
   │           │  │             │              │              │            │
   │(waiting   │  │(wait(),     │              │(sleep(ms),   │            │
   │for lock)  │  │join())      │              │wait(ms))     │            │
   └─────┬─────┘  └──────┬──────┘              └──────┬───────┘            │
         │               │                            │                     │
         │         notify()/                    timeout/                    │
         │         notifyAll()                  interrupt                   │
         │               │                            │                     │
         └───────────────┴────────────────────────────┴─────────────────────┘
                                        │
                                        ▼ (run() completes)
                                 ┌──────────────┐
                                 │  TERMINATED  │
                                 └──────────────┘
```

### Q: Explain each Thread state

| State | Description | How to reach |
|-------|-------------|--------------|
| **NEW** | Thread created but not started | `new Thread()` |
| **RUNNABLE** | Ready to run or running | `thread.start()` |
| **BLOCKED** | Waiting to acquire a lock | Trying to enter `synchronized` block |
| **WAITING** | Waiting indefinitely for another thread | `wait()`, `join()`, `LockSupport.park()` |
| **TIMED_WAITING** | Waiting for specified time | `sleep(ms)`, `wait(ms)`, `join(ms)` |
| **TERMINATED** | Thread finished execution | `run()` method completes |

### Q: Important Thread Methods

```java
// Get current thread reference
Thread currentThread = Thread.currentThread();

// Get/Set thread name
thread.getName();
thread.setName("WorkerThread-1");

// Get thread ID
thread.getId();

// Check if thread is alive
thread.isAlive();

// Sleep - pause current thread (in milliseconds)
Thread.sleep(1000);  // Sleep for 1 second

// Join - wait for another thread to complete
thread.join();        // Wait indefinitely
thread.join(5000);    // Wait max 5 seconds

// Yield - hint to scheduler to give other threads a chance
Thread.yield();

// Interrupt - request thread to stop
thread.interrupt();
thread.isInterrupted();  // Check if interrupted

// Daemon thread (background thread)
thread.setDaemon(true);  // Must be set before start()
thread.isDaemon();
```

### Q: What is a Daemon Thread?

Daemon threads are background threads that don't prevent JVM from exiting.

```java
Thread daemonThread = new Thread(() -> {
    while (true) {
        System.out.println("Daemon running...");
        Thread.sleep(1000);
    }
});
daemonThread.setDaemon(true);  // MUST be before start()
daemonThread.start();

// When all non-daemon threads finish, JVM exits
// Daemon threads are abruptly terminated
```

**Use Cases:** Garbage Collector, Signal Handlers, Background cleanup tasks

---

## 4. Race Conditions & Thread Safety

### Q: What is a Race Condition?

A race condition occurs when multiple threads access shared data simultaneously, and the outcome depends on the order of execution.

```java
// PROBLEM: Race Condition Example
class Counter {
    private int count = 0;

    public void increment() {
        count++;  // NOT ATOMIC! Actually: read -> increment -> write
    }

    public int getCount() {
        return count;
    }
}

// Main
Counter counter = new Counter();

// Two threads incrementing
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) counter.increment();
});
Thread t2 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) counter.increment();
});

t1.start(); t2.start();
t1.join(); t2.join();

System.out.println(counter.getCount());  // Expected: 20000, Actual: Random < 20000!
```

**Why does this happen?**

```
count++ is NOT atomic - it's actually 3 operations:

Thread 1                    Thread 2                    count (memory)
────────                    ────────                    ──────────────
                                                              0
READ count (0)                                               0
                            READ count (0)                   0
INCREMENT (0→1)                                              0
                            INCREMENT (0→1)                  0
WRITE count (1)                                              1
                            WRITE count (1)                  1  ← LOST UPDATE!

Expected: 2, Got: 1
```

### Q: What is Thread Safety?

A class is thread-safe if it behaves correctly when accessed from multiple threads, regardless of scheduling or interleaving of those threads.

**Ways to achieve Thread Safety:**
1. **Synchronization** - `synchronized` keyword
2. **Volatile variables** - for visibility
3. **Atomic classes** - `AtomicInteger`, etc.
4. **Immutable objects** - objects that can't be modified
5. **Thread-local storage** - `ThreadLocal`
6. **Concurrent collections** - `ConcurrentHashMap`, etc.

---

## 5. synchronized Keyword

### Q: What is synchronized and how does it work?

`synchronized` provides two guarantees:
1. **Mutual Exclusion** - Only one thread can execute synchronized code at a time
2. **Visibility** - Changes made by one thread are visible to others

```
┌─────────────────────────────────────────────────────────────────┐
│                    synchronized MECHANISM                        │
│                                                                  │
│   Thread 1          LOCK          Thread 2                      │
│       │              │                │                          │
│       │   acquire    │                │                          │
│       ├─────────────►│                │                          │
│       │              │◄───────────────┤ acquire (BLOCKED)       │
│       │   (owns      │    waiting     │                          │
│       │    lock)     │                │                          │
│       │              │                │                          │
│       │   release    │                │                          │
│       ├─────────────►│                │                          │
│       │              ├───────────────►│ (now owns lock)         │
│       │              │                │                          │
└─────────────────────────────────────────────────────────────────┘
```

### Q: Different ways to use synchronized

#### 1. Synchronized Method (Instance Level)

```java
class Counter {
    private int count = 0;

    // Lock is on 'this' object
    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

#### 2. Synchronized Block (More Flexible)

```java
class Counter {
    private int count = 0;
    private final Object lock = new Object();  // Dedicated lock object

    public void increment() {
        synchronized (lock) {  // Lock on specific object
            count++;
        }
    }

    // Can also lock on 'this'
    public void increment2() {
        synchronized (this) {
            count++;
        }
    }
}
```

#### 3. Static Synchronized Method (Class Level)

```java
class Counter {
    private static int count = 0;

    // Lock is on Counter.class object
    public static synchronized void increment() {
        count++;
    }

    // Equivalent to:
    public static void increment2() {
        synchronized (Counter.class) {
            count++;
        }
    }
}
```

### Q: Instance Lock vs Class Lock

```java
class Example {
    // Instance lock - different objects have different locks
    public synchronized void instanceMethod() { }

    // Class lock - one lock for all objects
    public static synchronized void staticMethod() { }
}

Example obj1 = new Example();
Example obj2 = new Example();

// These can run SIMULTANEOUSLY (different locks)
Thread t1 = new Thread(() -> obj1.instanceMethod());
Thread t2 = new Thread(() -> obj2.instanceMethod());

// These CANNOT run simultaneously (same class lock)
Thread t3 = new Thread(() -> Example.staticMethod());
Thread t4 = new Thread(() -> Example.staticMethod());

// Instance method and static method CAN run simultaneously (different locks)
Thread t5 = new Thread(() -> obj1.instanceMethod());
Thread t6 = new Thread(() -> Example.staticMethod());
```

### Q: What is Reentrant Synchronization?

A thread can acquire the same lock multiple times without blocking itself.

```java
class Reentrant {
    public synchronized void outer() {
        System.out.println("Outer");
        inner();  // Can call another synchronized method
    }

    public synchronized void inner() {
        System.out.println("Inner");  // Same thread, same lock - OK!
    }
}
```

### Fixed Counter Example

```java
class ThreadSafeCounter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}

// Now with two threads, result will always be 20000
```

---

## 6. volatile Keyword

### Q: What is volatile and when to use it?

`volatile` ensures **visibility** of changes across threads. When a variable is volatile, any write to it is immediately visible to all threads.

```java
class VolatileExample {
    private volatile boolean running = true;

    public void run() {
        while (running) {
            // do work
        }
        System.out.println("Stopped!");
    }

    public void stop() {
        running = false;  // Immediately visible to run() thread
    }
}
```

### Q: Why is volatile needed? (Memory Visibility Problem)

Without volatile, each thread may have its own cached copy:

```
WITHOUT volatile:
┌───────────────┐                               ┌───────────────┐
│   Thread 1    │                               │   Thread 2    │
│   ┌───────┐   │                               │   ┌───────┐   │
│   │ flag  │   │     ┌─────────────────┐      │   │ flag  │   │
│   │ =true │   │     │   Main Memory   │      │   │ =true │   │
│   │(cached)│   │     │   flag = true  │      │   │(cached)│   │
│   └───────┘   │     └─────────────────┘      │   └───────┘   │
└───────────────┘                               └───────────────┘

Thread 2 sets flag = false... but Thread 1 still sees true (from cache)!

WITH volatile:
┌───────────────┐                               ┌───────────────┐
│   Thread 1    │                               │   Thread 2    │
│      │        │     ┌─────────────────┐      │      │        │
│      │        │◄────│   Main Memory   │◄─────│      │        │
│      │        │     │   flag = false  │      │ flag = false  │
│   reads from  │     └─────────────────┘      │ writes to     │
│   main memory │                               │ main memory   │
└───────────────┘                               └───────────────┘
```

### Q: volatile vs synchronized - When to use which?

| Aspect | volatile | synchronized |
|--------|----------|--------------|
| **Purpose** | Visibility only | Visibility + Atomicity |
| **Blocking** | Non-blocking | Can block threads |
| **Compound operations** | NOT safe (count++) | Safe |
| **Performance** | Faster (no locking) | Slower (lock overhead) |
| **Use case** | Flags, single reads/writes | Complex operations |

### Q: When is volatile NOT enough?

```java
// WRONG - volatile doesn't make ++ atomic!
private volatile int count = 0;

public void increment() {
    count++;  // Still NOT thread-safe!
}

// For this, use AtomicInteger or synchronized
```

### Q: Classic volatile Use Case - Double-Checked Locking Singleton

```java
class Singleton {
    private static volatile Singleton instance;  // MUST be volatile!

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                    // First check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Why volatile is needed:** Without it, a thread might see a partially constructed object due to instruction reordering.

---

## 7. wait(), notify(), notifyAll()

### Q: What are wait(), notify(), and notifyAll()?

These are methods in `Object` class used for inter-thread communication.

| Method | Description |
|--------|-------------|
| `wait()` | Releases lock and waits until notified |
| `notify()` | Wakes up ONE waiting thread (random) |
| `notifyAll()` | Wakes up ALL waiting threads |

**Rules:**
1. Must be called from within a `synchronized` block/method
2. Must hold the lock on the object being waited on
3. `wait()` releases the lock; `notify()` does not release immediately

```
┌─────────────────────────────────────────────────────────────────────┐
│                    wait() / notify() FLOW                            │
│                                                                      │
│   Thread A                           Thread B                        │
│       │                                  │                           │
│  synchronized(obj) {                synchronized(obj) {              │
│       │                                  │                           │
│   obj.wait();  ─────────────────►   (waiting to                      │
│       │        releases lock         acquire lock)                   │
│       │        and waits                 │                           │
│       │                                  │                           │
│       │   ◄──── woken up ────────  obj.notify();                     │
│       │        but needs lock            │                           │
│       │        to continue               │                           │
│       │                                  │  releases lock            │
│   (continues after                   }   │  on block exit            │
│    reacquiring lock)                     │                           │
│  }                                       │                           │
│       │                                  │                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Q: Producer-Consumer using wait/notify

```java
class SharedBuffer {
    private Queue<Integer> buffer = new LinkedList<>();
    private int capacity;

    public SharedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public synchronized void produce(int item) throws InterruptedException {
        // Wait while buffer is full
        while (buffer.size() == capacity) {
            System.out.println("Buffer full, producer waiting...");
            wait();
        }

        buffer.add(item);
        System.out.println("Produced: " + item);

        // Notify consumers that data is available
        notifyAll();
    }

    public synchronized int consume() throws InterruptedException {
        // Wait while buffer is empty
        while (buffer.isEmpty()) {
            System.out.println("Buffer empty, consumer waiting...");
            wait();
        }

        int item = buffer.poll();
        System.out.println("Consumed: " + item);

        // Notify producers that space is available
        notifyAll();

        return item;
    }
}

// Usage
SharedBuffer buffer = new SharedBuffer(5);

Thread producer = new Thread(() -> {
    for (int i = 1; i <= 10; i++) {
        buffer.produce(i);
    }
});

Thread consumer = new Thread(() -> {
    for (int i = 1; i <= 10; i++) {
        buffer.consume();
    }
});

producer.start();
consumer.start();
```

### Q: Why use while loop instead of if with wait()?

```java
// WRONG - Spurious wakeups can cause issues
if (buffer.isEmpty()) {
    wait();
}
// Thread might continue even if condition is still true!

// CORRECT - Always use while loop
while (buffer.isEmpty()) {
    wait();
}
// Re-checks condition after waking up
```

**Reasons:**
1. **Spurious wakeups** - Thread can wake up without notify()
2. **Multiple waiters** - Another thread might have consumed the data before this thread gets the lock

### Q: notify() vs notifyAll() - Which to use?

| Scenario | Use |
|----------|-----|
| Single waiter | `notify()` is sufficient |
| Multiple waiters, any can proceed | `notify()` is fine |
| Multiple waiters with different conditions | `notifyAll()` is safer |
| In doubt | Use `notifyAll()` (safer, slightly slower) |

---

## 8. Locks & ReentrantLock

### Q: What is ReentrantLock and why use it over synchronized?

`ReentrantLock` is an explicit lock from `java.util.concurrent.locks` package that provides more flexibility than `synchronized`.

### Q: ReentrantLock vs synchronized

| Feature | synchronized | ReentrantLock |
|---------|--------------|---------------|
| Lock acquisition | Implicit | Explicit (lock()/unlock()) |
| Fairness | No | Optional (fair mode) |
| Try lock | No | Yes (tryLock()) |
| Timed lock | No | Yes (tryLock(time, unit)) |
| Interruptible | No | Yes (lockInterruptibly()) |
| Multiple conditions | One (wait/notify) | Multiple (newCondition()) |
| Automatic unlock | Yes (block exit) | No (must unlock in finally) |

### Q: Basic ReentrantLock Usage

```java
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();  // Acquire lock
        try {
            count++;
        } finally {
            lock.unlock();  // ALWAYS unlock in finally!
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### Q: tryLock() - Non-blocking Lock Acquisition

```java
ReentrantLock lock = new ReentrantLock();

public void doWork() {
    if (lock.tryLock()) {  // Returns immediately
        try {
            // Critical section
        } finally {
            lock.unlock();
        }
    } else {
        // Lock not available, do something else
        System.out.println("Could not acquire lock, trying later...");
    }
}

// With timeout
public void doWorkWithTimeout() throws InterruptedException {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {  // Wait up to 5 seconds
        try {
            // Critical section
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("Timeout waiting for lock");
    }
}
```

### Q: Fair vs Non-Fair Lock

```java
// Non-fair lock (default) - faster, but threads may starve
ReentrantLock unfairLock = new ReentrantLock();
ReentrantLock unfairLock2 = new ReentrantLock(false);

// Fair lock - slower, but threads acquire lock in order
ReentrantLock fairLock = new ReentrantLock(true);
```

```
NON-FAIR LOCK:                    FAIR LOCK:
┌──────────────────┐              ┌──────────────────┐
│ Thread A waiting │              │ Queue: A → B → C │
│ Thread B waiting │   ◄── New    │                  │   ◄── New thread
│ Thread C waiting │     thread   │ Served in order  │      joins queue
│                  │   can jump!  │                  │
└──────────────────┘              └──────────────────┘
```

### Q: Condition Variables (wait/notify alternative)

```java
import java.util.concurrent.locks.*;

class BoundedBuffer {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int capacity = 10;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // For producers
    private final Condition notEmpty = lock.newCondition();  // For consumers

    public void produce(int item) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == capacity) {
                notFull.await();  // Wait until not full
            }
            buffer.add(item);
            notEmpty.signal();  // Signal consumers
        } finally {
            lock.unlock();
        }
    }

    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                notEmpty.await();  // Wait until not empty
            }
            int item = buffer.poll();
            notFull.signal();  // Signal producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

**Advantage:** Different conditions for different waiting reasons = more efficient signaling.

---

## 9. ReadWriteLock

### Q: What is ReadWriteLock and when to use it?

`ReadWriteLock` provides separate locks for reading and writing, improving performance when reads are much more frequent than writes.

**Rules:**
- Multiple threads can hold the read lock simultaneously
- Only one thread can hold the write lock
- Write lock is exclusive (no reads allowed while writing)

```
┌─────────────────────────────────────────────────────────────────┐
│                    ReadWriteLock BEHAVIOR                        │
│                                                                  │
│   READING STATE:              WRITING STATE:                     │
│   ┌───────────────┐          ┌───────────────┐                  │
│   │ Reader 1 ✓    │          │ Writer 1 ✓    │                  │
│   │ Reader 2 ✓    │          │               │                  │
│   │ Reader 3 ✓    │          │ Reader 1 ✗    │ (blocked)        │
│   │               │          │ Reader 2 ✗    │ (blocked)        │
│   │ Writer 1 ✗    │(blocked) │ Writer 2 ✗    │ (blocked)        │
│   └───────────────┘          └───────────────┘                  │
│                                                                  │
│   Multiple readers OK        Only one writer, no readers        │
└─────────────────────────────────────────────────────────────────┘
```

### Q: ReadWriteLock Example - Cache Implementation

```java
import java.util.concurrent.locks.*;
import java.util.*;

class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple threads can read simultaneously
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only one thread can write at a time
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // All readers can call this simultaneously
    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
}
```

### Q: When to use ReadWriteLock vs synchronized?

| Scenario | Best Choice |
|----------|-------------|
| Read-heavy (90% reads) | ReadWriteLock |
| Write-heavy | synchronized or ReentrantLock |
| Balanced reads/writes | synchronized (simpler) |
| Short critical sections | synchronized |

---

## 10. Atomic Variables

### Q: What are Atomic classes and why use them?

Atomic classes provide lock-free, thread-safe operations on single variables using **Compare-And-Swap (CAS)**.

**Common Atomic Classes:**
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`
- `AtomicReference<V>`
- `AtomicIntegerArray`, `AtomicLongArray`, `AtomicReferenceArray<V>`
- `LongAdder`, `DoubleAdder` (for high contention)

### Q: How does CAS (Compare-And-Swap) work?

```
CAS Operation: compareAndSwap(expected, newValue)

┌──────────────────────────────────────────────────────────┐
│  Current Memory Value: 5                                  │
│                                                          │
│  Thread 1: CAS(5, 6)        Thread 2: CAS(5, 7)         │
│       │                           │                      │
│       ▼                           ▼                      │
│  Is current == 5?            Is current == 5?           │
│       │                           │                      │
│      YES                         YES                     │
│       │                           │                      │
│  Set to 6                    Wait... Thread 1 first     │
│  Return SUCCESS                   │                      │
│                              Is current == 6? (Changed!) │
│                                   │                      │
│                              NO (expected was 5)         │
│                              Return FAILURE, retry       │
└──────────────────────────────────────────────────────────┘
```

### Q: AtomicInteger Example

```java
import java.util.concurrent.atomic.AtomicInteger;

class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();  // Atomic!
    }

    public int getCount() {
        return count.get();
    }
}

// Common AtomicInteger methods
AtomicInteger ai = new AtomicInteger(0);

ai.get();                    // Returns current value
ai.set(10);                  // Sets value
ai.getAndSet(20);           // Returns old, sets new
ai.incrementAndGet();        // ++value (returns new)
ai.getAndIncrement();        // value++ (returns old)
ai.decrementAndGet();        // --value
ai.addAndGet(5);            // value += 5 (returns new)
ai.getAndAdd(5);            // value += 5 (returns old)
ai.compareAndSet(20, 30);   // If value==20, set to 30
ai.updateAndGet(x -> x * 2); // Apply function atomically
```

### Q: AtomicReference Example - Lock-free Stack

```java
import java.util.concurrent.atomic.AtomicReference;

class LockFreeStack<T> {
    private static class Node<T> {
        final T value;
        Node<T> next;

        Node(T value) {
            this.value = value;
        }
    }

    private AtomicReference<Node<T>> top = new AtomicReference<>();

    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        Node<T> oldTop;
        do {
            oldTop = top.get();
            newNode.next = oldTop;
        } while (!top.compareAndSet(oldTop, newNode));  // Retry until success
    }

    public T pop() {
        Node<T> oldTop;
        Node<T> newTop;
        do {
            oldTop = top.get();
            if (oldTop == null) return null;
            newTop = oldTop.next;
        } while (!top.compareAndSet(oldTop, newTop));
        return oldTop.value;
    }
}
```

### Q: LongAdder for High Contention

When many threads are updating the same counter, `LongAdder` is faster than `AtomicLong`:

```java
import java.util.concurrent.atomic.LongAdder;

LongAdder counter = new LongAdder();

// Multiple threads can call these concurrently
counter.add(1);
counter.increment();
counter.decrement();

// Get the sum (slightly expensive)
long sum = counter.sum();
```

**Why faster?** LongAdder maintains multiple cells, reducing contention:

```
AtomicLong:                    LongAdder:
┌─────────────┐               ┌─────────────────────┐
│   value=0   │◄── All        │ Cell1=0 │ Cell2=0   │
└─────────────┘    threads    │ Cell3=0 │ Cell4=0   │
                   contend    └─────────────────────┘
                                Different threads update
                                different cells!
                                sum() adds all cells
```

---

## 11. Thread Pools & ExecutorService

### Q: Why use Thread Pools?

Creating threads is expensive (memory, OS resources). Thread pools:
1. **Reuse threads** - No repeated creation/destruction
2. **Control concurrency** - Limit number of concurrent threads
3. **Manage resources** - Prevent resource exhaustion
4. **Provide task queuing** - Handle bursts of requests

### Q: Executors Framework Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTORS HIERARCHY                           │
│                                                                  │
│                      ┌──────────────┐                           │
│                      │   Executor   │  (Interface)              │
│                      │  execute()   │                           │
│                      └──────┬───────┘                           │
│                             │                                    │
│                    ┌────────▼────────┐                          │
│                    │ ExecutorService │  (Interface)             │
│                    │  submit()       │                          │
│                    │  shutdown()     │                          │
│                    │  invokeAll()    │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│          ┌──────────────────┼──────────────────┐                │
│          ▼                  ▼                  ▼                │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐     │
│  │ThreadPoolExec.│  │ScheduledExec. │  │  ForkJoinPool   │     │
│  └───────────────┘  └───────────────┘  └─────────────────┘     │
│                                                                  │
│  Created via Executors factory methods                          │
└─────────────────────────────────────────────────────────────────┘
```

### Q: Types of Thread Pools

```java
import java.util.concurrent.*;

// 1. Fixed Thread Pool - Fixed number of threads
ExecutorService fixed = Executors.newFixedThreadPool(10);
// Use when: You know the optimal thread count
// Threads reused, queue for excess tasks

// 2. Cached Thread Pool - Grows/shrinks dynamically
ExecutorService cached = Executors.newCachedThreadPool();
// Use when: Many short-lived tasks
// Creates new threads as needed, reuses idle ones

// 3. Single Thread Executor - Only one thread
ExecutorService single = Executors.newSingleThreadExecutor();
// Use when: Tasks must run sequentially
// Guarantees order of execution

// 4. Scheduled Thread Pool - For delayed/periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
// Use when: Need to schedule tasks

// 5. Work Stealing Pool (Java 8+) - Uses ForkJoinPool
ExecutorService workStealing = Executors.newWorkStealingPool();
// Use when: Recursive/divide-and-conquer tasks
```

### Q: Basic ExecutorService Usage

```java
ExecutorService executor = Executors.newFixedThreadPool(3);

// Submit Runnable (no result)
executor.execute(() -> System.out.println("Task 1"));

// Submit Callable (returns result)
Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "Task completed!";
});

// Get result (blocks until complete)
String result = future.get();
System.out.println(result);

// ALWAYS shutdown when done!
executor.shutdown();  // Graceful - waits for tasks to complete

// Or force shutdown
executor.shutdownNow();  // Interrupts running tasks

// Wait for termination
executor.awaitTermination(5, TimeUnit.SECONDS);
```

### Q: Submit Multiple Tasks

```java
ExecutorService executor = Executors.newFixedThreadPool(5);

List<Callable<Integer>> tasks = Arrays.asList(
    () -> { Thread.sleep(1000); return 1; },
    () -> { Thread.sleep(2000); return 2; },
    () -> { Thread.sleep(500);  return 3; }
);

// invokeAll - Wait for ALL to complete
List<Future<Integer>> results = executor.invokeAll(tasks);
for (Future<Integer> future : results) {
    System.out.println(future.get());  // 1, 2, 3
}

// invokeAny - Return FIRST completed result
Integer firstResult = executor.invokeAny(tasks);  // Returns 3 (fastest)

executor.shutdown();
```

### Q: Custom ThreadPoolExecutor

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                           // corePoolSize
    4,                           // maximumPoolSize
    60, TimeUnit.SECONDS,        // keepAliveTime for idle threads
    new LinkedBlockingQueue<>(100),  // Work queue (bounded)
    new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
);

// Parameters explained:
// - corePoolSize: Min threads kept alive
// - maximumPoolSize: Max threads allowed
// - keepAliveTime: Idle time before non-core threads are terminated
// - workQueue: Queue to hold pending tasks
// - rejectionPolicy: What to do when queue is full
```

### Q: ThreadPoolExecutor Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ThreadPoolExecutor FLOW                                   │
│                                                                              │
│   New Task                                                                   │
│      │                                                                       │
│      ▼                                                                       │
│  ┌────────────────────┐                                                     │
│  │ Active threads <   │ ────YES────► Create new thread, run task            │
│  │   corePoolSize?    │                                                     │
│  └────────┬───────────┘                                                     │
│           │NO                                                                │
│           ▼                                                                  │
│  ┌────────────────────┐                                                     │
│  │ Queue has space?   │ ────YES────► Add task to queue                      │
│  └────────┬───────────┘                                                     │
│           │NO                                                                │
│           ▼                                                                  │
│  ┌────────────────────┐                                                     │
│  │ Active threads <   │ ────YES────► Create new thread, run task            │
│  │  maximumPoolSize?  │                                                     │
│  └────────┬───────────┘                                                     │
│           │NO                                                                │
│           ▼                                                                  │
│  ┌────────────────────┐                                                     │
│  │  Apply rejection   │ (AbortPolicy, CallerRunsPolicy, etc.)               │
│  │      policy        │                                                     │
│  └────────────────────┘                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Q: Rejection Policies

| Policy | Behavior |
|--------|----------|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread executes the task |
| `DiscardPolicy` | Silently discards the task |
| `DiscardOldestPolicy` | Discards oldest queued task, retries |

---

## 12. Future & Callable

### Q: What is the difference between Runnable and Callable?

| Aspect | Runnable | Callable<V> |
|--------|----------|-------------|
| Method | `void run()` | `V call() throws Exception` |
| Return value | None | Returns result |
| Exceptions | Cannot throw checked exceptions | Can throw exceptions |
| Use with | `Thread`, `Executor` | Only `ExecutorService` |

### Q: What is Future?

`Future` represents the result of an asynchronous computation.

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

// Submit Callable, get Future
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(2000);
    return 42;
});

// Check if done (non-blocking)
boolean isDone = future.isDone();

// Cancel the task
boolean cancelled = future.cancel(true);  // true = interrupt if running

// Check if cancelled
boolean isCancelled = future.isCancelled();

// Get result (BLOCKS until complete)
try {
    Integer result = future.get();  // Blocks indefinitely
    // Or with timeout
    Integer result2 = future.get(5, TimeUnit.SECONDS);  // Throws TimeoutException if not done
} catch (InterruptedException e) {
    // Thread was interrupted while waiting
} catch (ExecutionException e) {
    // Task threw an exception
    Throwable cause = e.getCause();  // Get the actual exception
} catch (TimeoutException e) {
    // Timeout expired
}

executor.shutdown();
```

### Q: Future Limitations

```
Problems with Future:
1. get() is BLOCKING - wastes thread resources
2. Cannot chain/compose multiple Futures
3. No callback mechanism
4. Cannot manually complete
5. No exception handling pipeline

Future<A> futureA = executor.submit(taskA);
Future<B> futureB = executor.submit(taskB);

// To combine results, must block on both:
A resultA = futureA.get();  // BLOCKS!
B resultB = futureB.get();  // BLOCKS!

// No way to say: "When A is done, then do X"
```

**Solution:** `CompletableFuture` (next section)

---

## 13. CompletableFuture

### Q: What is CompletableFuture?

`CompletableFuture` is an enhanced `Future` that supports:
- Non-blocking operations
- Chaining/composition of async operations
- Exception handling
- Manual completion
- Combining multiple futures

### Q: Creating CompletableFuture

```java
import java.util.concurrent.CompletableFuture;

// 1. Run async task (no return value)
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});

// 2. Supply async value (returns value)
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "Hello, World!";
});

// 3. Already completed future
CompletableFuture<String> completed = CompletableFuture.completedFuture("Already done!");

// 4. Manually completable future
CompletableFuture<String> manual = new CompletableFuture<>();
// ... later ...
manual.complete("Completed manually!");
```

### Q: Transformation Methods (thenApply, thenAccept, thenRun)

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");

// thenApply - Transform result (Function)
CompletableFuture<Integer> lengthFuture = future.thenApply(s -> s.length());
// Hello → 5

// thenAccept - Consume result, no return (Consumer)
CompletableFuture<Void> printFuture = future.thenAccept(s -> System.out.println(s));
// Prints "Hello"

// thenRun - Run action after completion, no access to result (Runnable)
CompletableFuture<Void> doneFuture = future.thenRun(() -> System.out.println("Done!"));
// Prints "Done!"
```

```
┌─────────────────────────────────────────────────────────────────────┐
│            CHAINING FLOW                                             │
│                                                                      │
│   supplyAsync("Hello")                                               │
│         │                                                            │
│         ▼  thenApply(s -> s.toUpperCase())                          │
│      "HELLO"                                                         │
│         │                                                            │
│         ▼  thenApply(s -> s + "!")                                  │
│      "HELLO!"                                                        │
│         │                                                            │
│         ▼  thenAccept(System.out::println)                          │
│      [Prints: HELLO!]                                                │
│         │                                                            │
│         ▼  thenRun(() -> log("Completed"))                          │
│      [Logs: Completed]                                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Q: Chaining Example

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        // Simulate API call
        return "user123";
    })
    .thenApply(userId -> {
        // Fetch user data
        return "User: " + userId;
    })
    .thenApply(user -> {
        // Transform data
        return user.toUpperCase();
    });

String result = future.get();  // "USER: USER123"
```

### Q: Async Variants (thenApplyAsync, etc.)

```java
// Same thread as previous stage (or caller if already complete)
future.thenApply(s -> s.length());

// Different thread (from ForkJoinPool.commonPool())
future.thenApplyAsync(s -> s.length());

// Custom executor
future.thenApplyAsync(s -> s.length(), myExecutor);
```

### Q: Composing Futures (thenCompose vs thenCombine)

```java
// thenCompose - Flatten nested CompletableFuture (like flatMap)
// Use when: Next stage also returns CompletableFuture
CompletableFuture<String> result = getUserId()
    .thenCompose(userId -> getUser(userId));  // getUser returns CompletableFuture<User>

// thenCombine - Combine two independent futures
// Use when: Need results from two unrelated futures
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = future1.thenCombine(future2,
    (s1, s2) -> s1 + " " + s2);  // "Hello World"
```

```
thenCompose (Sequential dependency):
┌────────┐       ┌────────────────────┐       ┌──────────┐
│ Task A │ ────► │ Result A → Task B  │ ────► │ Result B │
└────────┘       └────────────────────┘       └──────────┘

thenCombine (Parallel, then combine):
┌────────┐
│ Task A │ ────┐
└────────┘     │    ┌────────────────────┐
               ├───►│ Combine A + B      │ ────► Result
┌────────┐     │    └────────────────────┘
│ Task B │ ────┘
└────────┘
```

### Q: Handling Multiple Futures

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> "C");

// allOf - Wait for ALL to complete (returns Void)
CompletableFuture<Void> allDone = CompletableFuture.allOf(f1, f2, f3);
allDone.get();  // Blocks until all complete

// Get individual results after allOf
String r1 = f1.get();
String r2 = f2.get();
String r3 = f3.get();

// anyOf - Complete when ANY one completes
CompletableFuture<Object> anyDone = CompletableFuture.anyOf(f1, f2, f3);
Object firstResult = anyDone.get();  // Returns first completed result
```

### Q: Exception Handling

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) {
            throw new RuntimeException("Something went wrong!");
        }
        return "Success";
    })
    // exceptionally - Handle exception, return fallback
    .exceptionally(ex -> {
        System.err.println("Error: " + ex.getMessage());
        return "Fallback value";
    });

// handle - Handle both success and failure
CompletableFuture<String> handled = CompletableFuture
    .supplyAsync(() -> "Success")
    .handle((result, ex) -> {
        if (ex != null) {
            return "Error: " + ex.getMessage();
        }
        return "Result: " + result;
    });

// whenComplete - Side effect, doesn't change result
CompletableFuture<String> withSideEffect = CompletableFuture
    .supplyAsync(() -> "Success")
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed", ex);
        } else {
            log.info("Completed with: " + result);
        }
    });
```

### Q: Complete Example - Async API Calls

```java
public class AsyncApiExample {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public CompletableFuture<OrderDetails> getOrderDetails(String orderId) {
        return CompletableFuture.supplyAsync(() -> fetchOrder(orderId), executor)
            .thenCompose(order -> {
                // Fetch user and products in parallel
                CompletableFuture<User> userFuture =
                    CompletableFuture.supplyAsync(() -> fetchUser(order.getUserId()), executor);
                CompletableFuture<List<Product>> productsFuture =
                    CompletableFuture.supplyAsync(() -> fetchProducts(order.getProductIds()), executor);

                // Combine results
                return userFuture.thenCombine(productsFuture, (user, products) ->
                    new OrderDetails(order, user, products)
                );
            })
            .exceptionally(ex -> {
                log.error("Failed to fetch order details", ex);
                return OrderDetails.empty();
            });
    }

    // Usage
    public void processOrder(String orderId) {
        getOrderDetails(orderId)
            .thenAccept(details -> System.out.println("Order: " + details))
            .join();  // Block only at the end
    }
}
```

---

## 14. BlockingQueue

### Q: What is BlockingQueue?

`BlockingQueue` is a thread-safe queue that blocks when:
- `put()`: Queue is full (waits for space)
- `take()`: Queue is empty (waits for element)

```
┌─────────────────────────────────────────────────────────────────┐
│                    BlockingQueue OPERATIONS                      │
│                                                                  │
│   ┌───────────────────────────────────────────────────────┐     │
│   │ │ A │ B │ C │ D │ E │   │   │   │   │                │     │
│   └───────────────────────────────────────────────────────┘     │
│     ▲                                   ▲                        │
│     │                                   │                        │
│   PUT                                 TAKE                       │
│   (blocks if full)                   (blocks if empty)           │
│                                                                  │
│   Throws     Returns     Blocks    Times out                     │
│   exception  special     (waits)   (waits for time)              │
│   ─────────  ─────────   ─────────  ─────────────                │
│   add(e)     offer(e)    put(e)    offer(e, time, unit)         │
│   remove()   poll()      take()    poll(time, unit)              │
│   element()  peek()        -         -                           │
└─────────────────────────────────────────────────────────────────┘
```

### Q: BlockingQueue Implementations

| Implementation | Characteristics |
|----------------|-----------------|
| `ArrayBlockingQueue` | Bounded, fixed size, FIFO |
| `LinkedBlockingQueue` | Optionally bounded, linked nodes |
| `PriorityBlockingQueue` | Unbounded, priority ordering |
| `DelayQueue` | Elements available after delay |
| `SynchronousQueue` | Zero capacity, direct handoff |

### Q: Producer-Consumer with BlockingQueue

```java
import java.util.concurrent.*;

class ProducerConsumerDemo {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

        // Producer
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 20; i++) {
                    System.out.println("Producing: " + i);
                    queue.put(i);  // Blocks if queue is full
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // Consumer
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Integer item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consuming: " + item);
                    Thread.sleep(200);  // Consumer is slower
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

### Q: Implement Custom BlockingQueue (Interview Favorite!)

```java
import java.util.LinkedList;
import java.util.Queue;

class MyBlockingQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public MyBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // Wait until space available
        }
        queue.add(item);
        notifyAll();  // Notify waiting consumers
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // Wait until item available
        }
        T item = queue.poll();
        notifyAll();  // Notify waiting producers
        return item;
    }

    public synchronized int size() {
        return queue.size();
    }
}
```

### Q: Using ReentrantLock Version (More Efficient)

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.*;

class MyBlockingQueueWithLock<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public MyBlockingQueueWithLock(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }
            queue.add(item);
            notEmpty.signal();  // Only signal consumers (more efficient)
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            notFull.signal();  // Only signal producers (more efficient)
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 15. Concurrent Collections

### Q: Why do we need Concurrent Collections?

Regular collections are NOT thread-safe. Using synchronized wrappers (`Collections.synchronizedList()`) has limitations:
- Locks the entire collection
- Compound operations are not atomic
- Iteration requires external synchronization

### Q: ConcurrentHashMap

The most important concurrent collection. Thread-safe HashMap with better concurrency.

```java
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Basic operations - thread-safe
map.put("key", 1);
map.get("key");
map.remove("key");

// Atomic compound operations
map.putIfAbsent("key", 1);       // Only puts if key doesn't exist
map.computeIfAbsent("key", k -> expensiveComputation(k));
map.computeIfPresent("key", (k, v) -> v + 1);
map.compute("key", (k, v) -> v == null ? 1 : v + 1);
map.merge("key", 1, Integer::sum);  // Increment or set to 1

// Atomic replace
map.replace("key", 2);           // Replace if key exists
map.replace("key", 1, 2);        // Replace only if current value is 1

// Safe iteration (weakly consistent - may see updates)
map.forEach((k, v) -> System.out.println(k + ": " + v));
```

### Q: ConcurrentHashMap Internal Working (Interview Favorite!)

```
Java 7 - Segment-based locking:
┌─────────────────────────────────────────────────────────────┐
│ Segment 0  │ Segment 1  │ Segment 2  │ Segment 3  │ ...    │
│ (Lock 0)   │ (Lock 1)   │ (Lock 2)   │ (Lock 3)   │        │
│ ┌────────┐ │ ┌────────┐ │ ┌────────┐ │ ┌────────┐ │        │
│ │bucket 0│ │ │bucket 1│ │ │bucket 2│ │ │bucket 3│ │        │
│ │bucket 4│ │ │bucket 5│ │ │bucket 6│ │ │bucket 7│ │        │
│ └────────┘ │ └────────┘ │ └────────┘ │ └────────┘ │        │
└─────────────────────────────────────────────────────────────┘
Multiple threads can update different segments concurrently!

Java 8+ - Node-level locking (CAS + synchronized on first node):
┌─────────────────────────────────────────────────────────────┐
│ Bucket 0   │ Bucket 1   │ Bucket 2   │ Bucket 3   │ ...    │
│ [Node A]   │ [Node D]   │ [null]     │ [Node F]   │        │
│     │      │     │      │            │     │      │        │
│ [Node B]   │ [Node E]   │            │ [Node G]   │        │
│     │      │            │            │     │      │        │
│ [Node C]   │            │            │ [Tree]     │        │
└─────────────────────────────────────────────────────────────┘
Fine-grained locking on individual buckets!
```

### Q: HashMap vs Hashtable vs ConcurrentHashMap

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---------|---------|-----------|-------------------|
| Thread-safe | No | Yes | Yes |
| Null key/value | Yes (one null key) | No | No |
| Locking | None | Entire table | Segment/Node level |
| Performance | Best (single thread) | Worst | Best (multi-thread) |
| Iterator | Fail-fast | Fail-fast | Weakly consistent |

### Q: CopyOnWriteArrayList

Creates a new copy of the array on every write. Good for read-heavy scenarios.

```java
import java.util.concurrent.CopyOnWriteArrayList;

CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("A");
list.add("B");

// Iteration is safe - iterates over snapshot
for (String item : list) {
    list.add("C");  // Doesn't cause ConcurrentModificationException!
    // But iterator won't see "C"
}
```

**When to use:**
- Listeners/Observers lists
- Read-heavy, write-rare scenarios
- When you can't afford read locks

### Q: Other Concurrent Collections

```java
// CopyOnWriteArraySet - Set version of CopyOnWriteArrayList
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();

// ConcurrentLinkedQueue - Non-blocking queue
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();

// ConcurrentSkipListMap - Sorted concurrent map (like TreeMap)
ConcurrentSkipListMap<String, Integer> sortedMap = new ConcurrentSkipListMap<>();

// ConcurrentSkipListSet - Sorted concurrent set (like TreeSet)
ConcurrentSkipListSet<String> sortedSet = new ConcurrentSkipListSet<>();
```

---

## 16. CountDownLatch & CyclicBarrier

### Q: What is CountDownLatch?

A synchronization aid that allows threads to wait until a set of operations completes.

```
CountDownLatch (count = 3):
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Thread A: countDown() ──────┐                                 │
│                               │                                  │
│   Thread B: countDown() ──────┼────► Count: 3 → 2 → 1 → 0       │
│                               │            │   │   │   │        │
│   Thread C: countDown() ──────┘            │   │   │   │        │
│                                            │   │   │   ▼        │
│   Main Thread: await() ─────────────────────────────── UNBLOCK  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

One-time use only! Cannot be reset.
```

### Q: CountDownLatch Example

```java
import java.util.concurrent.CountDownLatch;

class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int numWorkers = 3;
        CountDownLatch latch = new CountDownLatch(numWorkers);

        // Start workers
        for (int i = 1; i <= numWorkers; i++) {
            final int workerId = i;
            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " starting...");
                    Thread.sleep((long) (Math.random() * 2000));
                    System.out.println("Worker " + workerId + " done!");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();  // Decrement count
                }
            }).start();
        }

        System.out.println("Main thread waiting for workers...");
        latch.await();  // Block until count reaches 0
        System.out.println("All workers completed! Main thread continuing...");
    }
}
```

**Use Cases:**
- Wait for multiple services to start before proceeding
- Parallel task completion
- Test harness - start all threads simultaneously

### Q: What is CyclicBarrier?

Allows threads to wait for each other at a common barrier point. Unlike CountDownLatch, it can be reused.

```
CyclicBarrier (parties = 3):
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Thread A: await() ────► [waiting]  ─┐                         │
│                                       │                          │
│   Thread B: await() ────► [waiting]  ─┼──► All arrived! PROCEED │
│                                       │    (barrier action runs) │
│   Thread C: await() ────► [waiting]  ─┘                         │
│                                                                  │
│   Barrier resets - can be used again!                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Q: CyclicBarrier Example

```java
import java.util.concurrent.CyclicBarrier;

class CyclicBarrierDemo {
    public static void main(String[] args) {
        int numThreads = 3;

        // Optional barrier action runs when all threads arrive
        Runnable barrierAction = () -> System.out.println("Barrier reached! All threads synchronized.");

        CyclicBarrier barrier = new CyclicBarrier(numThreads, barrierAction);

        for (int i = 1; i <= numThreads; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    // Phase 1
                    System.out.println("Thread " + threadId + " - Phase 1 working...");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println("Thread " + threadId + " - Phase 1 complete, waiting at barrier...");
                    barrier.await();

                    // Phase 2 (barrier reused!)
                    System.out.println("Thread " + threadId + " - Phase 2 working...");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println("Thread " + threadId + " - Phase 2 complete, waiting at barrier...");
                    barrier.await();

                    System.out.println("Thread " + threadId + " - All done!");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### Q: CountDownLatch vs CyclicBarrier

| Feature | CountDownLatch | CyclicBarrier |
|---------|----------------|---------------|
| Reusable | No (one-time use) | Yes (reset after each use) |
| Who counts | Any thread can countDown() | Waiting threads |
| Waiting | One/more threads wait for others | Threads wait for each other |
| Use case | Wait for N events | Synchronize N threads at point |

---

## 17. Semaphore

### Q: What is a Semaphore?

A semaphore controls access to a shared resource with a limited number of permits.

```
Semaphore (permits = 3):
┌─────────────────────────────────────────────────────────────────┐
│   Available Permits: 3                                           │
│                                                                  │
│   Thread A: acquire() ──► [Gets permit, permits = 2]            │
│   Thread B: acquire() ──► [Gets permit, permits = 1]            │
│   Thread C: acquire() ──► [Gets permit, permits = 0]            │
│   Thread D: acquire() ──► [BLOCKS - no permits available]       │
│                                                                  │
│   Thread A: release() ──► [Releases permit, permits = 1]        │
│   Thread D:           ──► [UNBLOCKS - gets permit, permits = 0] │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Q: Semaphore Example - Connection Pool

```java
import java.util.concurrent.Semaphore;

class ConnectionPool {
    private final Semaphore semaphore;
    private final Connection[] connections;
    private final boolean[] used;

    public ConnectionPool(int poolSize) {
        this.semaphore = new Semaphore(poolSize);
        this.connections = new Connection[poolSize];
        this.used = new boolean[poolSize];

        // Initialize connections
        for (int i = 0; i < poolSize; i++) {
            connections[i] = new Connection(i);
        }
    }

    public Connection acquire() throws InterruptedException {
        semaphore.acquire();  // Blocks if no permits
        return getConnection();
    }

    public void release(Connection connection) {
        if (returnConnection(connection)) {
            semaphore.release();  // Release permit
        }
    }

    private synchronized Connection getConnection() {
        for (int i = 0; i < connections.length; i++) {
            if (!used[i]) {
                used[i] = true;
                return connections[i];
            }
        }
        return null;  // Should never happen due to semaphore
    }

    private synchronized boolean returnConnection(Connection conn) {
        for (int i = 0; i < connections.length; i++) {
            if (connections[i] == conn) {
                used[i] = false;
                return true;
            }
        }
        return false;
    }
}
```

### Q: Binary Semaphore (Mutex)

```java
// Binary semaphore - acts like a mutex
Semaphore mutex = new Semaphore(1);

mutex.acquire();  // Lock
try {
    // Critical section
} finally {
    mutex.release();  // Unlock
}
```

---

## 18. Deadlock, Livelock & Starvation

### Q: What is Deadlock?

Deadlock occurs when two or more threads are blocked forever, each waiting for the other.

```
Deadlock Scenario:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Thread A                           Thread B                    │
│       │                                  │                       │
│   Holds Lock 1                      Holds Lock 2                 │
│       │                                  │                       │
│   Wants Lock 2 ──────────────────────►   │                      │
│       │                                  │                       │
│       │   ◄────────────────────── Wants Lock 1                  │
│       │                                  │                       │
│   [BLOCKED]                         [BLOCKED]                    │
│                                                                  │
│   Both waiting forever - DEADLOCK!                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Q: Deadlock Example

```java
class DeadlockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void method1() {
        synchronized (lock1) {
            System.out.println("Thread 1: Holding lock1...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 1: Waiting for lock2...");
            synchronized (lock2) {
                System.out.println("Thread 1: Holding lock1 and lock2");
            }
        }
    }

    public void method2() {
        synchronized (lock2) {  // REVERSED ORDER!
            System.out.println("Thread 2: Holding lock2...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 2: Waiting for lock1...");
            synchronized (lock1) {
                System.out.println("Thread 2: Holding lock2 and lock1");
            }
        }
    }

    public static void main(String[] args) {
        DeadlockExample example = new DeadlockExample();

        new Thread(() -> example.method1()).start();
        new Thread(() -> example.method2()).start();
        // DEADLOCK!
    }
}
```

### Q: How to Prevent Deadlock?

**1. Lock Ordering - Always acquire locks in same order**

```java
// FIXED - Same order
public void method1() {
    synchronized (lock1) {
        synchronized (lock2) { /* ... */ }
    }
}

public void method2() {
    synchronized (lock1) {  // Same order!
        synchronized (lock2) { /* ... */ }
    }
}
```

**2. Use tryLock() with timeout**

```java
public void safeMethod() {
    boolean gotLock1 = false;
    boolean gotLock2 = false;

    try {
        gotLock1 = lock1.tryLock(1, TimeUnit.SECONDS);
        gotLock2 = lock2.tryLock(1, TimeUnit.SECONDS);

        if (gotLock1 && gotLock2) {
            // Do work
        }
    } finally {
        if (gotLock2) lock2.unlock();
        if (gotLock1) lock1.unlock();
    }
}
```

**3. Use higher-level constructs (avoid manual locking)**

```java
// Use java.util.concurrent classes
ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
```

### Q: Four Conditions for Deadlock (Coffman Conditions)

All four must be true for deadlock:
1. **Mutual Exclusion** - Resource held exclusively
2. **Hold and Wait** - Thread holds one resource while waiting for another
3. **No Preemption** - Resources cannot be forcibly taken
4. **Circular Wait** - Circular chain of threads waiting

**Prevention:** Break any one condition!

### Q: What is Livelock?

Threads are not blocked but keep responding to each other without making progress.

```java
// Example: Two people in corridor trying to pass each other
// Both keep moving to same side, neither can pass

class LivelockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void method1() {
        while (true) {
            if (tryAcquire(lock1)) {
                if (tryAcquire(lock2)) {
                    // Do work
                    return;
                }
                release(lock1);  // Release and retry
            }
            // Back off and retry - but other thread does same!
        }
    }
}
```

**Solution:** Add random backoff delay

### Q: What is Starvation?

A thread never gets access to resource because other threads keep getting priority.

**Causes:**
- Low priority threads
- Unfair locks always favoring certain threads
- Poor scheduling

**Solutions:**
- Use fair locks: `new ReentrantLock(true)`
- Use fair semaphore: `new Semaphore(permits, true)`

---

## 19. Classic Concurrency Problems

### Q: Producer-Consumer Problem

See [BlockingQueue section](#14-blockingqueue) for implementation.

### Q: Readers-Writers Problem

Multiple readers OR one writer at a time.

```java
import java.util.concurrent.locks.*;

class ReadersWriters {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    private String data = "Initial Data";

    public String read() {
        readLock.lock();
        try {
            // Multiple readers can be here simultaneously
            return data;
        } finally {
            readLock.unlock();
        }
    }

    public void write(String newData) {
        writeLock.lock();
        try {
            // Only one writer at a time, no readers
            data = newData;
        } finally {
            writeLock.unlock();
        }
    }
}
```

### Q: Dining Philosophers Problem

Five philosophers sit at a table. Each needs two forks to eat. Solution must avoid deadlock.

```java
import java.util.concurrent.locks.*;

class DiningPhilosophers {
    private final int NUM_PHILOSOPHERS = 5;
    private final ReentrantLock[] forks = new ReentrantLock[NUM_PHILOSOPHERS];

    public DiningPhilosophers() {
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            forks[i] = new ReentrantLock();
        }
    }

    public void startDining() {
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            final int philosopherId = i;
            new Thread(() -> {
                try {
                    while (true) {
                        think(philosopherId);
                        eat(philosopherId);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }

    private void think(int id) throws InterruptedException {
        System.out.println("Philosopher " + id + " is thinking");
        Thread.sleep((long) (Math.random() * 1000));
    }

    private void eat(int id) throws InterruptedException {
        // Resource ordering: always pick lower-numbered fork first
        int left = id;
        int right = (id + 1) % NUM_PHILOSOPHERS;
        int first = Math.min(left, right);
        int second = Math.max(left, right);

        forks[first].lock();
        forks[second].lock();
        try {
            System.out.println("Philosopher " + id + " is eating");
            Thread.sleep((long) (Math.random() * 1000));
        } finally {
            forks[second].unlock();
            forks[first].unlock();
        }
    }
}
```

---

## 20. Interview Coding Problems

### Problem 1: Print Numbers Alternately (Odd-Even)

**Question:** Print numbers 1-20 using two threads - one prints odd, one prints even, in order.

```java
class OddEvenPrinter {
    private int count = 1;
    private final int MAX = 20;
    private final Object lock = new Object();

    public void printOdd() {
        synchronized (lock) {
            while (count <= MAX) {
                if (count % 2 == 0) {  // Not my turn
                    try { lock.wait(); } catch (InterruptedException e) {}
                } else {
                    System.out.println("Odd: " + count++);
                    lock.notify();
                }
            }
        }
    }

    public void printEven() {
        synchronized (lock) {
            while (count <= MAX) {
                if (count % 2 == 1) {  // Not my turn
                    try { lock.wait(); } catch (InterruptedException e) {}
                } else {
                    System.out.println("Even: " + count++);
                    lock.notify();
                }
            }
        }
    }

    public static void main(String[] args) {
        OddEvenPrinter printer = new OddEvenPrinter();

        new Thread(() -> printer.printOdd()).start();
        new Thread(() -> printer.printEven()).start();
    }
}
```

### Problem 2: Thread-Safe Singleton

```java
// Method 1: Eager Initialization (Thread-safe but creates even if not used)
class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() {}
    public static EagerSingleton getInstance() { return INSTANCE; }
}

// Method 2: Double-Checked Locking (Lazy + Thread-safe)
class DCLSingleton {
    private static volatile DCLSingleton instance;
    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (DCLSingleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// Method 3: Bill Pugh Singleton (Best - Lazy + Thread-safe + No synchronization)
class BillPughSingleton {
    private BillPughSingleton() {}

    // Inner class loaded only when getInstance() called
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}

// Method 4: Enum Singleton (Simplest + Safe from serialization/reflection)
enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        // business logic
    }
}
```

### Problem 3: Implement Thread-Safe Counter

```java
// Using synchronized
class SynchronizedCounter {
    private int count = 0;

    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

// Using AtomicInteger (Better performance)
class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); }
    public int get() { return count.get(); }
}

// Using ReentrantLock
class LockCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try { count++; }
        finally { lock.unlock(); }
    }

    public int get() {
        lock.lock();
        try { return count; }
        finally { lock.unlock(); }
    }
}
```

### Problem 4: Implement Custom Thread Pool

```java
import java.util.concurrent.*;

class CustomThreadPool {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<WorkerThread> threads;
    private volatile boolean isShutdown = false;

    public CustomThreadPool(int numThreads, int queueSize) {
        taskQueue = new LinkedBlockingQueue<>(queueSize);
        threads = new ArrayList<>();

        for (int i = 0; i < numThreads; i++) {
            WorkerThread worker = new WorkerThread();
            worker.start();
            threads.add(worker);
        }
    }

    public void execute(Runnable task) {
        if (isShutdown) throw new IllegalStateException("Pool is shutdown");
        try {
            taskQueue.put(task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public void shutdown() {
        isShutdown = true;
        for (WorkerThread thread : threads) {
            thread.interrupt();
        }
    }

    private class WorkerThread extends Thread {
        @Override
        public void run() {
            while (!isShutdown) {
                try {
                    Runnable task = taskQueue.take();
                    task.run();
                } catch (InterruptedException e) {
                    // Shutdown signal
                    break;
                }
            }
        }
    }
}
```

### Problem 5: Print FooBar Alternately

**Question:** (LeetCode 1115) Print "foobar" n times using two threads.

```java
class FooBar {
    private int n;
    private volatile boolean fooTurn = true;

    public FooBar(int n) {
        this.n = n;
    }

    public synchronized void foo(Runnable printFoo) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            while (!fooTurn) {
                wait();
            }
            printFoo.run();
            fooTurn = false;
            notify();
        }
    }

    public synchronized void bar(Runnable printBar) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            while (fooTurn) {
                wait();
            }
            printBar.run();
            fooTurn = true;
            notify();
        }
    }
}
```

### Problem 6: Rate Limiter

**Question:** Implement a rate limiter that allows max N requests per second.

```java
import java.util.concurrent.*;

class RateLimiter {
    private final Semaphore semaphore;
    private final int maxRequests;
    private final ScheduledExecutorService scheduler;

    public RateLimiter(int maxRequestsPerSecond) {
        this.maxRequests = maxRequestsPerSecond;
        this.semaphore = new Semaphore(maxRequestsPerSecond);
        this.scheduler = Executors.newScheduledThreadPool(1);

        // Replenish permits every second
        scheduler.scheduleAtFixedRate(() -> {
            int permitsToAdd = maxRequests - semaphore.availablePermits();
            semaphore.release(permitsToAdd);
        }, 1, 1, TimeUnit.SECONDS);
    }

    public boolean tryAcquire() {
        return semaphore.tryAcquire();
    }

    public void acquire() throws InterruptedException {
        semaphore.acquire();
    }
}
```

### Problem 7: Implement ReadWriteLock from Scratch

```java
class SimpleReadWriteLock {
    private int readers = 0;
    private int writers = 0;
    private int writeRequests = 0;

    public synchronized void lockRead() throws InterruptedException {
        while (writers > 0 || writeRequests > 0) {
            wait();
        }
        readers++;
    }

    public synchronized void unlockRead() {
        readers--;
        notifyAll();
    }

    public synchronized void lockWrite() throws InterruptedException {
        writeRequests++;
        try {
            while (readers > 0 || writers > 0) {
                wait();
            }
            writers++;
        } finally {
            writeRequests--;
        }
    }

    public synchronized void unlockWrite() {
        writers--;
        notifyAll();
    }
}
```

### Problem 8: H2O (LeetCode 1117)

**Question:** Create water molecules. For every 2 hydrogen threads, 1 oxygen thread should proceed.

```java
import java.util.concurrent.*;

class H2O {
    private Semaphore hydrogen = new Semaphore(2);
    private Semaphore oxygen = new Semaphore(0);
    private CyclicBarrier barrier = new CyclicBarrier(3, () -> {
        hydrogen.release(2);  // Reset for next molecule
    });

    public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
        hydrogen.acquire();
        try {
            releaseHydrogen.run();
            barrier.await();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }

    public void oxygen(Runnable releaseOxygen) throws InterruptedException {
        oxygen.acquire();
        try {
            releaseOxygen.run();
            barrier.await();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }

    public H2O() {
        // Start by allowing 2 hydrogen
        oxygen.release();
    }
}
```

---

## 21. Best Practices & Common Pitfalls

### Best Practices

1. **Prefer higher-level concurrency utilities**
   - Use `ExecutorService` instead of raw `Thread`
   - Use `ConcurrentHashMap` instead of synchronized HashMap
   - Use `BlockingQueue` instead of wait/notify

2. **Always release locks in finally block**
   ```java
   lock.lock();
   try {
       // work
   } finally {
       lock.unlock();  // ALWAYS here!
   }
   ```

3. **Minimize synchronized blocks**
   ```java
   // BAD - locks too long
   public synchronized void process() {
       heavyComputation();  // Doesn't need lock
       updateSharedState();  // Needs lock
   }

   // GOOD - lock only what's needed
   public void process() {
       heavyComputation();
       synchronized(this) {
           updateSharedState();
       }
   }
   ```

4. **Use immutable objects when possible**
   ```java
   // Thread-safe by design
   final class ImmutableUser {
       private final String name;
       private final int age;
       // Constructor and getters only
   }
   ```

5. **Use ThreadLocal for per-thread data**
   ```java
   private static final ThreadLocal<SimpleDateFormat> formatter =
       ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

   public String format(Date date) {
       return formatter.get().format(date);
   }
   ```

### Common Pitfalls

1. **Double-checked locking without volatile**
   ```java
   // BROKEN!
   private static Singleton instance;
   public static Singleton getInstance() {
       if (instance == null) {
           synchronized(Singleton.class) {
               if (instance == null) {
                   instance = new Singleton();  // May be partially constructed!
               }
           }
       }
       return instance;
   }

   // FIXED: Add volatile
   private static volatile Singleton instance;
   ```

2. **Synchronizing on wrong object**
   ```java
   // WRONG - synchronizing on primitive wrapper
   private Integer count = 0;
   synchronized(count) { count++; }  // count changes, different object!

   // FIXED - use dedicated lock object
   private final Object lock = new Object();
   synchronized(lock) { count++; }
   ```

3. **Using wait() in if instead of while**
   ```java
   // WRONG - spurious wakeups
   if (queue.isEmpty()) wait();

   // CORRECT
   while (queue.isEmpty()) wait();
   ```

4. **Not handling InterruptedException properly**
   ```java
   // WRONG - swallowing interrupt
   try { Thread.sleep(1000); }
   catch (InterruptedException e) { /* ignore */ }

   // CORRECT - preserve interrupt status
   try { Thread.sleep(1000); }
   catch (InterruptedException e) {
       Thread.currentThread().interrupt();  // Restore interrupt flag
       return;  // Or handle appropriately
   }
   ```

5. **Publishing this reference in constructor**
   ```java
   // DANGEROUS - partially constructed object escapes
   public class BadClass {
       public BadClass(EventListener listener) {
           listener.register(this);  // 'this' escapes before construction complete!
       }
   }
   ```

---

## 22. Quick Revision Cheat Sheet

### Keywords

| Keyword | Purpose |
|---------|---------|
| `synchronized` | Mutual exclusion + visibility |
| `volatile` | Visibility only, no atomicity |
| `final` | Immutability, safe publication |

### Key Classes

| Class | Purpose |
|-------|---------|
| `Thread` | Basic thread |
| `Runnable` | Task without return |
| `Callable<V>` | Task with return |
| `Future<V>` | Result of async computation |
| `CompletableFuture<V>` | Enhanced async with composition |
| `ExecutorService` | Thread pool interface |
| `ReentrantLock` | Explicit lock |
| `ReadWriteLock` | Separate read/write locks |
| `Semaphore` | Permit-based access control |
| `CountDownLatch` | Wait for N events |
| `CyclicBarrier` | Synchronize N threads |
| `AtomicInteger` | Lock-free integer |
| `ConcurrentHashMap` | Thread-safe map |
| `BlockingQueue` | Thread-safe queue with blocking |

### Method Comparison

| Operation | synchronized | ReentrantLock |
|-----------|--------------|---------------|
| Lock | Enter block | `lock()` |
| Unlock | Exit block | `unlock()` |
| Try lock | Not available | `tryLock()` |
| Timed lock | Not available | `tryLock(time, unit)` |
| Interruptible | No | `lockInterruptibly()` |
| Condition | `wait()/notify()` | `condition.await()/signal()` |

### CompletableFuture Methods

| Method | Input | Output | Description |
|--------|-------|--------|-------------|
| `thenApply` | Function | New CF | Transform value |
| `thenAccept` | Consumer | Void CF | Consume value |
| `thenRun` | Runnable | Void CF | Run action |
| `thenCompose` | Function returning CF | New CF | Flatten nested CF |
| `thenCombine` | CF + BiFunction | New CF | Combine two CFs |
| `exceptionally` | Exception handler | CF | Handle errors |
| `handle` | BiFunction | CF | Handle both |
| `allOf` | CFs... | Void CF | Wait for all |
| `anyOf` | CFs... | Object CF | Wait for any |

### Thread Pool Types

| Type | Pool Size | Queue | Use Case |
|------|-----------|-------|----------|
| Fixed | Fixed | Unbounded | Known number of tasks |
| Cached | 0 to ∞ | Synchronous | Many short-lived tasks |
| Single | 1 | Unbounded | Sequential execution |
| Scheduled | Fixed | Delayed | Scheduled/periodic tasks |
| WorkStealing | CPU cores | Per-thread | Fork-join tasks |

---

## Additional Resources

1. **Java Concurrency in Practice** (Book by Brian Goetz) - The definitive guide
2. **Java Documentation** - `java.util.concurrent` package
3. **JMM (Java Memory Model)** - For deep understanding of visibility

---

*Good luck with your interviews! Remember: Practice writing these problems by hand and explaining your thought process.*
