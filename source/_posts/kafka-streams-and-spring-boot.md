Processing continuous data streams in distributed systems without any time delay poses a number of challenges. We show you how stream processing can succeed with Kafka Streams and Spring Boot.

![](https://media.graphassets.com/S9gkDSDBR8mtW0N24a95)

Everything in flow: If you look at data as a continuous stream of information, you can get a lot of speed out of it.

## TL;DR

Little time? Then here's the ultra-short summary:

-   Stream processing is well suited to process large amounts of data asynchronously and with minimal delay.
-   Modern streaming frameworks allow you to align your application architecture completely with event streams and turn your data management inside out. The event stream becomes the “ _source of truth_ ”.
-   With Kafka Streams, Kafka offers an API to process streams and map complex operations to them. means[_KStreams_](https://docs.confluent.io/platform/current/streams/concepts.html#streams-concepts-kstream) and[_KTables_](https://docs.confluent.io/platform/current/streams/concepts.html#ktable) you can also map more complex use cases that have to maintain a state. This state is managed by Kafka, so you don't have to worry about data management yourself.
-   Spring Boot offers one[Stream abstraction](https://spring.io/projects/spring-cloud-stream) that can be used to implement stream processing workloads.

You can find the entire code for our sample project at[GitHub](https://github.com/codecentric/spring-kafka-streams-example).

## Processing large amounts of data quickly – a perennial topic

In our everyday project environment, we often deal with use cases in which we have to process a continuous stream of events through several systems involved with as little delay as possible. Two examples:

-   Let's imagine a classic web shop: customers order goods around the clock. The information about the incoming order is of interest for various subsystems: Among other things, the warehouse needs information about the items to be shipped, we have to write an invoice and maybe now reorder goods ourselves.
-   Another scenario: A car manufacturer analyzes their vehicles' telemetry data to improve the durability of their vehicles. For this purpose, the components of thousands of cars send sensor data every second, which then has to be examined for anomalies.

The larger the amounts of data in both examples, the more difficult it becomes for us to scale our system adequately and to process the data in the shortest possible time. This describes a general problem: the volume of data that we are confronted with in everyday life is constantly increasing, while our customers expect us to process the data and make it usable as quickly as possible.[Moderne Stream Processing Frameworks](https://blog.codecentric.de/2017/03/verteilte-stream-processing-frameworks-fuer-fast-data-big-data-ein-meer-moeglichkeiten/) should address precisely these aspects.

In this blog post, we would like to use a concrete use case to demonstrate how a stream processing architecture can be implemented with Spring Boot and Apache Kafka Streams. We want to go into the conception of the overall system as well as the everyday problems that we should take into account during implementation.

## Streams and events briefly outlined

We can classify stream processing as an alternative to batch processing. Instead of "heaping" all incoming data and processing them en bloc at a later point in time, the idea behind stream processing is to view incoming data as a continuous stream: the data is processed continuously. Depending on the API and programming model, we can use a corresponding domain-specific language to define operations on the data stream. Sender and receiver do not need any knowledge about each other. The systems participating in the stream are usually decoupled from one another via a corresponding messaging backbone.

In addition to the advantages of time-critical processing of data, the concept is well suited to reducing dependencies between services in distributed systems. Through indirection via a central messaging backbone, services can now switch to an asynchronous communication model in that they no longer communicate via commands but via events. While commands are direct, synchronous calls between services that trigger an action or result in a change of state, events only transmit the information that an event has occurred. With event processing, the recipient decides when and how this information is processed. This procedure is helpful to achieve a looser coupling between the components of an overall system[\[1\]](https://www.confluent.io/designing-event-driven-systems/).

Using Kafka as a stream processing platform allows us to align our overall system to events, as the next section shows.

## Kafka as a stream processing platform

What distinguishes Kafka from classic message brokers such as RabbitMQ or Amazon SQS is the permanent storage of event streams and the provision of an API for processing these events as streams. This enables us to turn the architecture of a distributed system inside out and make these events the _"source of truth"_ : If the state of our entire system can be established based on the sequence of all events and these events are stored permanently, this state can be changed at any time by processing of the event log can be (re)established. Martin Kleppmann described the concept of a globally available, unchangeable event log as _"turning the database inside out"_ .[\[2\]](https://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/). What is meant by this is that we can distribute the concepts that we traditionally provide encapsulated as a black box within a relational database (a transaction log, a query engine, indexes and caches) through Kafka and Kafka Streams to the components of a system.  
To build a streaming architecture based on this theory, we use two different components from the Kafka ecosystem:

-   **Kafka Cluster** : Provides event storage. Acts as the immutable and permanently stored transaction log.
-   **Kafka Streams** : Provides the API for stream processing (Streams API). Abstracts the components for generating and consuming the messages and provides the programming model to process the events and map caches and queries to them[\[3\]](https://kafka.apache.org/30/documentation/streams/core-concepts)

In addition to aligning to events and providing APIs to process them, Kafka also comes with some mechanisms to scale with large amounts of data. The most important mechanism is partitioning: messages are distributed to different partitions so that they can be read and written in parallel as efficiently as possible. Provides a good overview of the central concepts and vocabulary in the Kafka cosmos[\[4\]](https://kafka.apache.org/documentation/#intro_concepts_and_terms).

Based on a concrete use case, we now want to show you how we can implement a distributed streaming architecture with Spring Boot and Kafka.

## An exemplary use case

Let's imagine that our customer - a space agency - commissioned us to develop a system for evaluating telemetry data from various space probes in space. The general conditions and requirements are as follows:

-   We have an unspecified set of probes giving us a steady stream of telemetry readings. These probes belong to either the US Space Agency (NASA) or the European Space Agency (ESA).
-   All space probes send their measurement data in the imperial system.
-   Our customer is only interested in the aggregated measurement data per probe:
    -   What is the total distance a given probe has traveled so far?
    -   What is the top speed the probe has reached so far?
-   Since the measurement data from the NASA and ESA probes are processed by different teams, they should be able to be consumed separately.
    -   Data from the ESA probes are to be converted from the imperial to the metric system.

## The target architecture

In our example, we are dealing with a continuous stream of readings that we consider our events. Since we have to perform a series of transformations and aggregations on these, the use case lends itself well to processing as a stream. The need to aggregate measurement data also suggests that some part of our application needs to be able to remember a state in order to keep the summed values per probe.

To implement the use case, we divide the application into 3 subcomponents. A Kafka cluster forms the central hub for communication between the components:

![Architectural sketch of our sample application](https://blog.codecentric.de/_next/image?url=https%3A%2F%2Fmedia.graphassets.com%2FXy3kzvYgT6GRsaxYg6va&w=2048&q=75)  
We will build this example architecture to illustrate our use case. We use Kafka as the central communication hub between our services.

We arrange the distribution of tasks between the services as follows:

-   **kafka-samples-producer** : Converts the received measurement data into a machine-readable format and stores it on a Kafka topic. Since we don't have any real space probes handy at the moment, we let this service generate random measurement data.
-   **kafka-samples-streams** : Performs the calculation of the aggregated measurement data and the subdivision by measurement data for NASA or ESA. Since the previously calculated values are also included in the calculation, the application must maintain a local state. We map this using the Streams API in the form of two KTables (we already separate them by space agency here). The KTables are materialized transparently for the application by a so-called state store, which saves the history of the state in Kafka Topics.
-   **kafka-samples-consumer** : Represents an example client service of a space agency, which is responsible for the further processing of the aggregated measurement data. In our case, this reads both output topics, in the case of the ESA, converts them to the metric system and logs these values to stdout.

## Implementation of the Services

We have implemented all services with Spring Boot and Kotlin and use the for configuration and implementation[Spring abstraction for streams](https://spring.io/projects/spring-cloud-stream). In the following sections we will go into the concrete implementation of the individual services.

### Generation of the telemetry data (kafka-samples-producer)

To write the (fictitious) probe measurement data, we use the Kafka Producer API, available in Spring via the[Spring Cloud Stream Binder for Kafka](https://cloud.spring.io/spring-cloud-stream-binder-kafka/spring-cloud-stream-binder-kafka.html#_apache_kafka_binder) provided. We configure the service via the ([application.yml](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-producer/src/main/resources/application.yml)) as follows:

```
spring:
  application:
    name: kafka-telemetry-data-producer
  cloud:
    stream:
      kafka:
        binder:
          brokers: "localhost:29092"
        bindings:
          telemetry-data-out-0:
            producer:
              configuration:
                key.serializer: org.springframework.kafka.support.serializer.ToStringSerializer
                value.serializer: org.springframework.kafka.support.serializer.JsonSerializer
                # Otherwise de.codecentric.samples.kafkasamplesproducer.event.TelemetryData will be added as a header info
                # which can't be deserialized by consumers (unless they have kafka.properties.spring.json.use.type.headers: false themselves)
                spring.json.add.type.headers: false
      bindings:
        telemetry-data-out-0:
          producer:
            # use kafka internal encoding
            useNativeEncoding: true
          destination: space-probe-telemetry-data
```

The configuration consists of a Kafka-specific (upper `bindings`configuration block) and a technology-agnostic (lower `bindings`configuration block) part, which are bound together via the binding.

In the example we create the binding `telemetry-data-out-0`. This declaration is based on the following convention:

```
<Funktionsname>-<in|out>-<n>
```

The `in`or `out`defines whether the binding is an input (an incoming stream of data) or an output (an outgoing stream of data). With the number increasing from 0 at the end, a function can be attached to several bindings - and thus read from several topics with one function - or written to several topics.

In the Kafka-specific part, we prevent Spring from adding a Type header to every message. Otherwise, this would mean that a consumer of the message - should he not actively prevent this - does not know the class specified in the header and therefore cannot deserialize the message.

The technology-agnostic part is used in this form for all Spring Cloud Streams-supported implementations like RabbitMQ, AWS SQS, etc. `destination`All you have to do here is specify the output target ( ) – in our case, this maps to the name of the Kafka topic that we want to describe.

After the service is configured, we define a Spring component to write the metrics ([TelemetryDataStreamBridge.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-producer/src/main/kotlin/de/codecentric/samples/kafkasamplesproducer/TelemetryDataStreamBridge.kt)):

```
package de.codecentric.samples.kafkasamplesproducer

import de.codecentric.samples.kafkasamplesproducer.event.TelemetryData
import mu.KotlinLogging
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.cloud.stream.function.StreamBridge
import org.springframework.kafka.support.KafkaHeaders
import org.springframework.messaging.support.MessageBuilder
import org.springframework.stereotype.Component

@Component
class TelemetryDataStreamBridge(@Autowired val streamBridge: StreamBridge) {

    private val logger = KotlinLogging.logger {}

    fun send(telemetryData: TelemetryData) {
        val kafkaMessage = MessageBuilder
            .withPayload(telemetryData)
            // Make sure all messages for a given probe go to the same partition to ensure proper ordering
            .setHeader(KafkaHeaders.MESSAGE_KEY, telemetryData.probeId)
            .build()
        logger.info { "Publishing space probe telemetry data: Payload: '${kafkaMessage.payload}'" }
        streamBridge.send("telemetry-data-out-0", kafkaMessage)
    }
}
```

As an entry point into the streaming world, Spring Cloud Streaming offers two different options:

-   The imperative`StreamBridge`
-   the reactive one`EmitterProcessor`

For this use case we use the `StreamBridge`. We can have Spring inject this and write the generated probe data to our topic. We use the ID of the respective probe as the message key, so that data from a probe always end up on the same partition. `send()`We pass the binding created in the configuration to the function .

### Processing of the telemetry data (kafka-samples-streams)

Most of the use case is processed in this part of our application. We use the Kafka Streams API to consume the generated probe data, perform the necessary calculations, and then write the aggregated measurement data to the two target topics. In Spring Boot, we can access the Streams API via the[Spring Cloud Stream Binder for Kafka Streams](https://cloud.spring.io/spring-cloud-stream-binder-kafka/spring-cloud-stream-binder-kafka.html#_kafka_streams_binder).

Analogous to the Producer API, we start with creating our bindings and configure our service via the file [application.yml](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-streams/src/main/resources/application.yml).

```
spring:
  kafka.properties.spring.json.use.type.headers: false
  application:
    name: kafka-telemetry-data-aggregator
  cloud:
    function:
      definition: aggregateTelemetryData
    stream:
      bindings:
        aggregateTelemetryData-in-0:
          destination: space-probe-telemetry-data
        aggregateTelemetryData-out-0:
          destination: space-probe-aggregate-telemetry-data-nasa
        aggregateTelemetryData-out-1:
          destination: space-probe-aggregate-telemetry-data-esa
      kafka:
        binder:
          brokers: "localhost:29092"
        streams:
          bindings:
            aggregateTelemetryData-in-0.consumer:
              keySerde: org.apache.kafka.common.serialization.Serdes$StringSerde
              valueSerde: com.example.kafkasamplesstreams.serdes.TelemetryDataPointSerde
              deserializationExceptionHandler: logAndContinue
            aggregateTelemetryData-out-0.producer:
              keySerde: org.apache.kafka.common.serialization.Serdes$StringSerde
              valueSerde: com.example.kafkasamplesstreams.serdes.AggregateTelemetryDataSerde
            aggregateTelemetryData-out-1.producer:
              keySerde: org.apache.kafka.common.serialization.Serdes$StringSerde
              valueSerde: com.example.kafkasamplesstreams.serdes.AggregateTelemetryDataSerde
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

To implement the feature, we use the **_functional style_** that was introduced with Spring Cloud Stream 3.0.0. To do this, we specify `aggregateTelemetryData` the name of our bean in the _function definition_ that implements the function. This will contain the actual technical logic.

Since we are reading from one topic and writing to two topics, we need three bindings here:

-   A `IN`binding to consume our metrics
-   A `OUT`binding to write our aggregated measurement data for NASA
-   A `OUT`binding to write our aggregated measurement data for the ESA

_We can view the function_ declared in the upper part of the configuration as a mapping of our `IN`bindings to our `OUT`bindings. In order for this to be associated with the bindings, we must adhere to the Spring convention described in the previous section.

With the binding configuration complete, we can move on to implementing our business logic. To do this, we create a function that matches the name of the functional binding from our configuration. This function maps our Kafka Streams topology and calculation logic ([KafkaStreamsHandler.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-streams/src/main/kotlin/com/example/kafkasamplesstreams/KafkaStreamsHandler.kt)):

```
package com.example.kafkasamplesstreams

import com.example.kafkasamplesstreams.events.AggregatedTelemetryData
import com.example.kafkasamplesstreams.events.SpaceAgency
import com.example.kafkasamplesstreams.events.TelemetryDataPoint
import com.example.kafkasamplesstreams.serdes.AggregateTelemetryDataSerde
import mu.KotlinLogging
import org.apache.kafka.common.serialization.Serdes
import org.apache.kafka.streams.kstream.KStream
import org.apache.kafka.streams.kstream.Materialized
import org.apache.kafka.streams.kstream.Predicate
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration


@Configuration
class KafkaStreamsHandler {

    private val logger = KotlinLogging.logger {}

    @Bean
    fun aggregateTelemetryData(): java.util.function.Function<
            KStream<String, TelemetryDataPoint>,
            Array<KStream<String, AggregatedTelemetryData>>> {
        return java.util.function.Function<
                KStream<String, TelemetryDataPoint>,
                Array<KStream<String, AggregatedTelemetryData>>> { telemetryRecords ->
            telemetryRecords.branch(
                // Split up the processing pipeline into 2 streams, depending on the space agency of the probe
                Predicate { _, v -> v.spaceAgency == SpaceAgency.NASA },
                Predicate { _, v -> v.spaceAgency == SpaceAgency.ESA }
            ).map { telemetryRecordsPerAgency ->
                // Apply aggregation logic on each stream separately
                telemetryRecordsPerAgency
                    .groupByKey()
                    .aggregate(
                        // KTable initializer
                        { AggregatedTelemetryData(maxSpeedMph = 0.0, traveledDistanceFeet = 0.0) },
                        // Calculation function for telemetry data aggregation
                        { probeId, lastTelemetryReading, aggregatedTelemetryData ->
                            updateTotals(
                                probeId,
                                lastTelemetryReading,
                                aggregatedTelemetryData
                            )
                        },
                        // Configure Serdes for State Store topic
                        Materialized.with(Serdes.StringSerde(), AggregateTelemetryDataSerde())
                    )
                    .toStream()
            }.toTypedArray()
        }
    }

    /**
     * Performs calculation of per-probe aggregate measurement data.
     * The currently calculated totals are held in a Kafka State Store
     * backing the KTable created with aggregate() and the most recently
     * created aggregate telemetry data record is passed on downstream.
     */
    fun updateTotals(
        probeId: String,
        lastTelemetryReading: TelemetryDataPoint,
        currentAggregatedValue: AggregatedTelemetryData
    ): AggregatedTelemetryData {
        val totalDistanceTraveled =
            lastTelemetryReading.traveledDistanceFeet + currentAggregatedValue.traveledDistanceFeet
        val maxSpeed = if (lastTelemetryReading.currentSpeedMph > currentAggregatedValue.maxSpeedMph)
            lastTelemetryReading.currentSpeedMph else currentAggregatedValue.maxSpeedMph
        val aggregatedTelemetryData = AggregatedTelemetryData(
            traveledDistanceFeet = totalDistanceTraveled,
            maxSpeedMph = maxSpeed
        )
        logger.info {
            "Calculated new aggregated telemetry data for probe $probeId. New max speed: ${aggregatedTelemetryData.maxSpeedMph} and " +
                    "traveled distance ${aggregatedTelemetryData.traveledDistanceFeet}"

        }
        return aggregatedTelemetryData
    }
}
```

In order to calculate the aggregated telemetry data per probe, we have to solve three problems in the implementation of the function:

#### Implementation of the calculation

The Kafka Streams API provides a set of predefined operations that we can apply to the incoming stream to perform the computation according to the use case. Since we need to separate the aggregated probe data by space agency, we first use the operation, which returns `branch()`an array of as a result . `KStream`We get two streams, which are now already separated by space agency. The probe data can now be calculated. Since the calculation is identical for both agencies, we use `map()`Kotlin's operation so that we only have to define the following steps once for both streams. To group and ultimately aggregate the probe data by their Probe ID, we again use native Stream API operations. The operation[`aggregate()`](https://docs.confluent.io/platform/current/streams/concepts.html#aggregations) needs three parameters:

-   An **Initializer** that determines the initial value if there is no aggregated data yet
-   An **Aggregator** function that determines our calculation logic for aggregating the metrics
-   The **serializers/deserializers** to use to store the aggregated values

`aggregate()`As a result, the operation returns one that `KTable`contains the most recently calculated total value for each probe ID. Since our customers are interested in the most up-to-date data, we convert it `KTable`back into one `KStream`\- every change in the `KTable`generates an event that contains the last calculated total value.

#### Storage of the aggregated measurement data

The `aggregate()`function that we use in our example to calculate the total values is a so-called _stateful operation_ - i.e. a stateful operation that requires a local state in order to be able to take into account all previously calculated values for calculating the currently valid total value. The Kafka Streams API handles the management of state for us by `KTable`materializing the operation a . One `KTable`can be thought of as a changelog for key/value pairs, which allows us to persist (and restore if necessary) a local state. `KTables`are in the Kafka cluster by so-called _state stores_materialized. The state store uses a topic managed by Kafka to persist data in the cluster. This saves us from having to manage other infrastructure components, such as a database to keep track of the status.

#### Delivery to the various space agencies

In order to supply NASA and ESA with the aggregated measurement data relevant to them, we used the operation before the calculation `branch()`. As a result, our function has a return value of `Array>`, whose indices correlate with the space agencies. In our case, this means that `Array[0]`the data is from NASA and `Array[1]`the data from ESA. This division in turn matches our binding config.

The resulting KStreams are our aggregation result and are written to the two output topics.

### Consume the telemetry data (kafka-samples-consumer)

To read the aggregated probe measurement data, we use the Kafka Consumer API, which, like the Producer, is available via the[Spring Cloud Stream Binder for Kafka](https://cloud.spring.io/spring-cloud-stream-binder-kafka/spring-cloud-stream-binder-kafka.html#_apache_kafka_binder) provided. We configure the service for this as follows ([application.yml](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-consumer/src/main/resources/application.yml)):

```
spring:
  application:
    name: kafka-telemetry-data-consumer
  # Ignore type headers in kafka message
  kafka.properties.spring.json.use.type.headers: false
  cloud:
    stream:
      kafka:
        binder:
          brokers: "localhost:29092"
        bindings:
          # this has to match the consumer bean name with suffix in-0 (for consumer)
          processNasaTelemetryData-in-0:
            consumer:
              configuration:
                key.deserializer: org.apache.kafka.common.serialization.StringDeserializer
                value.deserializer: de.codecentric.samples.kafkasamplesconsumer.serdes.TelemetryDataDeserializer
          processEsaTelemetryData-in-0:
            consumer:
              configuration:
                key.deserializer: org.apache.kafka.common.serialization.StringDeserializer
                value.deserializer: de.codecentric.samples.kafkasamplesconsumer.serdes.TelemetryDataDeserializer
      bindings:
        processNasaTelemetryData-in-0:
          group: ${spring.application.name}
          destination: space-probe-aggregate-telemetry-data-nasa
        processEsaTelemetryData-in-0:
          group: ${spring.application.name}
          destination: space-probe-aggregate-telemetry-data-esa
    function:
      # We define this explicitly since we have several consumer functions
      definition: processNasaTelemetryData;processEsaTelemetryData
```

We continue according to the familiar pattern: We implement beans that implement the binding, starting with NASA.

We create a function that `Consumer`implements (see[KafkaConsumerConfiguration.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-consumer/src/main/kotlin/de/codecentric/samples/kafkasamplesconsumer/KafkaConsumerConfiguration.kt)). This means that we have an input but no output to another topic and the stream ends in this function.

The consumer for ESA follows the same pattern as the consumer for NASA, the only difference being that the data transmitted is converted from the imperial system to the metric system. We have this functionality in the `init`function of our class[MetricTelemetryData.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-consumer/src/main/kotlin/de/codecentric/samples/kafkasamplesconsumer/event/MetricTelemetryData.kt)  capsuled.

With the implementation of the consumer, our stream processing pipeline is complete and all requirements have been implemented.

## The finished solution in action

If we start our services now, after a few seconds we should see the first aggregated probe data in the Consumer Service log. Additionally, we can take a look at[AKHQ](https://github.com/tchiotludo/akhq) Get an overview of the topics and messages in Kafka:

![AKHQ Web UI in Kafka Topics view](https://blog.codecentric.de/_next/image?url=https%3A%2F%2Fmedia.graphassets.com%2FK3ygZt4RvWW2Evm17u2w&w=2048&q=75)  
We recognize the inbound and outbound topics accessed by our services, as well as the state stores that our aggregator service has materialized behind the scenes for us in the form of Kafka topics.

## Lessons learned

If you are now thinking about using the whole thing in your projects, we have prepared some questions and argumentation aids for you so that you can make the right decision for you.

### When should you think about using stream processing?

Stream processing can be useful wherever you are faced with processing large amounts of data and time delays need to be minimized. The decision for or against stream processing should not be based on a single component - the solution should fit the overall architecture of the system and the problem. If your use case has the following attributes, stream processing could be a solution:

-   You are faced with a constant stream of data. Example: IoT devices continuously send you sensor data.
-   Your workload is continuous and does not have the character of a data delivery. Example: You receive a data export from an old system once a day and must be able to definitely determine the end of a delivery, for example. Batch processing usually makes more sense here.
-   The data to be processed are time-critical and must be processed immediately. In the case of larger amounts of data or complex calculations, you always have to think about the scalability of your services in order to keep the processing time low. Streams are suitable for this because you can scale horizontally quite easily through partitioning and asynchronous event processing.

### Should you use Kafka for your stream processing workloads?

We give a cautious _“yes”_ to that . Kafka's data storage, paired with the Stream Processing API, is a powerful tool and is offered by various providers _as a service_ . This flattens the learning curve and minimizes maintenance. In event-driven use cases, this feels good and right. Unfortunately, we have seen several times that Kafka is used in application architectures as a pure message bus, for batch workloads or in situations where synchronous communication between services would have made more sense. The advantages of Kafka and event-driven architectures remain unused or worse: we find it more difficult than necessary to solve the problem.

If Kafka is already present in your architecture and your problem fits the technology, we would recommend you to start with the[Stream API capabilities](https://docs.confluent.io/platform/current/streams/concepts.html#stream-processing-application) to deal with - data pipelines can often be set up without additional infrastructure components and you can do without components such as relational databases or in-memory data stores. Confluent offers very good[Documents to get you started](https://www.confluent.io/blog/how-kafka-streams-works-guide-to-stream-processing/) an.

In cases where you cannot use Kafka _as a service_ , the effort involved in setting up a Kafka cluster and operating it yourself can outweigh the benefits. In these cases, it may therefore make more sense to use a classic message broker and a relational database.

### Should you implement stream processing workloads with Kafka Streams and Spring Boot?

Clear answer: It depends. If you are already using Spring Boot across the board in your projects, the[Spring Streams abstraction](https://spring.io/projects/spring-cloud-stream) save some time when commissioning new services, since configuration and implementation always follow a very similar scheme and we can hide some of the complexity during implementation. However, the Spring path is not quite perfect. Here are the issues that caused us pain:

-   **Conventions & Documentation** : The configuration with the Spring abstraction consists of a few conventions that are not always properly documented and are sometimes non-transparent, which can cost nerves and time. At the time of writing this article, parts of the Spring documentation were out of date (e.g. the functional programming paradigm we are using is not yet mentioned in the current version of the documentation)
-   **Error Handling** : When using the Stream Binder for Kafka Streams as in our class[KafkaStreamsHandler.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/main/kafka-samples-streams/src/main/kotlin/com/example/kafkasamplesstreams/KafkaStreamsHandler.kt) There is currently no convenient solution for handling exceptions that occur outside of deserialization using on-board tools (we define what should happen to errors during deserialization in[application.yml](https://github.com/codecentric/spring-kafka-streams-example/blob/39e6482ecbc0ecaefa228539e3582f40425fb1aa/kafka-samples-streams/src/main/resources/application.yml#L24)). The only solution for this at the moment is[to implement error handling past the Streams API](https://cloud.spring.io/spring-cloud-stream-binder-kafka/spring-cloud-stream-binder-kafka.html#_handling_non_deserialization_exceptions) or ensure that any deserialization errors are caught. Provides an exemplary approach[TelemetryAggregationTransformer.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/94c4df005e7dbe820e9d16a260962f61980639fc/kafka-samples-streams/src/main/kotlin/com/example/kafkasamplesstreams/TelemetryAggregationTransformer.kt). By bypassing the Streams API, we can handle errors at the message level, for example by `try/catch`implementing logic. Since we have descended an abstraction level in this example, we unfortunately also lose the automatic state management `KTables`\- we have to manage state stores ourselves if necessary. In this case, unfortunately, you currently have to weigh up what is more important to you.
-   **Up-to- dateness** : The Spring Dependencies are always a few Kafka releases behind, so that not all features can always be used immediately (see previous point).

As an alternative to the Spring abstraction, there are various freely usable libraries to integrate the concepts from Kafka Streams into various tech stacks. Confluent offers[well-documented step-by-step recipes](https://developer.confluent.io/kafka-languages-and-tools/) for a wide range of supported environments and programming languages to keep the barriers to entry low, regardless of your environment. In this respect, you are free to decide here. If you feel comfortable with Spring Boot: great! If not: that's ok too!

## A few final words

In this blog post, we have demonstrated how you can implement the concepts of stream processing using a concrete use case with Spring Boot and Kafka Streams. We hope that with Stream Processing you now have another tool in your toolbox and that you can now approach your next project with complete peace of mind.

You can find the complete code for our sample project at[GitHub](https://github.com/codecentric/spring-kafka-streams-example).

## bonus material

In order not to go beyond the scope of our blog post, we have limited ourselves to a fairly simple use case. However `KTables`, much more demanding scenarios can also be implemented with . Another slightly more complex example (how do we merge multiple incoming streams?) can be found in our GitHub repo on a[separate branch](https://github.com/codecentric/spring-kafka-streams-example/tree/feature/demonstrate-ktable-joins). We combine the incoming streams in the class[KafkaStreamsHandler.kt](https://github.com/codecentric/spring-kafka-streams-example/blob/feature/demonstrate-ktable-joins/kafka-samples-streams/src/main/kotlin/com/example/kafkasamplesstreams/KafkaStreamsHandler.kt) using the `join()`operation.

## credentials

\[1\] Ben Stopford (2018):[Designing Event Driven Systems](https://www.confluent.io/designing-event-driven-systems/) , S. 29 ff.

\[2\] Martin Kleppmann (2015): [Turning the Database inside-out with Apache Samza](https://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/)

\[3\] Apache Software Foundation (2017): [Kafka Streams Core Concepts](https://kafka.apache.org/30/documentation/streams/core-concepts)

\[4\] Apache Software Foundation (2017): [Kafka Main Concepts and Terminology](https://kafka.apache.org/documentation/#intro_concepts_and_terms)