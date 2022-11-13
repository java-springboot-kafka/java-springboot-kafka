---
title: Virtual Threads in Java 19
---

The Project Loom is an experimental version of the JDK. It extends Java with virtual threads that allow lightweight concurrency. Preview releases are already available and show what is possible.

Server-side Java applications should be able to process many requests in parallel. The model that is most common in server-side Java programming to date is thread-per-request. A thread is assigned to an incoming request and everything that needs to be done to respond with an appropriate response is processed on this thread. However, this severely limits the maximum number of requests that can be processed concurrently. Because the Java threads, which each claim a native operating system thread, are not lightweights. On the one hand because of the memory requirements: Each individual thread takes up more than one megabyte of memory by default. On the other hand because of the costs for switching between the threads, the context switch.

As a reaction to these disadvantages, many asynchronous libraries that use CompletableFutures, for example, but also entire "reactive" frameworks, such as e.g. B. RxJava \[1\], Reactor \[2\] or Akka Streams \[3\]. Although they all use the resources far more effectively, they require the developer to switch to a significantly different programming model. Some people moan about the cognitive ballast, as they prefer to sequentially list what the program is supposed to do instead of dealing with callbacks, observables or flows. This raises the question of whether it would not be possible to combine the advantages of both worlds: as effective as asynchronous programming and still being able to program in the usual, sequential command sequence. Oracle's Project Loom \[4\] wants to explore exactly this option with a modified JDK. On the one hand, it brings with it a new, lightweight construct for concurrency, namely the virtual threads, and on the other hand a customized standard library that is based on this.

## Virtual Threads

YOu remember multithreading in Java 1.1? At that time, Java only knew so-called green threads. The possibility of using multiple operating system threads was not used at all. Threads were only emulated within the JVM. As of Java 1.2, a native operating system thread was actually started for each Java thread.

Now the Green Threads are celebrating a revival. Although in a significantly changed, modernized form, virtual threads are basically nothing else: they are threads that are managed within the JVM. However, they are no longer a replacement for the native threads, but a supplement. A relatively small number of native threads is used as a carrier to process an almost arbitrarily large number of virtual threads. The overhead of the virtual threads is so low that the programmer doesn't have to worry about how many of them he starts.

In a 64-bit JVM with default settings, a native thread already reserves one megabyte for the stack (the thread stack size, which can also be set explicitly with the _\-Xss option)._ There is also some additional metadata. And if memory isn't the limit, the operating system will stop at a few thousand.

Listing 1

```
// Achtung, kann Rechner einfrieren...
void excessiveThreads(){
  ThreadFactory factory = Thread.builder().factory();
  ExecutorService executor = Executors.newFixedThreadPool(10000, factory);
  IntStream.range(0, 10000).forEach((num) -> {
    executor.submit(() -> {
      try {
          out.println(num);
          // Wir warten ein bisschen, damit die Threads wirklich alle gleichzeitig laufen
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
  // Factory für virtuelle Threads
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
  // Der scope ist ein Hilfsmittel, um geschachtelte Continuations zu   // ermöglichen.
  ContinuationScope scope = new ContinuationScope("demo");
  Continuation a = new Continuation(scope, () -> {
    out.print("To be");
    // hier wird die Funktion eingefroren und gibt die Kontrolle an den     // Aufrufer.
    Continuation.yield(scope);
    out.println("continued!");
  });
  a.run();
  out.print(" ... ");
  // die Continuation kann von dort, wo sie angehalten wurde, fortgesetzt   // werden.
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

## When are virtual threads coming for everyone?

Project Loom keeps a low profile when it comes to the question of which Java release the features should be included in. At the moment everything is still experimental and APIs are subject to change. In JDK 15 you probably shouldn't expect that yet.

If you want to try it out, you can either check out the source code from GitHub \[7\] and build the JDK yourself, or download ready-made preview releases \[8\].

![huehnken_lutz_sw.tif_fmt1.jpg](https://s3.eu-west-1.amazonaws.com/redsys-prod/articles/7049438b39cdcf98a347c687/images/huehnken_lutz_sw.tif_fmt1.jpg)Lutz Hühnken is Chief Solutions Architect at the Hamburg Süd shipping company. He is currently primarily involved with events-first microservices, domain-driven design, event storming and reactive systems.

[![Twitter](https://s3.eu-west-1.amazonaws.com/redsys-prod/articles/7049438b39cdcf98a347c687/images/SoMe-Twitter.png)](https://twitter.com/lutzhuehnken)