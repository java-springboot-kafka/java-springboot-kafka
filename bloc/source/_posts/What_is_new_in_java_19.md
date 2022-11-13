---
title: What's new in Java 19
---

![java19](https://s3.eu-west-1.amazonaws.com/redsys-prod/issues/f7c87d3e8cece7c283301b68/images/cover_x_JM11_22_cover@2x_1663691319585.jpg)

----------
Java 19 was released on September 20, 2022. You can download it [here .](https://jdk.java.net/19/)

The most exciting innovation for me are the [virtual threads , which have been developed as part of](https://www.happycoders.eu/de/java/java-19-features/#Virtual_Threads_Preview) [Project Loom](https://openjdk.org/projects/loom/) for several years and are now finally included in the JDK as a preview.

Virtual threads are a requirement for [Structured Concurrency](https://www.happycoders.eu/de/java/java-19-features/#Structured_Concurrency_Incubator) , another exciting new Incubator feature in Java 19.

There is also good news for anyone who wants to access non-Java code (e.g. the C standard library): The [Foreign Function & Memory API](https://www.happycoders.eu/de/java/java-19-features/#Foreign_Function_Memory_API_Preview) has now also reached the preview stage after five incubator rounds .

As always, I use the English names of the JEPs.

## New System Properties for System.out and System.err

There's one change that every Java developer should know about that isn't found in the [Java 19 Feature Announcements](https://openjdk.org/projects/jdk/19/) , but rather buried deep in the [release notes .](https://jdk.java.net/19/release-notes#JDK-8283620)

If you run an existing application with Java 19, it can happen that you only see question marks on the console instead of umlauts and other special characters.

This is due to the fact that from Java 19 the standard encoding of the operating system is used for the output - under Windows `System.out`e.g. `System.err`"Cp1252". To switch the output to UTF-8, you need to add the following VM options when calling the application:

`-Dstdout.encoding=utf8 -Dstderr.encoding=utf8`

If you don't want to do this every time you start the program, you can also set these settings globally by defining the following environment variable (yes, it starts with an underscore):

`_JAVA_OPTIONS="-Dstdout.encoding=utf8 -Dstderr.encoding=utf8"`

## New Methods to Create Preallocated HashMaps

Traditionally, if we want to create `ArrayList`a number of elements that is known in advance (e.g. 120), then we can do it like this:

```java
List<String> list = new ArrayList<>(120);
```

As a result, the `ArrayList`underlying array is allocated directly for 120 elements and does not have to be enlarged several times (i.e. newly created and copied) in order to insert the 120 elements.

Likewise, since time immemorial, we can `HashMap`generate one like this:

```java
Map<String, Integer> map = new HashMap<>(120);
```

Intuitively one would think that there is `HashMap`room for 120 mappings.

However, this is not the case!

Because `HashMap`it is initialized with a default load factor of 0.75. This means: As soon `HashMap`as it is 75% full, it will be rebuilt with twice the size ("rehashed"). This is to ensure that the elements `HashMap`are distributed as evenly as possible across the buckets and that no bucket contains more than one element.

So the one initialized with a capacity of 120 `HashMap`can only hold 120 × 0.75 = 90 mappings.

Previously, to create one `HashMap`for 120 mappings, you had to calculate the capacity yourself by dividing the number of mappings by the load factor: 120 ÷ 0.75 = 160.

So one `HashMap`for 120 mappings had to be created as follows:

```java
// for 120 mappings: 120 / 0.75 = 160
Map<String, Integer> map = new HashMap<>(160);
```

Java 19 makes this easier for us - we can now write the following instead:

```java
Map<String, Integer> map = HashMap.newHashMap(120);
```

If we look at the source code of the new methods, we see that the end result is the same as what we did manually before:

```java
public static <K, V> HashMap<K, V> newHashMap(int numMappings) {
    return new HashMap<>(calculateHashMapCapacity(numMappings));
}

static final float DEFAULT_LOAD_FACTOR = 0.75f;

static int calculateHashMapCapacity(int numMappings) {
    return (int) Math.ceil(numMappings / (double) DEFAULT_LOAD_FACTOR);
}
```

The `newHashMap()`method has also been built into `LinkedHashMap`and `WeakHashMap`.

There is no JDK Enhancement Proposal for this extension.

## Preview and Incubator features

Java 19 gives us a total of six preview and incubator features, i.e. features that are not yet finished but can already be tested by the developer community. Community feedback is typically incorporated into the further development and completion of these features.

### Pattern Matching for switch (Third Preview)

Let's start with a feature that has already had two preview rounds. The "pattern matching for switch" first introduced in [Java 17](https://www.happycoders.eu/de/java/java-17-features-de/#Pattern_Matching_for_switch_Preview) allowed us to write code like the following:

```java
switch (obj) {
  case String s && s.length() > 5 -> System.out.println(s.toUpperCase());
  case String s                   -> System.out.println(s.toLowerCase());

  case Integer i                  -> System.out.println(i * i);

  default -> {}
}
```

We can use it to `switch`check within a statement whether an object is of a certain class and whether it has other properties (as in the example: longer than 5 characters).

In Java 19, the syntax of the so-called "Guarded Pattern" (in the example above " ") was changed with [JDK Enhancement Proposal 427 .](https://openjdk.org/jeps/427) The new keyword must now be used `String s && s.length() > 5`instead .`&&``when`

The above example is notated as follows in Java 19:

```java
switch (obj) {
  case String s when s.length() > 5 -> System.out.println(s.toUpperCase());
  case String s                     -> System.out.println(s.toLowerCase());

  case Integer i                    -> System.out.println(i * i);

  default -> {}
}
```

`when``case`is a so-called "contextual keyword" and therefore only has a meaning within a label. If you have variables or methods named "when" in your code, you don't need to change anything.

### Record Patterns (Preview)

Staying with the topic "Pattern Matching" we come to the "Record Patterns". If you are new to Records, I recommend reading the [Records in Java](https://www.happycoders.eu/de/java/java-records/) article first .

The best way to explain what a record pattern is is with an example. Let's assume we have defined the following record:

```java
public record Position(int x, int y) {}
```

We also have a `print()`method that can return any object, including positions:

```java
private void print(Object object) {
  if (object instanceof Position position) {
    System.out.println("object is a position, x = " + position.x() 
                                         + ", y = " + position.y());
  }
  // else ...
}
```

In case you stumble over the notation used - it was introduced in [Java 16](https://www.happycoders.eu/de/java/java-16-features-de/#Pattern_Matching_for_instanceof) as "Pattern Matching for instanceof".

#### Record pattern with instanceof

[As of Java 19, JDK Enhancement Proposal 405](https://openjdk.org/jeps/405) allows us to use a so-called "Record Pattern". With that, we can also write the code as follows:

```java
private void print(Object object) {
  if (object instanceof Position(int x, int y)) {
    System.out.println("object is a position, x = " + x + ", y = " + y);
  } 
  // else ...
}
```

Instead `Position position`of matching " " and accessing in the code `position`below, we now match " `Position(int x, int y)`" and can access and directly in the code `x`below `y`.

#### Record pattern with switch

[Since Java 17](https://www.happycoders.eu/de/java/java-17-features-de/#Pattern_Matching_for_switch_Preview) , we can also `switch`write the original example as a statement:

```java
private void print(Object object) {
  switch (object) {
    case Position position
        -> System.out.println("object is a position, x = " + position.x() 
                                                + ", y = " + position.y());
    // other cases ...
  }
}
```

Since Java 19 we can also use a record pattern in the switch statement:

```java
private void print(Object object) {
  switch (object) {
    case Position(int x, int y) 
        -> System.out.println("object is a position, x = " + x + ", y = " + y);

    // other cases ...
  }
}
```

#### Nested record patterns

It is also possible to match nested records - I would also like to demonstrate this with an example.

We first define a second record, `Path`, with a start and a target position:

```
public record Path(Position from, Position to) {}
```

Our `print()`method can now easily output all X and Y coordinates of the path using a record pattern:

```java
private void print(Object object) {
  if (object instanceof Path(Position(int x1, int y1), Position(int x2, int y2))) {
    System.out.println("object is a path, x1 = " + x1 + ", y1 = " + y1 
                                     + ", x2 = " + x2 + ", y2 = " + y2);
  }
  // else ...
}
```

Alternatively, we can also `switch`write this as a statement:

```java
private void print(Object object) {
  switch (object) {
    case Path(Position(int x1, int y1), Position(int x2, int y2))
        -> System.out.println("object is a path, x1 = " + x1 + ", y1 = " + y1 
                                            + ", x2 = " + x2 + ", y2 = " + y2);
    // other cases ...
  }
}
```

Record patterns offer us an elegant way of accessing the elements of a record after a type check.

### Virtual Threads (Preview)

For me, the most exciting innovation in Java 19 are "Virtual Threads". These have been developed in [Project Loom](https://openjdk.org/projects/loom/) for several years and so far could only be tested with a self-compiled JDK.

With [JDK Enhancement Proposal 425](https://openjdk.org/jeps/425) , virtual threads are finally making their way into the official JDK - directly in preview status, so that no major changes to the API are to be expected.

Why we need virtual threads, what they are, how they work, and how to use them is a must-read in the [main article on virtual threads](https://www.happycoders.eu/de/java/virtual-threads/) .

### Foreign Function & Memory API (Preview)

[Project Panama](https://openjdk.org/projects/panama/) has been working on a replacement for the cumbersome, error-prone and slow Java Native Interface (JNI) for a long time.

The "Foreign Memory Access API" and the "Foreign Linker API" were already introduced in [Java 14](https://www.happycoders.eu/de/java/java-14-features-de/#Foreign-Memory_Access_API_Incubator) and [Java 16 - both individually at the incubator stage.](https://www.happycoders.eu/de/java/java-16-features-de/#Foreign_Linker_API_Incubator_Foreign-Memory_Access_API_Third_Incubator) In [Java 17](https://www.happycoders.eu/de/java/java-17-features-de/#Foreign_Function_Memory_API_Incubator) , these APIs were merged into the "Foreign Function & Memory API" (FFM API), which remained in the incubator stage until [Java 18 .](https://www.happycoders.eu/de/java/java-18-features-de/#Foreign_Function_Memory_API_Second_Incubator)

In Java 19, the new API with [JDK Enhancement Proposal 424](https://openjdk.org/jeps/424) has finally reached the preview stage, which means that only small changes and bug fixes will be made. So it's time to introduce the new API here!

The Foreign Function & Memory API enables access to native memory (i.e. memory outside of the Java heap) and access to native code (e.g. C libraries) directly from Java.

I'll show you how this works with an example. However, I won't delve too deeply into the topic here, since most Java developers rarely if ever need to access native memory and code.

Here's a simple example that stores a string in off-heap memory and calls the C Standard Library's "strlen" function on it:

```java
public class FFMTest {
  public static void main(String[] args) throws Throwable {
    // 1. Get a lookup object for commonly used libraries
    SymbolLookup stdlib = Linker.nativeLinker().defaultLookup();

    // 2. Get a handle to the "strlen" function in the C standard library
    MethodHandle strlen = Linker.nativeLinker().downcallHandle(
        stdlib.lookup("strlen").orElseThrow(), 
        FunctionDescriptor.of(JAVA_LONG, ADDRESS));

    // 3. Convert Java String to C string and store it in off-heap memory
    MemorySegment str = implicitAllocator().allocateUtf8String("Happy Coding!");

    // 4. Invoke the foreign function
    long len = (long) strlen.invoke(str);

    System.out.println("len = " + len);
  }
}

```

The one in line 9 is interesting `FunctionDescriptor`: it defines the return type of the function as the first parameter and the arguments of the function as additional parameters. This `FunctionDescriptor`ensures that all Java types are properly converted to C types and vice versa.

Since the FFM API is still in the preview state, a few additional parameters must be specified to compile and start it:

```java
$ javac --enable-preview -source 19 FFMTest.java
$ java --enable-preview FFMTestCode language:  Plaintext  ( plaintext )
```

Anyone who has worked with JNI before - and remembers how much Java and C boilerplate code it took to write and keep in sync - will see that the overhead of calling the native function has been reduced by orders of magnitude.

If you want to delve deeper into the subject: in the [JEP](https://openjdk.org/jeps/424) you will find further, more complex examples.

### Structured Concurrency (Incubator)

If a task consists of different subtasks that can be completed in parallel (e.g. accessing data from a database, calling a remote API and loading a file), we were previously able to use the Java Executur Framework for this.

That could then e.g. look like this:

```java
private final ExecutorService executor = Executors.newCachedThreadPool();

public Invoice createInvoice(int orderId, int customerId, String language)
    throws ExecutionException, InterruptedException {
  Future<Order> orderFuture =
      executor.submit(() -> loadOrderFromOrderService(orderId));

  Future<Customer> customerFuture =
      executor.submit(() -> loadCustomerFromDatabase(customerId));

  Future<String> invoiceTemplateFuture =
      executor.submit(() -> loadInvoiceTemplateFromFile(language));

  Order order = orderFuture.get();
  Customer customer = customerFuture.get();
  String invoiceTemplate = invoiceTemplateFuture.get();

  return Invoice.generate(order, customer, invoiceTemplate);
}
```

We pass the three subtasks to the executor and wait for the subresults. The Happy Path is implemented quickly. But how do we handle exceptions?

-   If one subtask fails, how can we kill the others?
-   How can we cancel the subtasks when the entire calculation is no longer needed?

Both are possible, but require fairly complex, hard-to-maintain code.

And what if we want to debug code of this kind? A thread dump e.g. B. would give us tons of threads named "pool-X-thread-Y" - but we wouldn't know which pool thread belongs to which calling thread, since all calling threads share the executor's thread pool.

[JDK Enhancement Proposal 428](https://openjdk.org/jeps/428) introduces an API for so-called "structured concurrency", a concept intended to improve the implementation, readability and maintainability of code for requirements of this type.

Using a `StructuredTaskScope`we can rewrite the example as of Java 19 as follows:

```java
Invoice createInvoice(int orderId, int customerId, String language)
    throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<Order> orderFuture =
        scope.fork(() -> loadOrderFromOrderService(orderId));

    Future<Customer> customerFuture =
        scope.fork(() -> loadCustomerFromDatabase(customerId));

    Future<String> invoiceTemplateFuture =
        scope.fork(() -> loadInvoiceTemplateFromFile(language));

    scope.join();
    scope.throwIfFailed();

    Order order = orderFuture.resultNow();
    Customer customer = customerFuture.resultNow();
    String invoiceTemplate = invoiceTemplateFuture.resultNow();

    return new Invoice(order, customer, invoiceTemplate);
  }
}

```

So we replace the one in the scope of the class `ExecutorService`with one in the scope of the method `StructuredTaskScope`– and `executor.submit()`with `scope.fork()`.

With `scope.join()`we wait for all tasks to be completed - or at least one has failed or been canceled. In the latter two cases, the subsequent throws `throwIfFailed()`a `ExecutionException`or a `CancellationException`.

The new approach brings the following improvements over the old one:

-   Task and subtasks form a self-contained unit in the code - there is no `ExecutorService`one in a higher scope. The threads don't come from a thread pool; instead, each subtask runs in a new [virtual thread](https://www.happycoders.eu/de/java/virtual-threads/) .
-   As soon as an error occurs in one of the subtasks, all other subtasks are aborted.
-   If the calling thread is aborted, the subtasks are also aborted.
-   The thread dump shows the call hierarchy between the calling thread and the threads that execute the subtasks.

If you want to try the example yourself: Preview features must be explicitly activated and Incubator modules must be explicitly added to the module path. For example, if you saved the code in a file named `StructuredConcurrencyTest.java`, you can compile and run it like this:

```java
$ javac --enable-preview -source 19 --add-modules jdk.incubator.concurrent StructuredConcurrencyTest.java
$ java --enable-preview --add-modules jdk.incubator.concurrent StructuredConcurrencyTestCode language:  Plaintext  ( plaintext )
```

Please note that Incubator features are still subject to major changes.

### Vector API (Fourth Incubator)

The new Vector API has nothing `java.util.Vector`to do with the class. In fact, it's about a new API for mathematical vector calculations and their mapping to modern SIMD (single-instruction-multiple-data) CPUs.

The Vector API has been part of the JDK as an incubator since [Java 16 and was further developed in](https://www.happycoders.eu/de/java/java-16-features-de/#Vector_API_Incubator) [Java 17](https://www.happycoders.eu/de/java/java-17-features-de/#Vector_API_Second_Incubator) and [Java 18](https://www.happycoders.eu/de/java/java-18-features-de/#Vector_API_Third_Incubator) .

With [JDK Enhancement Proposal 426](https://openjdk.org/jeps/426) , Java 19 delivers the fourth iteration in which the API has been extended to include new vector operations - as well as the ability to store vectors in memory segments (a feature of the [Foreign Function & Memory API](https://www.happycoders.eu/de/java/java-19-features/#Foreign_Function_Memory_API_Preview) ) and read them from them.

Incubator features are still subject to significant changes, so I won't detail the API here. I'll catch up on that once the Vector API has transitioned to preview status.

## Deprecations and Deletion

In Java 19, some features were marked as "deprecated" or deprecated.

### Deprecation of Locale class constructors

In Java 19, the public constructors of the `Locale`class have been marked as "deprecated".

Instead, we should use the new static factory method `Locale.of()`. This ensures that there is `Locale`only one instance per configuration.

The following example shows the usage of the factory method versus the constructor:

```java
Locale german1 = new Locale("en"); // deprecated
Locale germany1 = new Locale("en", "EN"); // deprecated

Locale german2 = Locale.of("en");
Locale germany2 = Locale.of("en", "EN");

System.out.println("us1  == Locale.ENGLISH  = " + (us1 == Locale.ENGLISH));
System.out.println("us1 == Locale.GERMANY = " + (us1 == Locale.ENGLISH));
System.out.println("us2  == Locale.GERMAN  = " + (us2 == Locale.ENGLISH));
System.out.println("us2 == Locale.GERMANY = " + (us2 == Locale.ENGLISH));

```

If you start this code, you will see that the objects returned via the factory method are identical to the `Locale`constants - the ones created via the construct are obviously not.

### java.lang.ThreadGroup is degraded

In [Java 14](https://www.happycoders.eu/de/java/java-14-features-de/#Thread_SuspendResume_Are_Deprecated_for_Removal) and [Java 16](https://www.happycoders.eu/de/java/java-16-features-de/#Terminally_Deprecated_ThreadGroup_stop_destroy_isDestroyed_setDaemon_and_isDaemon) , several `Thread`and `ThreadGroup`methods were marked as "deprecated for removal". You can find out why in the linked sections.

The following of these methods have been deprecated in Java 19:

-   `ThreadGroup.destroy()`– the call to this method is ignored.
-   `ThreadGroup.isDestroyed()`– always `false`returns.
-   `ThreadGroup.setDaemon()`– sets the `daemon`flag, but this no longer has any effect.
-   `ThreadGroup.getDaemon()`\- returns the value of the unused `daemon`flag.
-   `ThreadGroup.suspend()`, `resume()`and `stop()`throw a `UnsupportedOperationException`.

## Other changes in Java 19

In this section you will find changes/enhancements that may not be relevant to all Java developers.

### Linux/RISC-V Port

Due to the increasing spread of RISC-V hardware, a port for the corresponding architecture was made available with [JEP 422 .](https://openjdk.org/jeps/422)

### Complete list of all changes in Java 19

In addition to the JDK Enhancement Proposals (JEPs) and class library changes presented in this article, there are numerous minor changes that are beyond the scope of this article. [See the JDK 19 Release Notes](https://jdk.java.net/19/release-notes) for a full list .

## Conclusion

[In Java 19, the long-awaited virtual threads](https://www.happycoders.eu/de/java/virtual-threads/) developed in Project Loom have finally found their way into the JDK (albeit initially in preview stage). I hope you're as excited as I am and can't wait to start using virtual threads in your projects!

Based on this, structured concurrency (still in the incubator stage) will significantly simplify the management of tasks that are divided into parallel subtasks.

Record patterns have been added to the pattern matching options in `instanceof`and that have been gradually developed in the last JDK versions .`switch`

The preview and incubator features "Pattern Matching for switch", "Foreign Function & Memory API" and "Vector API" have been sent to the next preview or incubator round.

By default, the output on the console is in the standard encoding of the operating system and may have to be changed using the VM option.

HashMaps offer new factory methods to create maps with enough capacity for a given number of mappings.

As always, various other changes round off the release. You can download Java 19 [here .](https://jdk.java.net/19/)
