---
title: Virtual Threads in Java 19
---

The Project Loom is an experimental version of the JDK. It extends Java with virtual threads that allow lightweight concurrency. Preview releases are already available and show what is possible.

Server-side Java applications should be able to process many requests in parallel. The model that is most common in server-side Java programming to date is thread-per-request. A thread is assigned to an incoming request and everything that needs to be done to respond with an appropriate response is processed on this thread. However, this severely limits the maximum number of requests that can be processed concurrently. Because the Java threads, which each claim a native operating system thread, are not lightweights. On the one hand because of the memory requirements: Each individual thread takes up more than one megabyte of memory by default. On the other hand because of the costs for switching between the threads, the context switch.

As a reaction to these disadvantages, many asynchronous libraries that use CompletableFutures, for example, but also entire "reactive" frameworks, such as e.g. B. RxJava \[1\], Reactor \[2\] or Akka Streams \[3\]. Although they all use the resources far more effectively, they require the developer to switch to a significantly different programming model. Some people moan about the cognitive ballast, as they prefer to sequentially list what the program is supposed to do instead of dealing with callbacks, observables or flows. This raises the question of whether it would not be possible to combine the advantages of both worlds: as effective as asynchronous programming and still being able to program in the usual, sequential command sequence. Oracle's Project Loom \[4\] wants to explore exactly this option with a modified JDK. On the one hand, it brings with it a new, lightweight construct for concurrency, namely the virtual threads, and on the other hand a customized standard library that is based on this.

## Virtual Threads

You remember multithreading in Java 1.1? At that time, Java only knew so-called green threads. The possibility of using multiple operating system threads was not used at all. Threads were only emulated within the JVM. As of Java 1.2, a native operating system thread was actually started for each Java thread.

Now the Green Threads are celebrating a revival. Although in a significantly changed, modernized form, virtual threads are basically nothing else: they are threads that are managed within the JVM. However, they are no longer a replacement for the native threads, but a supplement. A relatively small number of native threads is used as a carrier to process an almost arbitrarily large number of virtual threads. The overhead of the virtual threads is so low that the programmer doesn't have to worry about how many of them he starts.

In a 64-bit JVM with default settings, a native thread already reserves one megabyte for the stack (the thread stack size, which can also be set explicitly with the _\-Xss option)._ There is also some additional metadata. And if memory isn't the limit, the operating system will stop at a few thousand.

Listing 1

```
void excessiveThreads(){
  ThreadFactory factory = Thread.builder().factory();
  ExecutorService executor = Executors.newFixedThreadPool(10000, factory);
  IntStream.range(0, 10000).forEach((num) -> {
    executor.submit(() -> {
      try {
          out.println(num);
          Thread.sleep(10000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      });
    });
  executor.shutdown();
  executor.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
}
```

The attempt in Listing 1 to start 10,000 threads will bring most computers to their knees (or the JVM to crash). Warning: the program may reach the thread limit of your operating system and your computer could freeze as a result.

With virtual threads, however, it is no problem to start a whole million threads. Listing 2 runs without problems on the Project Loom JVM!

Listing 2

```
void virtualThreads(){
  // Factory for virtual Threads
  ThreadFactory factory = Thread.builder().virtual().factory();
  ExecutorService executor = Executors.newFixedThreadPool(1000000, factory);
  IntStream.range(0, 1000000).forEach((num) -> {
    executor.submit(() -> {
      try {
        out.println(num);
        // Thread.sleep schickt hier nur den virtuellen Thread schlafen
        Thread.sleep(10000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    });
  });
  executor.shutdown();
  executor.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
}
```

## JDK APIs

So we could start a million threads at the same time. This may be a nice effect, but its usefulness is still limited. It gets really interesting when all these virtual threads only use the CPU for a short time. Most server-side applications are less CPU-heavy and more I/O-heavy. There might be a bit of input validation, but then mostly data is fetched (or written) over the network, for example from the database, or over HTTP from another service.

In the thread-per-request model with synchronous I/O, this results in the thread being blocked for the duration of the I/O operation. The operating system recognizes that the thread is waiting for I/O, and the scheduler switches directly to the next one. That doesn't seem too bad at first, since the blocked thread doesn't use the CPU. However, each switch between threads introduces an overhead. Incidentally, this effect has been exacerbated by modern, complex CPU architectures with multiple cache layers (non-uniform memory access, NUMA).

The number of context switches should be minimized to actually utilize the CPU effectively. From a CPU point of view, it would be perfect if exactly one thread ran continuously on each core and was never swapped out. We will not usually be able to achieve this state, after all other processes are running on the server in addition to the JVM. But it remains to be said: "A lot helps a lot" does not apply, at least for native threads - you can definitely overdo it here.

In order to be able to execute many parallel requests with few native threads, the virtual thread introduced in Project Loom voluntarily relinquishes control when waiting for I/O and pausing. However, this does not block the underlying native thread, which executes the virtual thread as a worker. Rather, the virtual thread signals that it can't do anything right now, while the native thread can grab the next virtual thread without CPU context switching. But how do you do that without using asynchronous I/O APIs? After all, Project Loom has made it its mission to protect programmers from the callback desert.

This shows the advantage of providing the new functionality in the form of a new JDK version. A third-party library for a currently current JDK relies on using an asynchronous programming model. Instead, Project Loom comes with a customized standard library. Many I/O libraries have been rewritten so that they now use virtual threads internally (box: "Changed standard libraries"). A usual network call is - without any change in the program code - suddenly no longer blocking I/O. Only the virtual thread is paused. With this trick, existing programs also benefit from the virtual threads without the need for adjustments.

## Changed standard libraries

The following classes have been adjusted so that blocking calls in them no longer block the native thread, only the virtual thread.

-   java.net.Socket

-   java.net.ServerSocket

-   java.net.DatagramSocket/MulticastSocket

-   java.nio.channels.SocketChannel

-   java.nio.channels.ServerSocketChannel

-   java.nio.channels.DatagramChannel

-   java.nio.channels.Pipe.SourceChannel

-   java.nio.channels.Pipe.SinkChannel

-   java.net.InetAddress


## Continuations

The concept that forms the basis for the implementation of the virtual threads is called Delimited Continuations. Most of you will probably have used a debugger at some point. To do this, set a breakpoint in the code. When this point is reached, execution is halted and the current state of the program is displayed in the debugger. It would now be conceivable to freeze this state. This is the basic idea of continuation: stop at a point in the flow, take the state (of the current thread, i.e. the call stack, the current position in the code, etc.) and convert it into a function that "do-there- continue-where-you-left-off function”. This can then be called up at a later point in time and the process started can be resumed. Exactly what is needed for the virtual threads:

Continuations also have their place outside of virtual threads and are a powerful construct for influencing the flow of a program at will. Project Loom provides an API for working with continuations. For application development it should not be necessary to work with it directly. It is primarily the low-level construct that makes virtual threads possible. However, if you want to experiment with it, you can do so (Listing 3).

Listing 3

```
void continuationDemo() {
  ContinuationScope scope = new ContinuationScope("demo");
  Continuation a = new Continuation(scope, () -> {
    out.print("To be");
    // This is where the function freezes and gives control to the // caller.
    Continuation.yield(scope);
    out.println("continued!");
  });
  a.run();
  out.print(" ... ");
  // the continuation can be resumed from where // it stopped.
  a.run();
  // ...
  }
```

By the way, virtual threads are a form of cooperative multitasking. Native threads are stripped of CPU by the operating system regardless of what they are doing (preemptive multitasking). Even an endless loop will not block the CPU core, but others will still have their turn. However, at the virtual thread level, there is no such scheduler - the virtual thread itself must return control to the native thread.

## Not yet in the package: tail-call elimination

For the sake of completeness, it should be mentioned that among the features to be implemented in Project Loom is the optimization of tail recursion. When a recursive function calls itself as the last action, the compiler can turn it into a nonrecursive loop. This is already happening in some languages other than Java. On the JVM, Scala, Kotlin (with _tailrec_ ) or Clojure (with _recur_ ) support this tail call elimination.

Project Loom also wants to introduce a directive that directs the compiler to this optimization, which makes the use of many recursive algorithms possible in the first place. However, this is not yet included in the current previews. It's not even specified yet what it should look like.

## Beyond virtual threads

The problems with threads described at the beginning relate solely to efficiency. A completely different challenge has not yet been considered: communication between threads. Programming with the current Java mechanisms is not very easy and is therefore error-prone. Threads communicate via shared variables (shared mutable state). In order to avoid race conditions, these must be replaced by _synchronized_or explicit locks are protected. If errors occur here, they are particularly difficult to find due to the non-determinism at runtime. And even when done right, these locks often represent a point of contention, a bottleneck in execution. Because potentially many will then have to wait for exactly the one who is currently using the lock.

There are certainly alternative models. In the context of virtual threads, channels should be mentioned here in particular. Kotlin and Clojure (box: “What are the others doing?”) offer this as the preferred communication model for their coroutines. Instead of a shared, changeable state, they rely on immutable messages that are written (preferably asynchronously) into a channel and from there picked up by the receiver. However, it is still open whether channels will become part of Project Loom.

## What are the others doing?

The virtual threads may be new to Java, but they are not new to the JVM. If you know Clojure or Kotlin, you probably feel reminded of Coroutines. In fact, they are technically very similar to these and solve the same problem. However, there is at least one small but interesting difference from a developer's perspective: there are special keywords for coroutines in the respective languages. In Clojure, a macro for a Go block, in Kotlin, the suspend keyword.

The virtual threads in Loom do not require any additional syntax. The same method can be run unmodified from a virtual thread, or directly from a native thread.

However, it may not be necessary for Project Loom to solve all problems - any gaps will surely be filled by new third-party libraries that offer solutions at a higher level of abstraction based on the virtual threads. The Fibry experiment \[5\], for example, is an actuator library for Loom (box: “Wasn't there something about fibers?”).

## Wasn't there something about Fibers?

Anyone who has heard of Project Loom before knows the term fibers. Together with the thread (thread), this probably also led to the project name, because Loom means loom. In the early versions of Project Loom, Fiber was the name for the virtual thread. It goes back to a previous project of current Loom project leader Ron Pressler, the Quasar Fibers \[6\]. However, the name Fiber, like the alternative coroutine, was discarded at the end of 2019, and Virtual Thread prevailed.

## The unique selling proposition of Project Loom

Basically, there is already an established solution for the problem that Project Loom solves: Asynchronous I/O, either through callbacks or through "reactive" frameworks. However, using these means adopting a different programming model. Not all developers find it easy to switch to an asynchronous mindset. There is also a partial lack of support in common libraries - everything that stores data in ThreadLocal is suddenly unusable. And in tooling: Debugging asynchronous code often results in several aha moments. Also in the sense that the code to be examined is not executed on the thread that you are currently single-stepping through with the debugger.

The special appeal of Project Loom is that it makes the changes at the JDK level, so the program code can remain unchanged. A currently inefficient program that consumes a native thread for each HTTP connection could run unchanged on the Project Loom JDK and would suddenly be efficient and scalable (Box: "When will virtual threads be available for everyone?"). Thanks to the modified java.net library, now based on virtual threads.

## Virtual threads released in Java 19

Recently, JDK 19 was released and introduced several new features, one of which is worth paying attention to is the addition of virtual threads.

Many people may be confused, what is a virtual thread, and what is the difference between it and the platform thread we are using now?

To clarify the virtual threads in JDK 19, we must first understand how threads are implemented.

## How the thread is implemented

We all know that in the operating system, a thread is a lighter-weight scheduling execution unit than a process. The introduction of a thread can separate the resource allocation and execution scheduling of a process. Each thread can share process resources and schedule independently.

In fact, **there are three main ways to implement threads: using kernel threads, using user threads, and using user threads and lightweight processes.**

### Implemented using kernel threads

**A kernel thread (Kernel-Level Thread, KLT) is a thread directly supported by the operating system kernel (Kernel, hereinafter referred to as the kernel). It is responsible for mapping the tasks of the thread to each processor, and provides an API interface to the application to manage the thread.**

Applications generally do not use kernel threads directly, but use a high-level interface of kernel threads—Light Weight Process (LWP). A lightweight process is a thread in the usual sense. Since each lightweight process is backed by a kernel thread, there can only be a lightweight process if kernel threads are supported first.

With the support of kernel threads, each lightweight process becomes an independent scheduling unit. Even if a lightweight process is blocked in a system call, it will not affect the entire process to continue working.

But lightweight processes have their limitations: First, because they are implemented based on kernel threads, various thread operations, such as creation, destruction, and synchronization, require system calls. The cost of system calls is relatively high, and it needs to switch back and forth between User Mode and Kernel Mode. Secondly, each lightweight process needs to have the support of a kernel thread, so the lightweight process consumes a certain amount of kernel resources (such as the stack space of the kernel thread), so the number of lightweight processes supported by a system is limited .

### Implemented using user threads

Establish a thread library in user space, and complete thread management through the Run-time System. Because the implementation of this thread is in user space, the kernel of the operating system does not know the existence of threads, so the kernel manages It is still a process, so this thread switching does not require kernel operation.

In this implementation, the relationship between a process and a thread is one-to-many.

The advantage of this thread implementation is that the thread switching is fast, and it can run on any operating system, just need to implement the thread library. **However, the disadvantage is also obvious, that is, the operation of all threads needs to be handled by the user program itself, and because most system calls are blocked, once a process is blocked, all threads in the process will also be blocked. There is also a big problem in how to map threads to other processors in a multiprocessor system.**

### Mixed implementation using user threads and lightweight processes

There is also a mixed implementation method, that is, the creation of threads is done in user space and performed through the thread library, but the scheduling of threads is done by the kernel. Multiple user threads multiplex multiple kernel threads through multiplexing. This will not be discussed further.

## Java thread implementation

The above are three ways to implement threads in operating systems. Different operating systems use different mechanisms when implementing threads. For example, Windows uses kernel threads to implement them, while Solaris implements them through mixed modes.

As a cross-platform programming language, Java actually depends on the specific operating system for its thread implementation. The more commonly used windows and linux are implemented in the way of kernel threads.

That is to say, when we create a Tread in JAVA code, it actually needs to be mapped to the specific implementation of the thread of the operating system, because the common method implemented by kernel threads requires the kernel to participate in the creation and scheduling. Therefore, the cost is relatively high. Although JAVA provides a thread pool method to avoid repeated creation of threads, there is still a lot of room for optimization. **Moreover, this implementation means that the number of platform threads is limited due to the impact of machine resources.**

## virtual thread

The virtual thread introduced by JDK 19 is a lightweight thread implemented by JDK, which can avoid the extra cost caused by context switching. \*\*His implementation principle is actually that JDK is no longer a thread that corresponds to an operating system one-to-one for each thread, but maps multiple virtual threads to a small number of operating system threads, which can be avoided through effective scheduling. Those context switches.

![virtual-threads](source/images/virtual-threads.png)

Also, we can create a very large number of virtual threads in the application, independent of the number of platform threads. These virtual threads are managed by the JVM, so they don't add extra context switching overhead since they are stored in RAM as normal Java objects.

## The difference between virtual threads and platform threads

First, virtual threads are always daemon threads. The setDaemon(false) method cannot change a virtual thread to a non-daemon thread. **So, it is important to note that the JVM will terminate when all started non-daemon threads are terminated. This means that the JVM will not wait for the virtual thread to finish before exiting.**

Second, even with the setPriority() method, **virtual threads always have normal priority** and cannot be changed. Calling this method on a virtual thread has no effect.

Also, **virtual threads do not support methods such as stop(), suspend() or resume()** . These methods throw UnsupportedOperationException when called on a virtual thread.

## how to use

Next, let's introduce how to use virtual threads in JDK 19.

First, a virtual thread can be run via Thread.startVirtualThread():

```
Thread.startVirtualThread(() -> {
    System.out.println("Hello...");
});
```

Secondly, virtual threads can also be created through Thread.Builder. The Thread class provides ofPlatform() to create a platform thread and ofVirtual() to create a virtual scene.

```
Thread.Builder platformBuilder = Thread.ofPlatform().name("platform thread");
Thread.Builder virtualBuilder = Thread.ofVirtual().name("virtual thread");

Thread t1 = platformBuilder .start(() -> {...}); 
Thread t2 = virtualBuilder.start(() -> {...}); 

```

In addition, the thread pool also supports virtual threads. You can create virtual threads through Executors.newVirtualThreadPerTaskExecutor():

```
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}

```

However, **in fact, it is not recommended to use virtual threads and thread pools together** , because the Java thread pool is designed to avoid the overhead of creating new operating system threads, but the overhead of creating virtual threads is not large, so there is no need to put them in the thread pool. middle.

## performance difference

After talking for a long time, can virtual threads improve performance and how much? Let's do a test.

Let's write a simple task that waits 1 second before printing a message in the console:

```
final AtomicInteger atomicInteger = new AtomicInteger();

Runnable runnable = () -> {
  try {
    Thread.sleep(Duration.ofSeconds(1));
  } catch(Exception e) {
      System.out.println(e);
  }
  System.out.println("Work Done - " + atomicInteger.incrementAndGet());
};

```

Now, we will create 10,000 threads from this Runnable and execute them using virtual threads and platform threads to compare the performance of both.

Let's start with the implementation of platform threads that we are more familiar with:

```
Instant start = Instant.now();

try (var executor = Executors.newFixedThreadPool(100)) {
  for(int i = 0; i < 10_000; i++) {
    executor.submit(runnable);
  }
}

Instant finish = Instant.now();
long timeElapsed = Duration.between(start, finish).toMillis();  
System.out.println("总耗时 : " + timeElapsed); 

```

The output is:

```
total time : 102323

```

The total time is about 100 seconds. Next, run it with a virtual thread and see

> Because in JDK 19, virtual threads are a preview API and are disabled by default. So you need to use $ java -- source 19 -- enable-preview xx.java to run the code.

```
Instant start = Instant.now();

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
  for(int i = 0; i < 10_000; i++) {
    executor.submit(runnable);
  }
}

Instant finish = Instant.now();
long timeElapsed = Duration.between(start, finish).toMillis();  
System.out.println("total time : " + timeElapsed); 

```

Use Executors.newVirtualThreadPerTaskExecutor() to create virtual threads. The execution results are as follows:

```
total time : 1674

```

The total time is about 1.6 seconds!

The difference between 100 seconds and 1.6 seconds is enough to see that the performance improvement of virtual threads is immediate.

## Summarize

This article introduces the new virtual thread, or coroutine, introduced in JDK 19, mainly to solve the problem that threads in the reading operating system need to rely on the implementation of kernel threads, resulting in a lot of extra overhead. By introducing virtual threads at the Java language level, scheduling management is performed through the JVM, thereby reducing the cost of context switching.

At the same time, after a simple demo test, we found that the execution of virtual threads is indeed much more efficient. But you also need to pay attention when using it, the virtual thread is a daemon thread, so it may shut down the virtual machine before it finishes executing.


