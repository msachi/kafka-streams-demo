## Build a demo app with Kafka Streams

In this tutorial, we will write a stream processing application using Kafka Streams and then run it with a simple Kafka producer and consumer.

The application we will be building implements a word count algorithm, which computes a word occurrence histogram from the input text. It has the ability to operate on an infinite, unbounded stream of data.

This tutorial is more or less a prettified version of the official Kafka Streams offerings found [here](https://kafka.apache.org/25/documentation/streams/quickstart) and [here](https://kafka.apache.org/25/documentation/streams/tutorial).

### 1. Setting up the project

First, make sure that you have:
- JDK 8 and Apache Maven 3.6 installed on your machine (check using $ java –version and $ mvn –version)
- Kafka 2.4.1 [downloaded](https://mirrors.ukfast.co.uk/sites/ftp.apache.org/kafka/2.4.1/kafka_2.12-2.4.1.tgz) and un-tarred

Then clone this repo and import the project in your favourite IDE.

The `pom.xml` file included in the project already has the Streams dependency defined. Note that the generated `pom.xml` targets Java 8 and does not work with higher Java versions.

### 2 Setting up Kafka

Before we start writing our application, let's set up all things Kafka.

Navigate to the Kafka source on your computer and run a Zookeeper and a Kafka server:

```bash
$ bin/zookeeper-server-start.sh config/zookeeper.properties
$ bin/kafka-server-start.sh config/server.properties
```

Next, let's create the input topic that we will read from:
```bash
$ bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-plaintext-input
```

and a few output topics that we will write to:
```bash
> bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-pipe-output \
--config cleanup.policy=compact
```

```bash
> bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-linesplit-output \
--config cleanup.policy=compact
```

```bash
> bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-wordcount-output \
--config cleanup.policy=compact
```

Note: we create the output topic with compaction enabled because the output stream is a changelog stream.

You can inspect the newly created topics as follows:
```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

and should see all topics listed, with the expected partition counts and replication factors, and the assigned brokers:

```bash
Topic:streams-linesplit-output PartitionCount:1 ReplicationFactor:1 Configs:cleanup.policy=compact,segment.bytes=1073741824
	Topic: streams-linesplit-output Partition: 0 Leader: 0 Replicas: 0 Isr: 0
Topic:streams-pipe-output PartitionCount:1 ReplicationFactor:1 Configs:cleanup.policy=compact,segment.bytes=1073741824
	Topic: streams-pipe-output Partition: 0 Leader: 0 Replicas: 0 Isr: 0
Topic:streams-plaintext-input PartitionCount:1 ReplicationFactor:1 Configs:segment.bytes=1073741824
	Topic: streams-plaintext-input Partition: 0 Leader: 0 Replicas: 0 Isr: 0
Topic:streams-wordcount-output PartitionCount:1 ReplicationFactor:1 Configs:cleanup.policy=compact,segment.bytes=1073741824
	Topic: streams-wordcount-output Partition: 0 Leader: 0 Replicas: 0 Isr: 0
```

Finally, we will need a console producer to be able to write some data to the input topic:

```bash
$ bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
```

We will set up the console producers as we progress through the tutorial.

### 3. Piping data

#### 3.1 Configuring the application

We're now ready to start using Kafka Streams! In this part of the tutorial, we will start with piping some data from the input topic to the output topic.

Navigate to `myapps/Pipe.java`:

```java
package myapps;

public class Pipe {

  public static void main(String[] args) throws Exception {

  }
}
```

We are going to fill in the `main` function to write this pipe program. Your IDE should be able to add the import statements automatically.

The first step to write a Streams application is to create a `java.util.Properties` map to specify different Streams execution configuration values as defined in `StreamsConfig`. A couple of important configuration values you need to set are: `StreamsConfig.BOOTSTRAP_SERVERS_CONFIG`, which specifies a list of host/port pairs to use for establishing the initial connection to the Kafka cluster, and `StreamsConfig.APPLICATION_ID_CONFIG`, which gives the unique identifier of your Streams application to distinguish itself with other applications talking to the same Kafka cluster:

```java
Properties props = new Properties();  
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-pipe");  
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
```

In addition, you can customise other configurations in the same map, for example, default serialisation and deserialisation libraries for the record key-value pairs:

```java
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
```

#### 3.2 Defining the topology

Next we will define the computational logic of our Streams application. In Kafka Streams this computational logic is defined as a `topology` of connected processor nodes. We can use a topology builder to construct such a topology,

```java
final StreamsBuilder builder = new StreamsBuilder();
```

And then create a source stream from a Kafka topic named `streams-plaintext-input` using this topology builder:

```java
KStream<String, String> source = builder.stream("streams-plaintext-input");
```

Now we get a `KStream` that is continuously generating records from its source Kafka topic `streams-plaintext-input`. The records are organized as `String` typed key-value pairs. The simplest thing we can do with this stream is to write it into another Kafka topic, say it's named `streams-pipe-output`:

```java
source.to("streams-pipe-output");
```

Note that we can also concatenate the above two lines into a single line as:

```java
builder.stream("streams-plaintext-input").to("streams-pipe-output");
```

We can inspect what kind of `topology` is created from this builder by doing the following:

```java
final  Topology topology = builder.build();
```

And print its description to standard output as:

```java
System.out.println(topology.describe());
```

If we just stop here, compile and run the program,

```bash
mvn clean package
mvn exec:java -Dexec.mainClass=myapps.Pipe
```

it will output the following information:

```bash
Sub-topologies:
  Sub-topology: 0
    Source: KSTREAM-SOURCE-0000000000(topics: streams-plaintext-input) --> KSTREAM-SINK-0000000001
    Sink: KSTREAM-SINK-0000000001(topic: streams-pipe-output) <-- KSTREAM-SOURCE-0000000000
Global Stores:
  none
```

As shown above, it illustrates that the constructed topology has two processor nodes, a source node `KSTREAM-SOURCE-0000000000` and a sink node `KSTREAM-SINK-0000000001`. `KSTREAM-SOURCE-0000000000` continuously read records from Kafka topic `streams-plaintext-input` and pipe them to its downstream node `KSTREAM-SINK-0000000001`; `KSTREAM-SINK-0000000001` will write each of its received record in order to another Kafka topic `streams-pipe-output` (the `-->` and `<--` arrows dictates the downstream and upstream processor nodes of this node, i.e. "children" and "parents" within the topology graph). It also illustrates that this simple topology has no global state stores associated with it (we will talk about state stores more in the following sections).

Note that we can always describe the topology as we did above at any given point while we are building it in the code, so as a user you can interactively "try and taste" your computational logic defined in the topology until you are happy with it.

#### 3.3 Constructing the Streams client

Suppose we are already done with this simple topology that just pipes data from one Kafka topic to another in an endless streaming manner, we can now construct the Streams client with the two components we have just constructed above: the configuration map specified in a `java.util.Properties` instance and the `Topology` object.

```java
final KafkaStreams streams = new KafkaStreams(topology, props);
```

By calling its `start()` function we can trigger the execution of this client. The execution won't stop until `close()` is called on this client. We can, for example, add a shutdown hook with a countdown latch to capture a user interrupt and close the client upon terminating this program:

```java
final CountDownLatch latch = new CountDownLatch(1);  
  
// attach shutdown handler to catch control-c  
Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook") {
  @Override  
  public void run() {  
    streams.close();  
    latch.countDown();  
  }  
});  
  
try {  
  streams.start();  
  latch.await();  
} catch (Throwable e) {  
  System.exit(1);  
}  
System.exit(0);
```

The complete code so far looks like this:

```java
package myapps;  
  
import org.apache.kafka.common.serialization.Serdes;  
import org.apache.kafka.streams.KafkaStreams;  
import org.apache.kafka.streams.StreamsBuilder;  
import org.apache.kafka.streams.StreamsConfig;  
import org.apache.kafka.streams.Topology;  
  
import java.util.Properties;  
import java.util.concurrent.CountDownLatch;  
  
public class Pipe {  
  
  public static void main(String[] args) throws Exception {  
  Properties props = new Properties();  
  props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-pipe");  
  props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");  
  props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
  props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
  
  final StreamsBuilder builder = new StreamsBuilder();  
  
  builder.stream("streams-plaintext-input").to("streams-pipe-output");  
  
  final Topology topology = builder.build();  
  final KafkaStreams streams = new KafkaStreams(topology, props);  
  final CountDownLatch latch = new CountDownLatch(1);  
  
  // attach shutdown handler to catch control-c  
  Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook") {  
    @Override  
	public void run() {
	  streams.close();  
	  latch.countDown();  
	}  
  });  
  
  try {
    streams.start();
    latch.await();  
  } catch (Throwable e) {
    System.exit(1);  
  }
  System.exit(0);  
  }
}
```

#### 3.4 Starting the application

You should now be able to run your application code in the IDE or on the command line, using Maven:

```bash
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=myapps.Pipe
```

The application will read from the input topic `streams-plaintext-input` and continuously write to the output topic `streams-pipe-output`.

We already have a console producer set up that we can use to write some data to the input topic. Let's also set up a console consumer, subscribed to the output topic `streams-pipe-output`, to be able to inspect our application output:

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic streams-pipe-output \
--from-beginning \
--formatter kafka.tools.DefaultMessageFormatter \
--property print.key=true \
--property print.value=true \
--property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
--property value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

Let's write a message with the console producer into the input topic `streams-plaintext-input` by entering a single line of text and then hit <RETURN>. This will send a new message to the input topic, where the message key is null and the message value is the string encoded text line that you just entered (in practice, input data for applications will typically be streaming continuously into Kafka, rather than being manually entered as we do in this demo):

```
I am free and that is why I am lost
```

This message will be piped by the application and the same data will be written to the `streams-pipe-output` topic and printed by the console consumer:

```
I am free and that is why I am lost
```

That's great, our application is reading from a topic, piping the data and writing to another topic!

### 4. Splitting lines into words

Now let's move on to add some real processing logic by augmenting the current topology. We can first create another program by copying the existing `Pipe.java` class:

```bash
cp src/main/java/myapps/Pipe.java src/main/java/myapps/LineSplit.java
```

We also need to change its class name and application id config to distinguish it from the original program:
```java
public class LineSplit {
 
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-linesplit");
        // ...
    }
}
```

#### 4.1 Adding the logic

Since each of the source stream's record is a `String` typed key-value pair, let's treat the value string as a text line and split it into words with a `flatMapValues` operator:

```java
KStream<String, String> source = builder.stream("streams-plaintext-input");
KStream<String, String> words = source.flatMapValues(value -> Arrays.asList(value.split("\\W+")));
```

The operator will take the source stream as its input, and generate a new stream named words by processing each record from its source stream in order and breaking its value string into a list of words, and producing each word as a new record to the output words stream. This is a stateless operator that does not need to keep track of any previously received records or processed results.

And finally we can write the word stream back into another Kafka topic, `streams-linesplit-output`:

```java
KStream<String, String> source = builder.stream("streams-plaintext-input");
source.flatMapValues(value -> Arrays.asList(value.split("\\W+")))
      .to("streams-linesplit-output");
```

If we now describe this augmented topology as `System.out.println(topology.describe())`, we will get the following:

```brew
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=myapps.LineSplit
Sub-topologies:
  Sub-topology: 0
    Source: KSTREAM-SOURCE-0000000000(topics: streams-plaintext-input) --> KSTREAM-FLATMAPVALUES-0000000001
    Processor: KSTREAM-FLATMAPVALUES-0000000001(stores: []) --> KSTREAM-SINK-0000000002 <-- KSTREAM-SOURCE-0000000000
    Sink: KSTREAM-SINK-0000000002(topic: streams-linesplit-output) <-- KSTREAM-FLATMAPVALUES-0000000001
  Global Stores:
    none
```

As we can see above, a new processor node `KSTREAM-FLATMAPVALUES-0000000001` is injected into the topology between the original source and sink nodes. It takes the source node as its parent and the sink node as its child. In other words, each record fetched by the source node will first traverse to the newly added `KSTREAM-FLATMAPVALUES-0000000001` node to be processed, and one or more new records will be generated as a result. They will continue traverse down to the sink node to be written back to Kafka. Note this processor node is "stateless" as it is not associated with any stores (i.e. (`stores: []`)).

The complete code looks like this (assuming lambda expression is used):

```java
package myapps;
 
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.KStream;
 
import java.util.Arrays;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;
 
public class LineSplit {
 
  public static void main(String[] args) throws Exception {
    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-linesplit");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

    final StreamsBuilder builder = new StreamsBuilder();

    KStream<String, String> source = builder.stream("streams-plaintext-input");
    source.flatMapValues(value -> Arrays.asList(value.split("\\W+")))
          .to("streams-linesplit-output");

    final Topology topology = builder.build();
    final KafkaStreams streams = new KafkaStreams(topology, props);

    // ... same as Pipe.java above
  }
}
```

#### 4.2 Running the application

As before, we can run the application code in the IDE or on the command line, using Maven:

```bash
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=myapps.LineSplit
```

We will need to set up a new console consumer, subscribed to the output topic `streams-linesplit-output`:

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic streams-linesplit-output \
--from-beginning \
--formatter kafka.tools.DefaultMessageFormatter \
--property print.key=true \
--property print.value=true \
--property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
--property value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

Let's write a message with the console producer into the input topic `streams-plaintext-input`:

```
There is an infinite amount of hope in the universe ... but not for us
```

This message will be processed by the application and the word stream will be written to the `streams-linesplit-output` topic and printed by the console consumer:

```
there
is
an
infinite
amount
of
hope
in
the
universe
but
not
for
us
```

### 5. Counting the words

Let's now take a step further to add some "stateful" computations to the topology by counting the occurrence of the words split from the source text stream. Following similar steps let's create another program based on the `LineSplit.java` class:

```java
public class WordCount {
 
  public static void main(String[] args) throws Exception {
    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-wordcount");
    // ...
  }
}
```

#### 5.1 Adding the logic

In order to count the words we can first modify the `flatMapValues` operator to treat all of them as lower case:

```java
KStream<String, String> words = source.flatMapValues(value -> Arrays.asList(value.toLowerCase(Locale.getDefault()).split("\\W+")));
```

In order to do the counting aggregation we have to first specify that we want to key the stream on the value string, i.e. the lower cased word, with a `groupBy` operator. This operator generate a new grouped stream, which can then be aggregated by a `count` operator, which generates a running count on each of the grouped keys:

```java
KTable<String, Long> counts = words.groupBy((key, value) -> value)
  // Materialize the result into a KeyValueStore named "counts-store".
  // The Materialized store is always of type <Bytes, byte[]> as this is the format of the inner most store.
  .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store"));
```

Note that the count operator has a Materialized parameter that specifies that the running count should be stored in a state store named counts-store. This Counts store can be queried in real-time.

We can also write the `counts` KTable's changelog stream back into another Kafka topic, say `streams-wordcount-output`. Because the result is a changelog stream, the output topic `streams-wordcount-output` should be configured with log compaction enabled. Note that this time the value type is no longer `String` but `Long`, so the default serialization classes are not viable for writing it to Kafka anymore. We need to provide overridden serialization methods for `Long` types, otherwise a runtime exception will be thrown:

```java
counts.toStream().to("streams-wordcount-output", Produced.with(Serdes.String(), Serdes.Long()));
```

Note that in order to read the changelog stream from topic `streams-wordcount-output`, one needs to set the value deserialization as `org.apache.kafka.common.serialization.LongDeserializer`.

The above code can be simplified as:

```java
KStream<String, String> source = builder.stream("streams-plaintext-input");
source.flatMapValues(value -> Arrays.asList(value.toLowerCase(Locale.getDefault()).split("\\W+")))
  .groupBy((key, value) -> value)
  .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store"))
  .toStream()
  .to("streams-wordcount-output", Produced.with(Serdes.String(), Serdes.Long()));
```

If we again describe this augmented topology as `System.out.println(topology.describe())`, we will get the following:

```java
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=myapps.WordCount
Sub-topologies:
  Sub-topology: 0
    Source: KSTREAM-SOURCE-0000000000(topics: streams-plaintext-input) --> KSTREAM-FLATMAPVALUES-0000000001
    Processor: KSTREAM-FLATMAPVALUES-0000000001(stores: []) --> KSTREAM-KEY-SELECT-0000000002 <-- KSTREAM-SOURCE-0000000000
    Processor: KSTREAM-KEY-SELECT-0000000002(stores: []) --> KSTREAM-FILTER-0000000005 <-- KSTREAM-FLATMAPVALUES-0000000001
    Processor: KSTREAM-FILTER-0000000005(stores: []) --> KSTREAM-SINK-0000000004 <-- KSTREAM-KEY-SELECT-0000000002
    Sink: KSTREAM-SINK-0000000004(topic: Counts-repartition) <-- KSTREAM-FILTER-0000000005
  Sub-topology: 1
    Source: KSTREAM-SOURCE-0000000006(topics: Counts-repartition) --> KSTREAM-AGGREGATE-0000000003
    Processor: KSTREAM-AGGREGATE-0000000003(stores: [Counts]) --> KTABLE-TOSTREAM-0000000007 <-- KSTREAM-SOURCE-0000000006
    Processor: KTABLE-TOSTREAM-0000000007(stores: []) --> KSTREAM-SINK-0000000008 <-- KSTREAM-AGGREGATE-0000000003
    Sink: KSTREAM-SINK-0000000008(topic: streams-wordcount-output) <-- KTABLE-TOSTREAM-0000000007
Global Stores:
  none
```

As we can see above, the topology now contains two disconnected sub-topologies. The first sub-topology's sink node KSTREAM-SINK-0000000004 will write to a repartition topic Counts-repartition, which will be read by the second sub-topology's source node KSTREAM-SOURCE-0000000006. The repartition topic is used to "shuffle" the source stream by its aggregation key, which is in this case the value string. In addition, inside the first sub-topology a stateless KSTREAM-FILTER-0000000005 node is injected between the grouping KSTREAM-KEY-SELECT-0000000002 node and the sink node to filter out any intermediate record whose aggregate key is empty.

In the second sub-topology, the aggregation node `KSTREAM-AGGREGATE-0000000003` is associated with a state store named `Counts` (the name is specified by the user in the `count` operator). Upon receiving each record from its upcoming stream source node, the aggregation processor will first query its associated `Counts` store to get the current count for that key, augment by one, and then write the new count back to the store. Each updated count for the key will also be piped downstream to the `KTABLE-TOSTREAM-0000000007` node, which interpret this update stream as a record stream before further piping to the sink node `KSTREAM-SINK-0000000008` for writing back to Kafka.

For reference, the complete code should look like this:

```java
package myapps;
 
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.KStream;
 
import java.util.Arrays;
import java.util.Locale;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;
 
public class WordCount {
 
  public static void main(String[] args) throws Exception {
    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-wordcount");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

    final StreamsBuilder builder = new StreamsBuilder();

    KStream<String, String> source = builder.stream("streams-plaintext-input");
    source.flatMapValues(value -> Arrays.asList(value.toLowerCase(Locale.getDefault()).split("\\W+")))
          .groupBy((key, value) -> value)
          .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store"))
          .toStream()
          .to("streams-wordcount-output", Produced.with(Serdes.String(), Serdes.Long()));

    final Topology topology = builder.build();
    final KafkaStreams streams = new KafkaStreams(topology, props);
    final CountDownLatch latch = new CountDownLatch(1);

    // ... same as Pipe.java above
  }
}
```

#### 5.2 Running the application

As before, we can run the application code in the IDE or on the command line, using Maven:

```bash
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=myapps.WordCount
```

Again, we need to set up a new console consumer, subscribed to the output topic `streams-wordcount-output`:

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic streams-wordcount-output \
--from-beginning \
--formatter kafka.tools.DefaultMessageFormatter \
--property print.key=true \
--property print.value=true \
--property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
--property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

Let's write a message with the console producer into the input topic `streams-plaintext-input`:

```
A book is a narcotic
```

This message will be processed by the application and the word count stream will be written to the `streams-wordcount-output` topic and printed by the console consumer:

```
a           1
book        1
is          1
a           2
narcotic    1
```

Here, the first column is the Kafka message key in `java.lang.String` format and represents a word that is being counted, and the second column is the message value in `java.lang.Longformat`, representing the word's latest count.

Notice that the word `a` first appears with a count of 1, and is later on updated to the count of 2.

Let's enter another text line:

```
A book must be the axe for the frozen sea within us
```

and we should see the following output printed below the previous lines:

```
a       3
book    2
must    1
be      1
the     1
axe     1
for     1
the     1
frozen  1
sea     1
within  1
us      1
```

Notice that the words `a` and `book` have been incremented from 2 to 3, and from 1 to 2, respectively. The rest of the new words have also been printed with the word count of 1, but the words from previous lines have not been printed again.

With another line:
```
Many a book is like a key to unknown chambers within the castle of one’s own self
```

the output would be:

```
many        1
a           4
book        3
is          1
like        1
a           5
key         1
to          1
unknown     1
chambers    1
within      1
the         2
castle      1
of          1
one's       1
own         1
self        1
```

As one can see, the output of the word count application is actually a continuous stream of updates, where each output record (i.e. each line in the original output above) is an updated count of a single word, aka record key such as "book". For multiple records with the same key, each later record is an update of the previous one.

#### 5.3 Reflecting on stream processing

The diagram below illustrates what is essentially happening behind the scenes. The first column shows the word stream `KStream<String, String>` that results from the incoming stream of text lines. The second column shows the evolution of the current state of the `KTable<String, Long>` that is counting word occurrences for count. The second column shows the change records that result from state updates to the `KTable` and that are being sent to the output Kafka topic `streams-wordcount-output`.

As the first few words are being processed, the `KTable` is being built up as each new word results in a new table entry (highlighted with a green background), and a corresponding change record is sent to the downstream `KStream`.

Then, when words start repeating, existing entries in the KTable start being updated. And again, change records are being sent to the output topic.

Looking beyond the scope of this concrete example, what Kafka Streams is doing here is to leverage the duality between a table and a changelog stream (here: table = the `KTable`, changelog stream = the downstream `KStream`): you can publish every change of the table to a stream, and if you consume the entire changelog stream from beginning to end, you can reconstruct the contents of the table.

### 6. Tearing down the application

You can now stop the console consumer, the console producer, your word count application, the Kafka broker and the ZooKeeper server (in this order) via `Cmd-C` (or `Ctrl-C`).