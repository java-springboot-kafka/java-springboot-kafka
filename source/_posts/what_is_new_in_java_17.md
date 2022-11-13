---
title: What's new in Java 17
---

![java17](https://res.infoq.com/articles/why-how-upgrade-java17/en/headerimage/upgrade-to-java-16-or-17-header-image-1630333858357.jpg)

On September 14, 2021 the time had finally come: After the five "intermediate versions" Java 12 to 15, each of which was only maintained for half a year, the current long-term support (LTS) release, Java 17, was released.

Oracle will provide free upgrades for Java 17 for at least five years, i.e. until September 2026 - and extended, paid support until September 2029.

In Java 17, 14 JDK enhancement proposals were implemented. I sorted the changes by relevance to day-to-day programming work. The article begins with language enhancements and changes to the module system. Various extensions of the JDK class library follow, performance improvements, new preview and incubator features, deprecations and deletions and finally other changes that are rarely encountered in day-to-day work.

As always, I have used the English names of the JEPs and other changes. A translation into German would not bring any added value here.

## Sealed Classes

The big innovation in Java 17 (besides long-term support) are sealed classes and interfaces.

What sealed classes are, exactly how they work and why we need them, I will explain in a separate article due to the scope of the topic: [Sealed Classes in Java](https://www.java-springboot-kafka.github.io/de/java/sealed-classes-de/)

_(Sealed classes were first introduced in [Java 15](https://www.java-springboot-kafka.github.io/de/java/java-15-features-de/#Sealed_Classes_Preview) as a preview feature. Three small changes were released in [Java 16.](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Sealed_Classes_Second_Preview) [JDK Enhancement Proposal 409](https://openjdk.org/jeps/409) declares sealed classes production-ready in Java 17 with no further changes.)_

## Strongly Encapsulate JDK Internals

The module system ( [Project Jigsaw](https://openjdk.org/projects/jigsaw/) ) was introduced in Java 9, in particular to be able to better modularize code and to increase the security of the Java platform.

### Before Java 16: Relaxed Strong Encapsulation

Up until Java 16, this had little impact on existing code, as the JDK developers provided "Relaxed Strong Encapsulation" mode for a transitional period.

This allowed non-public classes and methods of packages in the JDK class library that existed before Java 9 to be accessed via deep reflection without changing the configuration.

The following example extracts the bytes of a string by reading its private field `value`:

```
public class EncapsulationTest {
  public static void main(String[] args) throws ReflectiveOperationException {
    byte[] value = getValue("Happy Coding!");
    System.out.println(Arrays.toString(value));
  }

  private static byte[] getValue(String string) throws ReflectiveOperationException {
    Field VALUE = String.class.getDeclaredField("value");
    VALUE.setAccessible(true);
    return (byte[]) VALUE.get(string);
  }
}
Code language:  Java  ( java )
```

If we call this program with Java 9 to 15, we get the following output:

```
$ java EncapsulationTest.java
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by EncapsulationTest (file:/.../EncapsulationTest.java) to field java.lang.String.value
WARNING: Please consider reporting this to the maintainers of EncapsulationTest
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
[72, 97, 112, 112, 121, 32, 67, 111, 100, 105, 110, 103, 33]
Code language:  Plaintext  ( plaintext )
```

We see some warnings, but then we get the desired bytes.

Deep reflection on _new_ packages, on the other hand, was not allowed by default and had to be explicitly allowed on the command line with "--add-opens" since the introduction of the module system.

The following example tries to instantiate the class `ConstantDescs`from the package added in Java 12 (i.e. after the introduction of the module system) `java.lang.constant`via its private constructor:

```
Constructor<ConstantDescs> constructor = ConstantDescs.class.getDeclaredConstructor();
constructor.setAccessible(true);
ConstantDescs constantDescs = constructor.newInstance();
Code language:  Java  ( java )
```

The program aborts with the following error message:

```
$ java ConstantDescsTest.java
Exception in thread "main" java.lang.reflect.InaccessibleObjectException: Unable to make private java.lang.constant.ConstantDescs() accessible: module java.base does not "opens java.lang.constant" to unnamed module @6c3f5566
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:361)
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:301)
        at java.base/java.lang.reflect.Constructor.checkCanSetAccessible(Constructor.java:189)
        at java.base/java.lang.reflect.Constructor.setAccessible(Constructor.java:182)
        at ConstantDescsTest.main(ConstantDescsTest.java:7)
Code language:  Plaintext  ( plaintext )
```

In order to make the program runnable, we need to `--add-opens`open via the new package for Deep Reflection:

```
$ java --add-opens java.base/java.lang.constant=ALL-UNNAMED ConstantDescsTest.java
Code language:  Plaintext  ( plaintext )
```

The code then runs error-free and without warnings.

### Since Java 16: Default Strong Encapsulation + Optional Relaxed Strong Encapsulation

In [Java 16](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Strongly_Encapsulate_JDK_Internals_by_Default) , the default mode was changed from "Relaxed Strong Encapsulation" to "Strong Encapsulation". Since then, access to pre-Java 9 packages has also had to be explicitly allowed.

If we run the first example on Java 16 without explicitly allowing access, we get the following error message:

```
$ java EncapsulationTest.java
Exception in thread "main" java.lang.reflect.InaccessibleObjectException: Unable to make field private final byte[] java.lang.String.value accessible: module java.base does not "opens java.lang" to unnamed module @62fdb4a6
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:357)
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:297)
        at java.base/java.lang.reflect.Field.checkCanSetAccessible(Field.java:177)
        at java.base/java.lang.reflect.Field.setAccessible(Field.java:171)
        at EncapsulationTest.getValue(EncapsulationTest.java:12)
        at EncapsulationTest.main(EncapsulationTest.java:6)
Code language:  Plaintext  ( plaintext )
```

However, Java 16 offered a workaround: You `--illegal-access=permit`could switch back to "Relaxed Strong Encapsulation" using the VM option:

```
$ java --illegal-access=permit EncapsulationTest.java
Java HotSpot(TM) 64-Bit Server VM warning: Option --illegal-access is deprecated and will be removed in a future release.
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by EncapsulationTest (file:/.../EncapsulationTest.java) to field java.lang.String.value
WARNING: Please consider reporting this to the maintainers of EncapsulationTest
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
[72, 97, 112, 112, 121, 32, 67, 111, 100, 105, 110, 103, 33]
Code language:  Plaintext  ( plaintext )
```

### Since Java 17: Strong encapsulation only

Per [JDK Enhancement Proposal 403](https://openjdk.org/jeps/403) , this option will be removed from Java 17. The VM option `--illegal-access`has been warned and inaccessible by `String.value`default:

```
java --illegal-access=permit EncapsulationTest.java
OpenJDK 64-Bit Server VM warning: Ignoring option --illegal-access=permit; support was removed in 17.0
Exception in thread "main" java.lang.reflect.InaccessibleObjectException: Unable to make field private final byte[] java.lang.String.value accessible: module java.base does not "opens java.lang" to unnamed module @3e77a1ed
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:354)
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:297)
        at java.base/java.lang.reflect.Field.checkCanSetAccessible(Field.java:178)
        at java.base/java.lang.reflect.Field.setAccessible(Field.java:172)
        at EncapsulationTest.getValue(EncapsulationTest.java:12)
        at EncapsulationTest.main(EncapsulationTest.java:6)
Code language:  Plaintext  ( plaintext )
```

If you want to use Deep Reflection from Java 17, you must now explicitly `--add-opens`allow it:

```
$ java --add-opens java.base/java.lang=ALL-UNNAMED EncapsulationTest.java
[72, 97, 112, 112, 121, 32, 67, 111, 100, 105, 110, 103, 33]
Code language:  Plaintext  ( plaintext )
```

The program is running and we no longer see any warnings - the long transition phase since Java 9 has thus been completed.

## Add java.time.InstantSource

The class `java.time.Clock`is extremely useful for writing tests that verify time-dependent functionality.

if `Clock`e.g. B. is injected into the application classes via dependency injection, you can mock them in tests or `Clock.fixed()`set a fixed time for the test execution.

Since `Clock`the method `getZone()`provides, you always have to think about which specific time zone you `Clock`instantiate an object with.

In order to enable alternative, time zone-independent time sources, the interface `java.time.InstantSource`from was `Clock`extracted in Java 17, which only provides the methods `instant()`and for querying the time, which is already implemented as a default method.`millis()``millis()`

The `Timer`class in the following example uses `InstantSource`to determine the start and end times of an `Runnable`execution and calculate the duration of the execution from that:

```
public class Timer {
  private final InstantSource instantSource;

  public Timer(InstantSource instantSource) {
    this.instantSource = instantSource;
  }

  public Duration measure(Runnable runnable) {
    Instant start = instantSource.instant();
    runnable.run();
    Instant end = instantSource.instant();
    return Duration.between(start, end);
  }
}
Code language:  Java  ( java )
```

In production we can `Timer`instantiate using the system clock (although in the absence of alternative implementations we have to `InstantSource`worry about the timezone - we'll use the system's default timezone):

```
Timer timer = new Timer(Clock.systemDefaultZone());Code language:  Java  ( java )
```

We can test the `measure()`method by `InstantSource`mocking its `instant()`method to return two fixed values and comparing the return value of `measure()`with the difference between these values:

```
@Test
void shouldReturnDurationBetweenStartAndEnd() {
  InstantSource instantSource = mock(InstantSource.class);
  when(instantSource.instant())
      .thenReturn(Instant.ofEpochMilli(1_640_033_566_000L))
      .thenReturn(Instant.ofEpochMilli(1_640_033_567_750L));

  Timer timer = new Timer(instantSource);
  Duration duration = timer.measure(() -> {});

  assertThat(duration, is(Duration.ofMillis(1_750)));
}Code language:  Java  ( java )
```

_There is no JDK Enhancement Proposal for this extension._

## Hex Formatting and Parsing Utility

Until now, we could use – or use – the `toHexString()`method of the classes `Integer`, `Long`, `Float`and to output hexadecimal numbers . The following code shows a few examples:`Double``String.format()`

```
System.out.println(Integer.toHexString(1_000));
System.out.println(Long.toHexString(100_000_000_000L));
System.out.println(Float.toHexString(3.14F));
System.out.println(Double.toHexString(3.14159265359));

System.out.println(
    "%x - %x - %a - %a".formatted(1_000, 100_000_000_000L, 3.14F, 3.14159265359));Code language:  Java  ( java )
```

The code results in the following output:

```
3e8
174876e800
0x1.91eb86p1
0x1.921fb54442eeap1
3e8 - 174876e800 - 0x1.91eb86p1 - 0x1.921fb54442eeap1Code language:  Plaintext  ( plaintext )
```

We could parse hexadecimal numbers with the respective counterparts:

```
Integer.parseInt("3e8", 16);
Long.parseLong("174876e800", 16);
Float.parseFloat("0x1.91eb86p1");
Double.parseDouble("0x1.921fb54442eeap1");
Code language:  Java  ( java )
```

Java 17 provides the new class `java.util.HexFormat`that is used to represent and parse hexadecimal numbers via a unified API. `HexFormat`supports all primitive integers ( `int`, `byte`, `char`, `long`, `short`) and `byte`arrays – but no floating point numbers.

Here is an example of the conversion to hexadecimal numbers:

```
HexFormat hexFormat = HexFormat.of();

System.out.println(hexFormat.toHexDigits('A'));
System.out.println(hexFormat.toHexDigits((byte) 10));
System.out.println(hexFormat.toHexDigits((short) 1_000));
System.out.println(hexFormat.toHexDigits(1_000_000));
System.out.println(hexFormat.toHexDigits(100_000_000_000L));
System.out.println(hexFormat.formatHex(new byte[] {1, 2, 3, 60, 126, -1}));Code language:  Java  ( java )
```

The output is:

```
0041
0a
03e8
000f4240
000000174876e800
0102033c7effCode language:  Plaintext  ( plaintext )
```

It is noticeable that the output always comes with leading zeros.

The representation we can z. B. adjust as follows:

```
HexFormat hexFormat = HexFormat.ofDelimiter(" ").withPrefix("0x").withUpperCase();
Code language:  Java  ( java )
```

-   `ofDelimiter()`Specifies a delimiter for formatting byte arrays.
-   `withPrefix()`defines a prefix – but only for byte arrays!
-   `withUpperCase()`switches the output to uppercase.

The output is now:

```
0041
0A
03E8
000F4240
000000174876E800
0x01 0x02 0x03 0x3C 0x7E 0xFFCode language:  Plaintext  ( plaintext )
```

The leading zeros cannot be removed.

We can parse integers as follows:

```
int i = HexFormat.fromHexDigits("F4240");
long l = HexFormat.fromHexDigitsToLong("174876E800");
Code language:  Java  ( java )
```

Corresponding methods for `char`, `byte`and `short`do not exist.

We can parse byte arrays e.g. B. as follows:

```
HexFormat hexFormat = HexFormat.ofDelimiter(" ").withPrefix("0x").withUpperCase();
byte[] bytes = hexFormat.parseHex("0x01 0x02 0x03 0x3C 0x7E 0xFF");
Code language:  Java  ( java )
```

There are other methods to B. to parse only a substring. [See the HexFormat JavaDoc for](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HexFormat.html) full documentation .

_There is no JDK Enhancement Proposal for this extension._

## Context-Specific Deserialization Filters

Object deserialization poses a significant security risk. Malicious attackers can use the data stream to be deserialized to construct objects, through which they can ultimately execute arbitrary code in arbitrary classes (available on the classpath).

Java 9 introduced deserialization filters, the ability to specify which classes may (or may not) be deserialized.

Previously there were two ways to define deserialization filters:

-   per `ObjectInputStream.setObjectInputFilter()`for each deserialization separately,
-   system-wide uniformly via system property `jdk.serialFilter`or via security property of the same name in the file `conf/security/java.properties`.

For complex applications, especially those with third-party libraries that also contain deserialization code, these variants are not satisfactory. For example, deserialization in third-party code cannot be `ObjectInputStream.setObjectInputFilter()`configured via (unless you change their source code), but only globally.

[As of Java 17, JDK Enhancement Proposal 415](https://openjdk.org/jeps/415) makes it possible to set context-specific deserialization filters, e.g. B. for a specific thread or based on the call stack for a specific class, module, or third-party library.

The configuration of the filters is not easy and would go beyond the scope of this article. Details can be found in the JEP linked above.

### JDK Flight Recorder Event for Deserialization

As of Java 17 it is also possible to monitor the deserialization of objects via the JDK Flight Recorder (JFR).

Deserialization events are disabled by default and must be enabled via the event identifier `jdk.Deserialization`in the JFR configuration file (see the article linked below for an example).

If a deserialization filter is activated, the JFR event contains an indication of whether the deserialization was executed or rejected.

For more detailed information and an example, see the article " [Monitoring Deserialization to Improve Application Security](https://inside.java/2021/03/02/monitoring-deserialization-activity-in-the-jdk/) ".

_The Flight Recorder events for the deserialization are not part of the above JDK Enhancement Proposal; there is also no separate JEP for them._

## Enhanced Pseudo-Random Number Generators

Until now, it has been tedious to replace the [random number-generating](https://www.java-springboot-kafka.github.io/de/java/zufallszahl/) classes `Random`and `SplittableRandom`in an application (or even to replace them with other algorithms) even though they offer a largely identical set of methods (e.g. `nextInt()`, `nextDouble()`and stream-generating methods such as `ints()`and `longs()`).

So far, the class hierarchy looked like this:

![Pre-Java 17 Pseudo-Random Number Generators](source/images/random_number_generators_pre_java17-600x249.png)

Pre-Java 17 Pseudo-Random Number Generators

JDK [Enhancement Proposal 356](https://openjdk.org/jeps/356) introduced a framework of inheriting interfaces for the existing and new algorithms in Java 17, so that the concrete algorithms can easily be exchanged in the future:

![Java 17 Pseudo-Random Number Generators](source/images/random_number_generators_java17-800x438.png)

Java 17 Pseudo-Random Number Generators

The methods common to all random number generators such as `nextInt()`and `nextDouble()`are `RandomGenerator`defined in . If you only need these methods, you should always use this interface in the future.

The framework includes three new types of random number generators:

-   `JumpableGenerator`: provides methods to skip a large number of random numbers (e.g. 2 <sup><span><span>64 ).</span></span></sup>
-   `LeapableGenerator`: provides methods to skip a _very_ large number of random numbers (e.g. 2 <sup><span><span>128 ).</span></span></sup>
-   `ArbitrarilyJumpableGenerator`: provides additional methods to skip _any_ number of random numbers.

Also, code duplicated in the existing classes has been eliminated and code has been extracted into non-public abstract classes (not visible in the class diagram) to make it reusable for future random number generator implementations.

`RandomGeneratorFactory`New random number generators can be added and instantiated via the Service Provider Interface (SPI) in the future .

## Performance

With asynchronous logging, Java 17 brings a long-overdue performance improvement to the Unified JVM logging system introduced in Java 9.

### Unified Logging Supports Asynchronous Log Flushing

Asynchronous logging is a feature supported by all Java logging frameworks. The application thread first writes log messages to a queue. A separate I/O thread then routes them to the configured output (console, file, or network).

The application thread does not have to wait for the I/O subsystem to process the message.

As of Java 17, asynchronous logging can also be activated for the JVM itself. This is done using the following VM option:

`-Xlog:async`

The logging queue is limited to a fixed size. If the application sends more log messages than the I/O thread can process, the queue fills up. She then rejects further messages without comment.

The size of the queue can be adjusted using the following VM option:

`-XX:AsyncLogBufferSize=<Bytes>`

_There is no JDK Enhancement Proposal for this extension._

## Preview and Incubator features

Even though Java 17 is a long-term support (LTS) release, it contains preview and incubator features that are expected to reach production maturity in one of the next "intermediate releases". If you only use LTS releases, you have to wait at least until Java 23 to use these features.

### Pattern Matching for switch (Preview)

In Java 16 ["Pattern Matching for instanceof"](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Pattern_Matching_for_instanceof) was introduced. This made explicit casts after `instanceof`tests superfluous. This allows e.g. B. Code like the following:

```
if (obj instanceof String s) {
  if (s.length() > 5) {
    System.out.println(s.toUpperCase());
  } else {
    System.out.println(s.toLowerCase());
  }
} else if (obj instanceof Integer i) {
  System.out.println(i * i);
}Code language:  Java  ( java )
```

With [JDK Enhancement Proposal 406](https://openjdk.org/jeps/406) , the check whether an object is an instance of a certain class can also be written as a `switch`statement (or expression) in the future.

#### Pattern matching for switch statements

Here is the example from above rewritten into a `switch`statement:

```
switch (obj) {
  case String s -> {
    if (s.length() > 5) {
      System.out.println(s.toUpperCase());
    } else {
      System.out.println(s.toLowerCase());
    }
  }

  case Integer i -> System.out.println(i * i);

  default -> {}
}Code language:  Java  ( java )
```

It is noticeable that the `default`case must be specified - in this case with an empty code block, since an action should only be carried out for strings and integers.

The code becomes much more readable if we combine `case`\- and - expressions with a logical "and" (this is then called a "guarded pattern"):`if`

```
switch (obj) {
  case String s && s.length() > 5 -> System.out.println(s.toUpperCase());
  case String s                   -> System.out.println(s.toLowerCase());

  case Integer i                  -> System.out.println(i * i);

  default -> {}
}
Code language:  Java  ( java )
```

It is important that a so-called "dominant _pattern_ " must come after a "dominant _pattern_ ". In the example, the shorter pattern from line 3 "String s" dominates the longer one from line 2.

If we swapped these lines, it would look like this:

```
switch (obj) {
  case String s                   -> System.out.println(s.toLowerCase());
  case String s && s.length() > 5 -> System.out.println(s.toUpperCase());

  ...
}
Code language:  Java  ( java )
```

In this case, the compiler would criticize line 3 with the following error message:

_Label is dominated by a preceding case label 'String s'_

The reason for this is that now every string – no matter what length – is matched by the pattern "String s" (line 2) and doesn't even get to the second case test (line 3).

#### Pattern matching for switch expressions

Pattern matching can also be used for [`switch`expressions](https://www.java-springboot-kafka.github.io/de/java/switch-expressions/#switch-als-ausdruck-mit-ruckgabewert) ( i.e. `switch`with a return value):

```
String output = switch (obj) {
  case String s && s.length() > 5 -> s.toUpperCase();
  case String s                   -> s.toLowerCase();

  case Integer i                  -> String.valueOf(i * i);
  
  default -> throw new IllegalStateException("Unexpected value: " + obj);
};
Code language:  Java  ( java )
```

The case must `default`return a value, otherwise the return value of the `switch`expression could be undefined - or, as in the example, throw an exception.

#### Completeness check with Sealed Classes

Incidentally, when using [sealed classes](https://www.java-springboot-kafka.github.io/de/java/sealed-classes-de/) , the compiler can check whether `switch`the statement or expression is complete. If that is the case, a `default`case is not needed.

This has an advantage that is not immediately apparent: if one day the sealed hierarchy is extended, the compiler will recognize the then incomplete `switch`expression and you will be forced to complete it. This saves you from making mistakes that go unnoticed.

"Pattern Matching for switch" will be presented again in [Java 18](https://www.java-springboot-kafka.github.io/de/java/java-18-features-de/#Pattern_Matching_for_switch_Second_Preview) as a preview feature and will probably be ready for production in Java 19.

### Foreign Function & Memory API (Incubator)

Since Java 1.1, the Java Native Interface (JNI) offers the possibility of calling native C code from within Java. However, JNI is extremely complex to implement and slow to run.

To create a JNI replacement, [Project Panama](https://openjdk.org/projects/panama/) was created. Concrete goals of this project are a) to simplify the effort of the implementation (90% of the work is to be eliminated) and b) to improve the performance (by a factor of 4 to 5).

In the last three Java versions, two new APIs have been introduced, each in the incubator stage:

1.  The Foreign Memory Access API (introduced in [Java 14](https://www.java-springboot-kafka.github.io/de/java/java-14-features-de/#Foreign-Memory_Access_API_Incubator) , refined in [Java 15](https://www.java-springboot-kafka.github.io/de/java/java-15-features-de/#Foreign-Memory_Access_API_Second_Incubator) and [Java 16](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Foreign_Linker_API_Incubator_Foreign-Memory_Access_API_Third_Incubator) ),
2.  The Foreign Linker API (introduced in [Java 16](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Foreign_Linker_API_Incubator_Foreign-Memory_Access_API_Third_Incubator) ).

JDK [Enhancement Proposal 412](https://openjdk.org/jeps/412) combined both APIs in Java 17 into the "Foreign Function & Memory API".

This is still in the incubator stage, so it may still be subject to significant changes. [I will introduce the new API in the Java 19](https://www.java-springboot-kafka.github.io/de/java/java-19-features/#Foreign_Function_Memory_API_Preview) article when it reaches the preview stage.

### Vector API (Second Incubator)

[As already described in the article about Java 16](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#Vector_API_Incubator) , the Vector API is not about the old `java.util.Vector`class, but about mapping mathematical vector calculations to modern CPU architectures with single-instruction-multiple-data (SIMD) support.

JDK [Enhancement Proposal 414](https://openjdk.org/jeps/414) improved performance and extended the API, e.g. B. to support `Character`(previously , `Byte`, `Short`, `Integer`, `Long`and `Float`were `Double`supported).

Because features in Incubator status can still undergo significant changes, I'll announce the feature when it reaches Preview status.

## Deprecations and Deletion

In Java 17, some obsolete features were again marked as "deprecated for removal" or completely removed.

### Deprecate the Applet API for Removal

Java applets are no longer supported by any modern web browser and were already marked as "deprecated" in Java 9.

JDK [Enhancement Proposal 398](https://openjdk.org/jeps/398) marks them as "deprecated for removal" in Java 17. This means that they will be completely removed in one of the next releases.

### Deprecate the Security Manager for Removal

The Security Manager has been part of the platform since Java 1.0 and was primarily intended to protect the user's computer and data from downloaded Java applets. These were launched in a sandbox in which the Security Manager generally denied access to resources such as the file system or the network.

As described in the previous section, Java applets have been marked as “deprecated for removal”; this aspect of the Security Manager will no longer be relevant.

_In addition to the browser sandbox, which generally_ denied access to resources , the Security Manager was also able to secure server applications using policy files. Elasticsearch and Tomcat are examples of this.

However, it's not of much interest anymore because it's complicated to configure and security can now be better implemented via the Java modular system or isolation through containerization.

In addition, the Security Manager represents a not inconsiderable amount of maintenance work. For all extensions to the Java class library, it must be evaluated to what extent they need to be secured using the Security Manager.

For these reasons, [JDK Enhancement Proposal 411](https://openjdk.org/jeps/411) in Java 17 classified the Security Manager as "deprecated for removal".

It is not yet certain when the Security Manager will be completely removed. In Java 18 it will still be included.

### Remove RMI Activation

Remote Method Invocation is a technology for invoking methods on "remote objects", i.e. objects on another JVM.

RMI Activation allows objects destroyed on the target JVM to be automatically re-instantiated as soon as they are accessed. This is intended to eliminate the need for error handling on the client side.

However, RMI activation is relatively complex and results in ongoing maintenance costs; it's also virtually unused, as analysis of open source projects and forums like StackOverflow have shown.

For this reason, RMI Activation was marked as "deprecated" in [Java 15](https://www.java-springboot-kafka.github.io/de/java/java-15-features-de/#Deprecate_RMI_Activation_for_Removal) and completely removed in Java 17 by [JDK Enhancement Proposal 407 .](https://openjdk.org/jeps/407)

### Remove the Experimental AOT and JIT Compiler

In Java 9, Graal was added to the JDK as an experimental ahead-of-time (AOT) compiler. In [Java 10](https://www.java-springboot-kafka.github.io/de/java/java-10-features-de/#Experimental_Java-Based_JIT_Compiler) , Graal was also made available as a just-in-time (JIT) compiler.

However, both features have been little used since then. Because the maintenance overhead is significant, Graal has been removed in the JDK 16 builds released by Oracle. Since nobody complained about this, both AOT and JIT compilers have been completely removed in Java 17 with [JDK Enhancement Proposal 410 .](https://openjdk.org/jeps/410)

The Java-Level JVM Compiler Interface (JVMCI) used to connect Graal has not been removed, and Graal itself is also being further developed. To use Graal as an AOT or JIT compiler, you can download the Java distribution [GraalVM](https://www.graalvm.org/) .

## Other changes in Java 17

In this section you will find minor changes to the Java class library that you will not come into contact with on a daily basis. Still, I recommend skimming through them at least once so you know where to look when you need a feature.

### New API for Accessing Large Icons

Here is a small Swing application that renders the directory `C:\Windows`'s filesystem icon on Windows:

```
FileSystemView fileSystemView = FileSystemView.getFileSystemView();
Icon icon = fileSystemView.getSystemIcon(new File("C:\\Windows"));

JFrame frame = new JFrame();
frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
frame.getContentPane().add(new JLabel(icon));
frame.pack();
frame.setVisible(true);Code language:  Java  ( java )
```

The icon has a size of 16 x 16 pixels, and there was no way to display a higher resolution icon.

In Java 17 the method was `getSystemIcon(File f, int width, int height)`added with which you can specify the size of the icon:

```
Icon icon = fileSystemView.getSystemIcon(new File("C:\\Windows"), 512, 512);Code language:  Java  ( java )
```

_There is no JDK Enhancement Proposal for this extension._

### Add support for UserDefinedFileAttributeView on macOS

The following code shows how extended attributes of a file can be written and read:

```
Path path = ...

UserDefinedFileAttributeView view =
    Files.getFileAttributeView(path, UserDefinedFileAttributeView.class);

// Write the extended attribute with name "foo" and value "bar"
view.write("foo", StandardCharsets.UTF_8.encode("bar"));

// Print a list of all extended attribute names
System.out.println("attribute names: " + view.list());

// Read the extended attribute "foo"
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
view.read("foo", byteBuffer);
byteBuffer.flip();
String value = StandardCharsets.UTF_8.decode(byteBuffer).toString();
System.out.println("value of 'foo': " + value);Code language:  Java  ( java )
```

This functionality has existed since Java 7, but was not previously supported on macOS either. Since Java 17, the function is now also available for macOS.

_There is no JDK Enhancement Proposal for this extension._

### System Property for Native Character Encoding Name

As of Java 17, the system property "native.encoding" can be used to call up the standard character encoding of the operating system:

```
System.out.println("native encoding: " + System.getProperty("native.encoding"));Code language:  Java  ( java )
```

On Windows it `Cp1252`outputs , on Linux and macOS `UTF-8`.

If you call this code with Java 16 or earlier, is `null`output.

_There is no JDK Enhancement Proposal for this extension._

### Restore Always-Strict Floating-Point Semantics

An almost unknown Java keyword is `strictfp`. It is used in class definitions to make floating point operations strict within a class. This means that they lead to the same, predictable results on all architectures.

Strict floating-point semantics was the default behavior prior to Java 1.2 (that is, more than 20 years ago).

As of Java 1.2, the "standard floating point semantics" was used by default, which could lead to slightly different results depending on the processor architecture. On the other hand, it performed better on the x87 floating-point coprocessor that was widespread at the time, since it had to perform additional operations for the strict semantics (you can find more details in this [Wikipedia article](https://en.wikipedia.org/wiki/Strictfp) ).

Starting with Java 1.2, anyone who wanted to continue to use strict calculations had to `strictfp`mark this with the keyword in the class definition:

```
public strictfp class PredictiveCalculator {
  // ...
}
Code language:  Java  ( java )
```

Modern hardware can carry out the strict floating-point semantics without any loss of performance, so that in [JDK Enhancement Proposal 306](https://openjdk.org/jeps/306) it was decided to make this the standard semantics again from Java 17.

The `strictfp`keyword is therefore superfluous. Using it results in a compiler warning:

```
$ javac PredictiveCalculator.java
PredictiveCalculator.java:3: warning: [strictfp] as of release 17, 
all floating-point expressions are evaluated strictly and 'strictfp' is not required
Code language:  Plaintext  ( plaintext )
```

### New macOS Rendering Pipeline

In 2018, Apple marked the OpenGL library previously used by Java Swing for rendering under macOS as "deprecated" and presented the [Metal framework](https://developer.apple.com/metal/) as its successor.

[JDK Enhancement Proposal 382](https://openjdk.org/jeps/382) transitions the Swing rendering pipeline for macOS to the Metal API.

### macOS/AArch64 Port

Apple has announced that it will switch Macs from x64 to AArch64 CPUs in the long term. Accordingly, a corresponding port is provided with [JDK Enhancement Proposal 391 .](https://openjdk.org/jeps/391)

The code is an extension of the AArch64 ports for Linux and Windows released in Java 9 and [Java 16 with macOS-specific adjustments.](https://www.java-springboot-kafka.github.io/de/java/java-16-features-de/#WindowsAArch64_Port)

### New Page for "New API" and Improved "Deprecated" Page

In JavaDoc generated from Java 17, there is a "NEW" page that shows all new features grouped by version. To do this, the `@since`tags of the modules, packages, classes, etc. are evaluated.

!["NEW" page in JavaDoc generated since Java 17](source/images/javadoc_since_java_17-800x503.png)

"NEW" page in JavaDoc generated since Java 17

In addition, the "DEPRECATED" page has been revised. Up until Java 16, here is an ungrouped list of all features marked as "deprecated":

![Java 16 DEPRECATED page](source/images/javadoc_deprecated_java_16-800x435.png)

Java 16 DEPRECATED page

Starting with Java 17, we see the features marked as "deprecated" grouped by release:

![Java 17 DEPRECATED page](source/images/javadoc_deprecated_java_17-800x460.png)

Java 17 DEPRECATED page

_There is no JDK Enhancement Proposal for this extension._

### Complete list of all changes in Java 17

This article has presented all changes defined in JDK Enhancement Proposals (JEPs) as well as numerous class library extensions for which no JEPs exist. For more changes, especially to the security libraries, see the [official Java 17 release notes](https://www.oracle.com/java/technologies/javase/17-relnote-issues.html) .

## Conclusion

Although Java 17 is the latest LTS release, this release is not significantly different from the previous ones. We got a mix of:

-   new language features (sealed classes),
-   API changes ( `InstantSource`, `HexFormat`, context-specific deserialization filters),
-   a performance improvement (asynchronous logging of the JVM),
-   Deprecations and deletions (Applet API, Security Manager, RMI Activation, AOT and JIT Compiler)
-   und neuen Preview- und Incubator-Features (Pattern Matching for switch, Foreign Function & Memory API, Vector API).

In addition, the path taken in Java 9 with Project Jigsaw has been brought to an end by removing the transitionally provided "Relaxed Strong Encapsulation" mode and access to private members of other modules (deep reflection) must always be explicitly released.

Did you like the article? Then please leave me a comment or share the article using one of the share buttons at the end.

[Java 18](https://www.java-springboot-kafka.github.io/de/java/java-18-features-de/) is just around the corner and the features it contains are certain. I will present these in the next article. Would you like to be informed when the article goes online? Then [click here](https://www.java-springboot-kafka.github.io/de/java/java-17-features-de/#) to sign up for the HappyCoders newsletter.