---
title: JVM Concurrency Basics
date: 2022-Jul-15
template: article.pug
hide: false
category: article
---
Here are some of my notes on JVM multithreading and concurrency course on [udemy] in a Q & A format. The course is a good start for people who want to write high-performance code on the JVM and are trying to wrap their heads around the available concurrency primitives.

_PS: Feel free to correct and add more details by opening a PR on [github]._

[github]: https://github.com/tusharmath/tusharmath.github.io
[udemy]: https://dream11.udemy.com/course/java-multithreading-concurrency-performance-optimizatio

- [Thread Coordination](#thread-coordination)
- [Locks and Race Conditions](#locks-and-race-conditions)
- [Data Race](#data-race)
- [Thread Interruption](#thread-interruption)
- [Synchronization](#synchronization)
- [Reentrant Lock](#reentrant-lock)
- [Semaphore](#semaphore)
- [Condition Variable](#condition-variable)
- [Lock Free Concurrency](#lock-free-concurrency)

# Thread Coordination

**What kind of data requires thread coordination on the JVM?**

Anything that is stored on the heap will require coordination between threads. Objects, Class Members, Static Variables are all allocated on the Heap and require thread coordination.

**What kind of data does not require thread coordination?**

Each thread creates it's own stack and the content of the stack (local variables and references) aren't shared with other threads and hence don't require any coordination.

# Locks and Race Conditions

**What is a deadlock?**

When two threads hold on and wait for other to release a common resource, it can cause a deadlock. For eg: Let’s say there are two threads T1, T2 and two resources A, B, the possible condition could happen

```scala
T1 -> attemptLock(A) // Success
T2 -> attemptLock(B) // Success
T1 -> attemptLock(B) // Wait for T2 to release
T2 -> attemptLock(A) // Wait for T1 to release
```

This can trigger a deadlock in the application because T1 is waiting on T2 and T2 is waiting on T1.

**What is a _race condition_ ?**

A condition where multiple threads are accessing the same resource, and at least one of those threads is modifying the resource and timing of the threads can cause incorrect results.

**What operations are Atomic and don’t require synchronization between threads on the JVM?**

1. All reference assignment are atomic.
2. All assignments to primitive types - Int, Short, Byte, Float, Char, Bool **except** for Long and Double. Long and Double are 64 bit long (even on a 64bit computer) it requires JVM to produce two instructions one to write on each 32 bit.

# Data Race

**What is _data race_ ?**

Sometimes the compiler rearranges instructions internally for better performance viz. branch prediction, parallel execution ([SIMD]), pre-fetching instructions and better cache performance. This can produce a condition where sometimes the final output of the program changes. The condition is observed far more rarely than race conditions.

[simd]: https://en.wikipedia.org/wiki/Single_instruction,_multiple_data

**What is a _volatile_ variable?**

Volatile is used as an annotation to a variable to **make its assignment atomic**. Since most assignments are already atomic, we typically use volatile with Long and Double type primitives.

Making the variable volatile, ensures that the variable is read from the main memory. This guarantees visibility and ordering when used in a multi threaded env.

**How can I prevent reordering of instructions?**

One can use the `volatile` keyword in front of the variable to restrict the JVM from doing any kind of re-ordering optimisations.

# Thread Interruption

**How can threads be interrupted?**

Threads can be interrupted by calling the _interrupt_ method on the thread.

**What does the _interrupt_ method do?**

Internally it just sets the interrupt flag to true. One can check the status of the flag by calling `Thread.interrupted`

**How does _interrupt_ work in threads?**

Calling `thread.interrupt` internally just sets a flag. It’s up to the implementor to check and act on it. Sometimes, for eg: in `Thread.sleep` and other implementations will throw an exception `InterruptedException` when the interrupt flag is set.

**What is a _Daemon_ Thread?**

Setting the flag to daemon, means forcefully kill the thread from whatever it’s doing on interrupt. JVM will not wait for daemon threads to complete before it exits. It will kill them as soon as the main thread is completed.

**Can I call _setDaemon_ after the thread has started?**

No, changing the state of the thread to _daemon_ once started will trigger a `IllegalThreadStateException` exception.

**Can I check in a daemon thread if it’s interrupted?**

No, there is no way to check if the daemon thread was interrupted. The JVM will immediately kill the thread.

# Synchronization

**Why do we pass an object to the _synchronize_ block?**

```scala
val ref = new Object()
synchronize (ref) {
   count += count
}
```

The block that needs to be synchronized, basically needs some context within which it’s synchronized. If two threads are synchronized using the same ref, the two threads will never run that piece of code in parallel. However, if the threads are using two different refs the blocks won’t be synchronized.

# Reentrant Lock

**What is a Reentrant Lock?**

Reentrant lock is another alternative to synchronize between threads. It provides more fine grained control over the `synchronized` method that’s available on objects.

**How is a Reentrant Lock different from the synchronized block?**

1. Reentrant lock needs to be initialized using the `new` keyword like — `new ReentrantLock()`
1. It provides additional methods such as `getQueuedThreads` `getOwner` `isHeldByCurrentThread` `isLocked`
1. Locking and unlocking is done by calling the `lock` and `unlock` methods respectively.

**How can you guarantee fairness in ReentrantLock?**

We can guarantee fairness at the cost of performance by passing `true` to `ReentrantLock` eg: `new ReentrantLock(true)`. This flag ensures that all threads get an equal chance at acquiring a lock on the block of code.

**Can you lock interruptibly using a reentrant lock?**

Yes, it supports `lockInterruptibly` method which can be interrupted from outside. Using `lock` directly can block forever sometimes. Locking interruptibly ensures that it can be cancelled by calling interrupt on the thread.

**What is _tryLock_ ?**

`tryLock` allows a reentrant lock to immediately return with a `false` if the lock is not acquired. It can also take in a **timeout** incase one doesn’t want to immediately exit if the lock can not be achieved. You can use it as follows —

```scala
val lock = new ReentrantLock()
// instead of calling `lock` call `tryLock`

lock.tryLock
```

**What is a _ReentrantReadWriteLock_ ?**

It’s a specialized version of the `ReentrantLock` that isolates “reads” from “writes”. This gives better performance because multi reads don’t require locks between them. Reads can be done in parallel and it’s only when you are doing a write you would attain a lock on the object.

# Semaphore

**What is a Semaphore?**

Semaphores are high level concurrency primitives that are designed to control how many consumers and producers are allowed to access a resource at any given time.

**How is a Semaphore different than a Reentrant Lock?**

Semaphore acquire and release works differently than lock and unlock of ReentrantLock. Semaphores work on permits. The initial number of permits can be passed on to the constructor.

Each call to acquire reduces the permit by one until each reaches zero. After which further calls to acquire block on the thread until a release is called.
You can call release as many number of times as you want and with each release, the permit count increases.
Also, a release can be called by any thread and not just the thread that called the lock method.

# Condition Variable

**What is a _conditionVariable_ ?**

ConditionVariables are useful for implementing polling mechanisms on a variable which is updated by multiple threads. You can create a new condition variable on a ReentrantLock by calling `newCondition` on it.

```scala
val lock = new ReentrantLock()
val cond = lock.newCondition()
var count = 10


// a thread that waits until the count becomes zero
val thread1 = new Thread(() => {
   

   while(true) {
      lock.lock
      if (count == 0) {
         println("Count is zero")
         lock.unlock
         true
      } else {
         cond.await
      }
   }

})

// a thread that keeps reducing the count


```

**How does _await_ work in a condition variable?**

The await method on a condition variable will immediately perform two operations —

1.  Release the acquired lock on the reentrant variable.
2.  Put the thread in a “wait” state and block it exactly on that statement.

It will only resume once another thread calls `signal` on the condition variable AND performs an unlock immediately following it.

The `await` statement will not resume unless it’s signaled and it can regain the released lock.

**What happens if you call _await_ without calling _lock_ and why?**

It will result in an invalid state, throwing an exception — `IllegalMonitorStateException`

**What happens if you call _signal_ without calling _unlock_ and why?**

Nothing is going to happen. The signal will be sent to the waiting thread however, unless the lock is released, the waiting thread will not continue. Unlock will required to be called after the signal is sent.

**What happens when I call _signal_ after _unlock_ ?**

It will result in an invalid state, throwing an exception — `IllegalMonitorStateException`.

**How can I use an object as a condition variable?**

Every object supports `wait` `notify` and `notifyAll` methods which are a proxy to the reentrant lock’s condition variables. The `lock` and `unlock` doesn’t exist on object, so one needs to use the `synchronized` block to achieve that.

# Lock Free Concurrency

**How can one implement lock free concurrency?**

Lock free concurrency can be implemented using the `compareAndSet` operator that’s available under `AtomicReference`, `AtomicInteger` and other Atomic data structures under `java.util.concurrent.atomic` .

It takes in two parameters viz. `expectedValue` and `newValue`. Internally it checks if the internal value is the same as the provided expected value only then should we set it to the newValue. The expected value can be accessed via `ref.get`

```scala
compareAndSet(expectedValue, newValue);
```
