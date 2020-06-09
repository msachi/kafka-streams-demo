## Build a demo app with Kafka Streams

In the following tutorial, you will write a stream processing application using Kafka Streams and then run it in your terminal with a simple Kafka producer and consumer. The application you will be building implements the WordCount algorithm, which computes a word occurrence histogram from the input text. It has the ability to operate on an infinite, unbounded stream of data.

This tutorial is more or less a prettified version of the official Kafka Streams offerings found [here](https://kafka.apache.org/25/documentation/streams/quickstart) and [here](https://kafka.apache.org/25/documentation/streams/tutorial).

### 1. Setting up the project

First, make sure that you have:
- JDK 8 and Apache Maven 3.6 installed on your machine (check using $ java –version and $ mvn –version)
- Kafka 2.4.1 [downloaded](https://mirrors.ukfast.co.uk/sites/ftp.apache.org/kafka/2.4.1/kafka_2.12-2.4.1.tgz) and un-tarred

Then clone this repo and import the project in your favourite IDE.

The `pom.xml` file included in the project already has the Streams dependency defined. Note that the generated `pom.xml` targets Java 8 and does not work with higher Java versions.

#### 2 Setting up Kafka

Before we start writing our application, let's set up all things Kafka.

Navigate to the Kafka source on your computer and run a Zookeeper and a Kafka server:

```bash
$ bin/zookeeper-server-start.sh config/zookeeper.properties
$ bin/kafka-server-start.sh config/server.properties
```

Next, let's create the input and the output topics that we will read from and write to:
```bash
$ bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-plaintext-input
```

```bash
> bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic streams-plaintext-output \
--config cleanup.policy=compact
```

Note: we create the output topic with compaction enabled because the output stream is a changelog stream.

You can inspect the newly created topics as follows:
```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

and should see both topics listed, with the expected partition counts and replication factors, and the assigned brokers:

```bash
Topic:streams-wordcount-output PartitionCount:1 ReplicationFactor:1 Configs:cleanup.policy=compact,segment.bytes=1073741824
	Topic: streams-wordcount-output Partition: 0 Leader: 0 Replicas: 0 Isr: 0
Topic:streams-plaintext-input PartitionCount:1 ReplicationFactor:1 Configs:segment.bytes=1073741824
	Topic: streams-plaintext-input Partition: 0 Leader: 0 Replicas: 0 Isr: 0
```

Finally, we will need a console producer to be able to write some data to the input topic,

```bash
$ bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
```

and a console consumer, subscribed to the output topic, to be able to inspect our application output:

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

The application will read from the input topic `streams-plaintext-input` and continuously write to the output topic `streams-plaintext-output`. 

Let's write some message with the console producer into the input topic `streams-plaintext-input` by entering a single line of text and then hit <RETURN>. This will send a new message to the input topic, where the message key is null and the message value is the string encoded text line that you just entered (in practice, input data for applications will typically be streaming continuously into Kafka, rather than being manually entered as we do in this demo):

```bash
all streams lead to kafka
```

This message will be processed by the application and the same data will be written to the `streams-plaintext-output` topic and printed by the console consumer:

```bash
all streams lead to kafka
```

That's cool but not very exciting in terms of data processing -- let's move on to the next step to actually start implementing the WordCount algorithm.