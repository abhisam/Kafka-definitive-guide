# Kafka the definitive guide

## Table of contents:
1. [Chapter 1: Meet Kafka](#Chapter1)
2. [Chapter 2: Installing Kafka](#Chapter2)
3. [Chapter 3: Kafka Producers: Writing Messages to Kafka](#Chapter3)
4. [Chapter 4: Kafka Consumers: Reading Data from Kafka](#Chapter4)
5. [Chapter 5: Kafka Internals](#Chapter5)
6. [Chapter 6: Reliable Data Delivery](#Chapter6)
7. [Chapter 7: Building Data Pipelines](#Chapter7)
8. [Chapter 8: Cross-Cluster Data Mirroring](#Chapter8)

## Chapter 1: Meet Kafka<a name="Chapter1"></a>
### Publish/Subscribe Messaging
Publish/subscribe messaging is a pattern that is characterized by the sender (publisher) of a piece of data (message) not specifically directing it to a receiver. Instead, the publisher classifies the message somehow, and that receiver (subscriber) subscribes to receive certain classes of messages.

### Enter Kafka
Kafka is a publish/subscribe messaging system often described as a _distributed commit log_ or _distributing streaming platform_. A filesystem or database commit log is designed to provide a durable record of all transactions so that they can be replayed to consistently build the state of a system, data within Kafka is stored durably, in order, and can be read deterministically.
The unit of data within kafka is the _message_, and it is treated as an array of bytes, a message can have an optional bit of metadata, which is referred to as a key (also an array of bytes). Messages are written into Kafka in batches (typically compressed), a batch is just a collection of messages, all of which are being produced to the same topic and partition.

### Schemas
While messages are opaque byte arrays to Kafka itself, it is recommended that additional structure, or schema, be imposed on the message content so that it can be easily understood. Many Kafka developers favor the use of Apache Avro, which provides a compact serialization format; schemas that are separate from the message pay‐ loads and that do not require code to be generated when they change; and strong data typing and schema evolution, with both backward and forward compatibility.

### Topics and Partitions
Messages in Kafka are categorized into topics (analogous to a table in a DB), and broken down into partitions which are append only logs read in order from the beginning to the end, order is only guaranteed in the partitions, but not in the topics.

### Producers and Consumers
Producers write messages to topics, but can also write messages to partitions by using the message key and a partitioner. Consumers read messages, they subscribe to one or more topics and read messages in order, keeping track of the messages that has already been consumed (by means of offset which is a bit of metadata). A consumer works in a consumer group, one or more consumers that works together to consume a topic. The group assures that each partition is only consumed by one member.

### Brokers and Clusters
A single Kafka server is called a broker, it receives messages from producers, assigns offsets to them, and commits the messages to storage on disk. It also services consumers, responding to fetch requests for partitions and responding with the mes‐ sages that have been committed to disk.
Kafka servers works in clusters, which has a _controller_ (elected from one of the nodes) responsible for administrative operations. A partition is handled by a single broker in the cluster (leader of the partition), although the partition can be replicated in multiple brokers. Messages are stored in the brokers by default for 7 days (retention), or until the topic reaches certain size (default to 1GB).

## Chapter 2: Installing Kafka<a name="Chapter2"></a>
### Installing zookeeper
Kafka requires java 8 and zookeeper, you can check the zookeeper status by connecting to the server/port and sending the command `srvr`. A Zookeeper cluster is called an ensemble, which is recommended to have an odd number of nodes (3,5 or 7 as maximum). The nodes in zookeeper must have a common configuration, example:

```text
    tickTime=2000
    dataDir=/var/lib/zookeeper
    clientPort=2181
    # Amount of time to allow followers to connect with a leader, measured in tickTime units (40s in this case)
    initLimit=20 
    # How out-of-sync followers can be with the leader, measured in tickTime units (10s in this case)
    syncLimit=5
    # hostname:peerPort(port over which servers in the ensemble communicate with each other):leaderPort(port over which leader election is performed)
    server.1=zoo1.example.com:2888:3888
    server.2=zoo2.example.com:2888:3888
    server.3=zoo3.example.com:2888:3888
```

### Installing kafka broker
there is several scripts to start a broker, produce and consume messages in <kafka_dir>/bin.

Start the server with: `<kafka_dir>/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties`
Create and verify a topic: `<kafka_dir>/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test`
Produce messages to a test topic: `<kafka_dir>/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test`
Consume messages from a test topic: `<kafka_dir>/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning`

### Broker Configuration
#### General Broker

    * broker.id: Integer corresponding to the broker identifier. Must be unique within a single kafka cluster
    * port: The kafka port, can be assigned randomly, but if the port is lower than 1024 kafka must be started as root
    * zookeeper.connect: Zookeeper address on the form hostname:port/path where path is an optional Zookeeper path to use as a chroot environment for the Kafka cluster
    * log.dirs: Kafka persists all messages to disk, stored in the path specified by this comma separated list of paths 
    * num.recovery.threads.per.data.dir: Configurable pool of threads to handle log segments. Only one thread per log directory is used by default
    * auto.create.topics.enable: By default Kafka automatically create a topic under the following circumstances (set to false to deactivate it)
        - When a producer starts writing messages to the topic
        - When a consumer starts reading messages from the topic
        - When any client requests metadata for the topic

#### Topic Defaults

    * num.partitions: Number of partitions a new topic is created with (defaults 1). The number of partitions for a topic can only be increased, never decreased. Some considerations you should have to choose the number of partitions:
        - Size of the writes per second
        - Maximum throughput per consumer from a single partition
        - Maximum throughput per producer from a single partition (safe to skip as it is usually limited by consumers and not producers)
        - If you are sending messages to partitions based on keys, calculate throughput based on your expected future usage, not the current usage
        - Consider the number of partitions on each broker and available diskspace and network bandwidth per broker
        - Avoid overestimating, each partition uses memory and resources
     Experience suggests that limiting the size of the partition on the disk to less than 6 GB per day of retention  often gives satisfactory results
    * log.retention.ms/log.retention.minutes/log.retention.hours: Amount of time after which messages may be deleted (the smaller unit size will take precedence if more than one is specified)
    * log.retention.bytes: Applied per partition, total number of bytes of messages retained
    * log.segment.bytes: As messages are produced to the Kafka broker, they are appended to the current log segment for the partition. Once the log segment has reached the size specified by the log.segment.bytes parameter, which defaults to 1 GB, the log segment is closed and a new one is opened. Once a log segment has been closed, it can be considered for expiration
    * log.segment.ms: Specifies the amount of time after which a log segment should be closed
    * message.max.bytes: Maximum size of a message that can be produced (defaults to 1MB) A producer that tries to send a message larger than this will receive an error, this value must be coordinated with the fetch.message.max.bytes configuration on consumer clients
    
### Hardware Selection
Several factors that will contribute to the overall performance: disk throughput and capacity, memory, networking, and CPU.
 
    * Disk Throughput: Faster disk writes will equal lower produce latency. SSD will usually have better performance
    * Disk Capacity: Depends on the retention configuration. Consider a 10% of overhead or other files appart of the capacity to store log messages
    * Memory: Consumers reads from the end of the partitions, where the consumer is caught up and lagging behind the producers very little, if at all. The messages the consumer is reading are optimally stored in the system’s page cache, having more memory available to the system for page cache will improve the performance of consumer clients
    * Networking: The available network throughput will specify the maximum amount of traffic that Kafka can handle. Cluster replication and mirroring will also increase requirements
    * CPU: Ideally, clients should compress messages to optimize network and disk usage. The Kafka broker must decompress all message batches, however, in order to validate the checksum of the individual messages and assign offsets
    
### Kafka Clusters
Considerations on configuring a kafka cluster:

    * Number of brokers: Depends on the overall disk capacity, capacity of network interfaces...
    * Broker configuration: All brokers must have the same config‐ uration for the _zookeeper.connect_ parameter and all brokers must have a unique value for the _broker.id_
    * OS Tuning: Few configuration changes can be made to improve kafka performance
        - Virtual memory: Try to avoid memory swaps. Disable it, or set vm.swappiness parameter to a very low value, such as 1. In regards to I/O the number of dirty pages that are allowed, before the flush background process starts writing them to disk, can be reduced by setting the =vm.dirty_background_ratio value lower than the default of 10. The total number of dirty pages that are allowed before the kernel forces synchronous operations to flush them to disk can also be increased by changing the value of vm.dirty_ratio (defaults 20) 
        - Disk: The Extents File System (XFS) is used in most of linux distributions due to performance reasons.
        - Networking: The recommended changes for Kafka are the same as those suggested for most web servers and 
        other networking applications. The first adjustment is to change the default and maximum amount of memory allocated for the send and receive buffers for each socket (_net.core.wmem_default_, _net.core.rmem_default_, _net.core.wmem_max_ and net.core.rmem_max). The send and receive buffer sizes for TCP sockets must be set separately using the _net.ipv4.tcp_wmem_ and _net.ipv4.tcp_rmem_ parameters, the maximum size cannot be larger than the values specified for all sockets.

### Production Concerns
#### Garbage Collector Options
Garbage First (or G1) garbage collector is designed to automatically adjust to different workloads and provide consis‐ tent pause times for garbage collection over the lifetime of the application. Some options:

    * MaxGCPauseMillis: Preferred pause time for each garbage-collection cycle (not a hard limit, can be exceeded)
    * InitiatingHeapOccupancyPercent: Percentage of the total heap that may be in use before G1 will start a collection cycle
    
The start script for Kafka does not use the G1 collector, instead defaulting to using parallel new and concurrent mark and sweep garbage collection. To use G1 do:
```bash
export JAVA_HOME=/usr/java/jdk1.8.0_51
export KAFKA_JVM_PERFORMANCE_OPTS="-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+DisableExplicitGC -Djava.awt.headless=true"
/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
```

### Colocating Applications on Zookeeper
Kafka utilizes Zookeeper for storing metadata information about the brokers, topics, and partitions. Writes to Zookeeper are only performed on changes to the member‐ ship of consumer groups or on changes to the Kafka cluster itself (does not justify a dedicated zookeeper ensemble). Consumers have a configurable choice to use either Zookeeper or Kafka for committing offsets. These commits can be a significant amount of Zookeeper traffic, especially in a cluster with many consumers, and will need to be taken into account.

## Chapter 3: Kafka Producers: Writing Messages to Kafka<a name="Chapter3"></a>
### Producer Overview
We start producing messages to Kafka by creating a _ProducerRecord_, which must include the topic we want to send the record to and a value (key and/or partition information is optional). The message would then be serialized as a byte array and sent to a partitioner (if the partition is specified in the message the partitioner does nothing). Once the producer knows which topic and partition the record will go to, it then adds the record to a batch of records that will also be sent to the same topic and partition. A separate thread is responsible for sending those batches of records to the appropriate Kafka brokers. The broker sends back a _RecordMetadata_ it the operation was successful containing the topic, partition, and the offset of the record within the partition or an error if the operation failed (in this case the Producer might attempt to retry the message delivery).

### Constructing a Kafka Producer
A Kafka producer has three mandatory properties:

    * bootstrap.servers: List of host:port pairs of brokers that the producer will use to establish initial connection to the Kafka cluster
    * key.serializer: Name of a class that will be used to serialize the keys of the records we will produce to Kafka. Kafka brokers expect byte arrays as keys and values of messages. 'key.serializer' should be set to a name of a class that implements the 'org.apache.kafka.common.serialization.Serializer' interface. Setting 'key.serializer' is required even if you intend to send only values.
    * value.serializer: Name of a class that will be used to serialize the values of the records we will produce to Kafka
    
There is three primary methods of sending messages:

    * Fire-and-forget: We send a message to the server and don’t really care if it arrives succesfully or not.
    * Synchronous send: We send a message, the send() method returns a Future object, and we use get() to wait on the future and see if the send() was successful or not
    * Asynchronous send: We call the send() method with a callback function, which gets triggered when it receives a response from the Kafka broker
     
### Sending a Message to Kafka
Example of Producer creationg and message sending:

```java
//Example of creating a producer and sending a message
private Properties kafkaProps = new Properties();
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProps.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
kafkaProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
KafkaProducer producer = new KafkaProducer<String, String>(kafkaProps);
ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "Precision Products", "France"); //Topic-key-value constructor

try {
      producer.send(record);
} catch (Exception e) {
            e.printStackTrace();
}
```

#### Sending a Message Synchronously
To send a message synchronoulsy, we just have to replace the previous `producer.send(record);` by ` producer.send(record).get();`.
KafkaProducer has two types of errors:

    * Retriable errors: Are those that can be resolved by sending the message again (i.e. connection failure), the KafkaProducer can be configured to retry those errors automatically.
    * Non retriable errors: For example 'message too large'.
    
#### Sending a Message Asynchronously
A callback needs to be implemented to send the message asynchronously:

```java
 private class DemoProducerCallback implements org.apache.kafka.clients.producer.Callback {
    @Override public void onCompletion(RecordMetadata recordMetadata, Exception e) {
     if (e != null) {
         e.printStackTrace();
        }
    }
}
producer.send(record, new DemoProducerCallback());
```

#### Configuring Producers
Some parameters that can have a significant impact on memory use, performance, and reliability of the producers:

    * acks: Controls how many partition replicas must receive the record before the producer can consider the write successful
        - ack=0: The producer assumes the sending was successful straight away (for hight throughput)
        - ack=1: The producer waits for the partition leader to reply, messages can be lost if the leader crashes unexpectedly. Throughput depends on the way to send the messages (synchronous or asynchronous)
        - ack=all: the producer will receive a success response from the broker once all in-sync replicas received the message
    * buffer.memory: Sets the amount of memory the producer will use to buffer messages waiting to be sent to brokers
    * compression.type: By default messages are uncompressed, this parameter can be set to snappy, gzip, or lz4
    * retries:  How many times the producer will retry sending the message before giving up and notifying the client of an issue
    * retry.backoff.ms: milliseconds to wait after a failed attempt of message send (defaults 100ms)
    * batch.size: Messages to the same partition are batched together, this parameter controls the amount of bytes used for each batch
    * linger.ms: Controls the amount of time to wait for additional messages before send‐ ing the current batch (a batch is send either when the buffer is full, or when the linger period is reached). By default, the producer will send messages as soon as there is a sender thread available to send them, even if there’s just one message in the batch.
    * client.id: Producer identifier, can be any string
    * max.in.flight.requests.per.connection: Controls how many messages the producer will send to the server without receiving responses
    * timeout.ms, request.timeout.ms, and metadata.fetch.timeout.ms: control how long the producer will wait for a reply from the server when sending data (request.timeout.ms) and when requesting metadata (metadata.fetch.timeout.ms) or the time the broker will wait for in-sync replicas to acknowledge the message in order to meet the acks configuration (timeout.ms).
    * max.block.ms: How long the producer will block when calling send() and when explicitly requesting metadata via partitionsFor()
    * max.request.size: Size of a produce request sent by the producer. It caps both the size of the largest message that can be sent and the number of messages that the producer can send in one request.
    * receive.buffer.bytes and send.bu er.bytes: Sizes of the TCP send and receive buffers used by the sockets when writing and reading data (-1 for using the OS defaults)
    
### Serializers
#### Custom Serializers 
Example of custom serializer for a java bean _Customer(customerId, customerName)_:

```java
import org.apache.kafka.common.errors.SerializationException;
import java.nio.ByteBuffer;
import java.util.Map;

public class CustomerSerializer implements Serializer<Customer> {
        @Override
  public void configure(Map configs, boolean isKey) {
   // nothing to configure
  }
  /**
  * We are serializing Customer as: 4 byte int representing customerId 4 byte int representing length of 
  * customerName in UTF-8 bytes (0 if name is Null) N bytes representing customerName in UTF-8
  */
  @Override
  public byte[] serialize(String topic, Customer data) {
    try {
        byte[] serializedName;
        int stringSize;
        if (data == null) return null;
        else if (data.getName() != null) {
                serializeName = data.getName().getBytes("UTF-8");
                stringSize = serializedName.length;
        } else {
            serializedName = new byte[0];
            stringSize = 0;
        }
        ByteBuffer buffer = ByteBuffer.allocate(4 + 4 + stringSize);
        buffer.putInt(data.getID());
        buffer.putInt(stringSize);
        buffer.put(serializedName);
        return buffer.array();
    } catch (Exception e) {
        throw new SerializationException("Error when serializing Customer to byte[] " + e);
    } 
  }
  
  @Override
  public void close() {
    // nothing to close
  }
}
```

It is clear that the code is fragile and difficult to maintain (imagine supporting backwards compatibility if we add a new field)

#### Serializing Using Apache Avro
Apache Avro is a language-neutral data serialization format, Avro data is described in a language-independent schema (usually JSON), Avro assumes that the schema is present when reading and writing files, usually by embedding the schema in the files themselves. Two rules have to be considered:

    * The schema used for writing the data and the schema expected by the reading application must be compatible. The Avro documentation includes compatibility rules
    * The deserializer will need access to the schema that was used when writing the data, even when it is different than the schema expected by the application that accesses the data. In Avro files, the writing schema is included in the file itself, but there is a better way to handle this for Kafka messages
    
Avro requires the entire schema to be present when reading the record, in kafka this is achieved with a common architecture pattern and use a Schema Registry: The schema is stored somewhere else and only the identifier is stored with each record. The key is that all this work — storing the schema in the registry and pulling it up when required — is done in the serializers and deserializers. To use the avro serializer, set the value of the 'key.serializer' or 'value.serializer' to 'io.confluent.kafka.serializers.KafkaAvroSerializer' and set the value of 'schema.registry.url' to the appropriate registry system.
You can also pass the Avro schema explicitly by creating the schema string and parsing it with `Schema schema = new Schema.Parser().parse(schemaString);`, then setting the message type in the producer to be a _GenericRecord_, and adding each individual fields on the genering record like this:

```java
GenericRecord customer = new GenericData.Record(schema);
customer.put("customerId", "1");
customer.put("customerName", "John Doe");
```

### Partitions
Keys in messages serves two goals: they are additional information that gets stored with the message, and they are also used to decide which one of the topic partitions the message will be written to (message with the same key goes to the same partition). When the key is null and the default partitioner is used, the record will be sent to one of the available partitions of the topic at random (round robin). If the key exists, and the default partitioner is used, then the key is hashed to get the partition. The mapping of keys to partitions is consistent only as long as the number of partitions in a topic does not change.
As with the Serializer, a custom Partitioner can be implemented by extending the _org.apache.kafka.clients.producer.Partitioner_ interface.

## Chapter 4: Kafka Consumers: Reading Data from Kafka<a name="Chapter4"></a>
### Kafka Consumer Concepts
#### Consumers and Consumer Groups
Kafka consumers are typically part of a consumer group. When multiple consumers are subscribed to a topic and belong to the same consumer group, each consumer in the group will receive messages from a different subset of the partitions in the topic. If we add more consumers to a single group with a single topic than we have partitions, some of the consumers will be idle and get no messages at all. Kafka topic is scaled by adding more consumers to a consumer group (but the number of partitions determines the consumption rate). To make sure an application gets all the messages in a topic (for example multiple applications consuming the same topic), ensure the application has its own consumer group, what a consumer group does with the messages doesn't affect the other consumer groups.

#### Consumer Groups and Partition Rebalance
Reassignment of partitions to consumers happens when a consumer is added to the group, a consumer leaves the group or the topics the consumer group is consuming are modified (i.e. new partitions are added). Moving partition ownership from one consumer to another is called a rebalance. During a rebalance, consumers can't consume messages, so a rebalance is basically a short window of unavail‐ ability of the entire consumer group. Consumers maintain membership in a consumer group and ownership of the partitions assigned to them is by sending heartbeats to a Kafka broker designated as the group coordinator.

### Creating a Kafka Consumer
Similar to creating a producer, you pass a list of properties to the constructor of consumer with the three mandatory configurations: _bootstrap.servers_, _key.deserializer_, and _value.deserializer_ (and optionally _group.id_):

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
```

To subscribe the previous consumer to a topic, we do `consumer.subscribe(Collections.singletonList("topicName"))`, although is possible to subscribe to topics based on a regular expression like in `consumer.subscribe("test.*")`, this is commonly used in applications that replicate data between Kafka and another system.

### The Poll Loop
A pool loop is used to consume messages from the server, once the consumer subscribe to topics, the poll loop handles all details of coordination, partition rebalances, heartbeats, and data fetching. We must consider the rule of one consumer per thread.

```java
try {
    while (true) {
//consumers must keep polling Kafka or they will be considered dead, the parameter is a timeout interval which specifies how long poll will block
//if data is not available in the consumer buffer
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> r : records){
            log.debug("topic = %s, partition = %s, offset = %d,customer = %s, country = %s\n", r.topic(), r.partition(), r.offset(),r.key(), r.value());
        }
    }
} finally {
    consumer.close(); //Closes the network connections and sockets and triggers a rebalance immediately
}
```

### Configuring Consumers
Other important consumer configuration parameters:

    * fetch.min.bytes: Specifies the minimum amount of data that it wants to receive from the broker when fetching records. If a broker receives a request for records from a consumer but the new records amount to fewer bytes than min.fetch.bytes
    * fetch.max.wait.ms: Configures Kafka to wait until it has enough data to send before responding to the consumer (defaults 500ms)
    * max.partition.fetch.bytes: Controls the maximum number of bytes the server will return per partition (defaults to 1MB), it must be larger than the largest message a broker will accept
    * session.timeout.ms: The amount of time a consumer can be out of contact with the brokers while still considered alive (defaults 3s)
    * auto.offset.reset: Controls the behavior of the consumer when it starts reading a partition for which it doesn’t have a committed offset or if the committed offset it has is invalid. The default is 'latest' but can be set to 'earliest' too
    * enable.auto.commit: Defaults to true, you might also want to control how frequently offsets will be committed using auto.commit.interval.ms
    * partition.assignment.strategy: There is two strategies (defaults to org.apache.kafka.clients.consumer.RangeAssignor):
        - Range: Assigns to each consumer a consecutive subset of partitions from each topic it subscribes to
        - Round RobinTakes all the partitions from all subscribed topics and assigns them to consumers sequentially, one by one
    * client.id: Can be any string. Used in logging and metrics and for quotas
    * max.poll.records: Controls the maximum number of records that a single call to poll() will return
    * receive.buffer.bytes and send.buffer.bytes: Sizes of the TCP send and receive buffers used by the sockets when writing and reading data
    
### Commits and Offsets
Kafka allows consumers to use Kafka to track their position (offset) in each partition. The action of updating the current position in the partition is called a commit. A commit happens when a consumer produces a message to kafka to '__consumer_offsets' topic with the committed offset for each partition. if a consumer crashes or a new consumer joins the consumer group, this will trigger a rebalance. After a rebalance, each consumer may be assigned a new set of partitions than the one it processed before. In order to know where to pick up the work, the consumer will read the latest committed offset of each partition and continue from there.

#### Automatic Commit
If 'enable.auto.commit=true', then every five seconds (controlled by 'auto.commit.interval.ms') the consumer will commit the largest offset your client received. With autocommit enabled, a call to poll will always commit the last offset returned by the previous poll, which might lead to duplicate messages.

#### Commit Current Offset
The simplest and most reliable of the commit APIs is `consumer.commitSync()` which commits the lates offset returnet by poll and returns once the offset is committed (might throw a _CommitFailedException_).

#### Asynchronous Commit
The asynchronous commit `consumer.commitAsync()` does not block the consumer until the broker replies to a commit, it just keeps going on, but it will not retry the commit if it fails. This methods also gives you an option to pass in a callback that will be triggered when the broker responds (by passing a _OffsetCommitCallback_ object).

#### Combining Synchronous and Asynchronous Commits
A common pattern is to combine commitAsync() with commitSync() just before shutdown to make sure the last offset is committed before a shutdown or a rebalance.

#### Commit Specified Offset
In case you want to commit the offset in the middle of a big batch processing, you can do it explicitely by calling commitSync() and commitAsync() and pass a map of partitions and offsets that you wish to commit:

```java
private Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();
...
currentOffsets.put(new TopicPartition(record.topic(),record.partition()), new OffsetAndMetadata(record.offset()+1, "no metadata"));
...
consumer.commitAsync(currentOffsets, null); // or commitSync
```

### Rebalance Listeners
If your consumer maintained a buffer with events that it only processes occasionally, you will want to process the events you accumulated before losing ownership of the partition (or close file handles, database connections...). You can pass a _ConsumerRebalanceListener_ when 
calling the _subscribe()_ method, which has two methods:

    * 'public void onPartitionsRevoked(Collection<TopicPartition> partitions)': Called before rebalancing starts and after the consumer stopped consuming messages
    * 'public void onPartitionsAssigned(Collection<TopicPartition> partitions)': Called after partitions have been reassigned to the broker, but before the con‐ sumer starts consuming messages
    
### Consuming Records with Specific Offsets
So start reading all messages from the beginning of the partition, use `consumer.seekToBeginning(TopicPartition tp)` to skip all the way to the end of the partition use `consumer.seekToEnd(TopicPartition tp)`, but to seek a specific offset use `consumer.seek(TopicPartition tp, Integer Offset)`. This is likely done in the previously described `onPartitionsAssigned` method of a _ConsumerRebalanceListener_ instance (for example getting the offset from a database instead of Kafka).

### But How Do We Exit?
When you decide to exit the poll loop, you will need another thread to call _consumer.wakeup()_ If you are running the consumer loop in the main thread, this can be done from ShutdownHook, calling wakeup will cause poll() to exit with _WakeupException_. The _WakeupException_ doesn't need to be handled, but before exiting the thread, you must call `consumer.close()`.
```java
// This code is meant to be executed in the main application thread
Runtime.getRuntime().addShutdownHook(new Thread() {
    public void run() {
        System.out.println("Starting exit...");
        consumer.wakeup();
        try {
            mainThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    } 
});
```

### Deserializers
Kafka consumers require deserializers to convert byte arrays recieved from Kafka into Java objects, the AvroSerializer can make sure that all the data written to a specific topic is compatible with the schema of the topic.

#### Custom deserializers
Example of custom deserializer for a java bean _Customer(customerId, customerName)_:

```java
import org.apache.kafka.common.errors.SerializationException;
import java.nio.ByteBuffer;
import java.util.Map;

public class CustomerDeserializer implements Deserializer<Customer> {
    @Override public void configure(Map configs, boolean isKey) {// nothing to configure}
      
    @Override public Customer deserialize(String topic, byte[] data) {
        try {
            if (data == null) return null;
            if (data.length < 8) throw new SerializationException("Size of data received by IntegerDeserializer is shorter than expected");
            
            ByteBuffer buffer = ByteBuffer.wrap(data);
            int id = buffer.getInt();
            int nameSize = buffer.getInt();
            byte[] nameBytes = new Array[Byte](nameSize);
            buffer.get(nameBytes);
            return new Customer(id, new String(nameBytes, 'UTF-8'));
        } catch (Exception e) {
            throw new SerializationException("Error when serializing Customer to byte[] " + e);
        }
    }
@Override public void close() {// nothing to close}
}
```

This deserializer needs to be configured in the "key.deserializer" or "value.deserializer" property so the consumer 
can be defined as `KafkaConsumer<String, Customer> consumer = new KafkaConsumer<>(props);`

#### Using Avro deserialization with Kafka consumer
To use the avro deserializer, we need to define a "schema.registry.url" and define the consumer like in the custom deserializer.

### Standalone Consumer: Why and How to Use a Consumer Without a Group
In case you want to have a single consumer that always needs to read data from all the partitions in a topic or from a specific partition in a topic, the consumer assign itself to a the partitions it wants to read from:

```java
List<PartitionInfo> partitionInfos = consumer.partitionsFor("topic");
for (PartitionInfo partition : partitionInfos) partitions.add(new TopicPartition(partition.topic(), partition.partition()));

//Assign the partitions directly, but if someone adds new partitions to the topic, the consumer will not be notified
consumer.assign(partitions);
```

### Older Consumer APIs
Apache Kafka still has two older clients written in Scala that are part of the kafka.consumer package, which is part of the core Kafka module: 
    
    * SimpleConsumer: Thin wrapper around the Kafka APIs that allows you to consume from specific partitions and offsets
    * ZookeeperConsumerConnector: Similar to the current consumer, it has consumer groups and it rebalances partitions but uses Zookeeper to manage consumer groups
    
## Chapter 5: Kafka Internals<a name="Chapter5"></a> 
### Cluster Membership
Kafka uses Apache Zookeeper to maintain the list of brokers that are currently members of a cluster. Every time a broker process starts, it registers itself with its ID in Zookeeper by creating an ephemeral node, which would be removed if a broker loses connectivity to zookeeper (other kafka components would be notified in this case). The broker ID will still exists in other data structures so if another broker join the cluster with the same ID, it gets the same partitions and topics assigned to it.

### The Controller
The controller is one of the Kafka brokers that is responsible for electing partition leaders, the first broker that starts in the cluster becomes the controller by creating an ephemeral node in ZooKeeper called '/controller'. The brokers create a Zookeeper watch on the controller node so they get notified of changes to this node. When the controller loses connectivity, the ephemeral node is removed and the brokers notified. The brokers will then try to create the Controller (only one will succeed, the rest will get a "node already exists" exception). Each time a controller is elected, it receives a new, higher controller epoch number through a Zookeeper conditional increment operation. Messages from a controller with an older number, will be ignored.

### Replication
In kafka, replicas are stored on brokers, and each broker typically stores hundreds or even thousands of replicas belonging to different topics and partitions. Types:

    * Leader replica: Each partition has a single replica designated as the leader. All produce/consume requests go through the leader, to guarantee consistency
    * Follower replica: Followers don’t serve client requests, they replicate messages from the leader and stay up-to-date with the newer messages the leader has

The leader has to know which of the follower replicas is up-to-date with the leader. To stay in sync with the leader, the replicas send the leader Fetch requests, the exact same type of requests that consumers send in order to consume messages (the messages are sent as response to this requests). If a replica hasn’t requested a message in more than 10 seconds (configurable through 'replica.lag.time.max.ms) or if it has requested messages but hasn’t caught up to the most recent mes‐ sage in more than 10 seconds, the replica is considered out of sync and can't be elected leader in case of failure. A preferred leader is the replica that was the leader when the topic was originally created, if 'auto.leader.rebalance.enable=true' if the preferred leader replica is not the current leader but is in-sync, a leader election will be triggered to make the preferred leader the current leader.

### Request Processing
Kafka has a binary protocol that specifies the format of the requests and how brokers respond to them, all requests sent to the broker from a specific client will be processed in the order in which they were received. All requests have a standard header that includes:

    * Request type (also called API key)
    * Request version (so the brokers can handle clients of different versions and respond accordingly)
    * Correlation ID: a number that uniquely identifies the request and also appears in the response and in the error logs (the ID is used for troubleshooting)
    * Client ID: used to identify the application that sent the request
    
For each port the broker listens on, the broker runs an acceptor thread that creates a connection and hands it over to a processor thread (also called network threads)for handling. The network threads are responsible for taking requests from client connections, placing them in a request queue, and picking up responses from a response queue and sending them back to clients.Once requests are placed on the request queue, IO threads are responsible for picking them up and processing them. The most common types of requests are:
    
    * Produce requests: Sent by producers and contain messages the clients write to Kafka brokers
    * Fetch requests: Sent by consumers and follower replicas when they read messages from Kafka brokers

Both produce requests and fetch requests have to be sent to the leader replica of a partition, which clients discovered through _metadata requests_, which includes a list of topics the client is interested in. Clients typically cache this information by sending another metadata request so they know if the topic metadata changed.

#### Produce Requests
When the broker that contains the lead replica for a partition receives a produce request for this partition, it will run a few validations:
    
    * Does the user sending the data have write privileges on the topic?
    * Is the number of acks specified in the request valid (only 0, 1, and “all” are allowed)?
    * If acks is set to all, are there enough in-sync replicas for safely writing the message? (Brokers can be configured to refuse new messages if the number of in-sync replicas falls below a configurable number)

If all the above passes it will write the message to local disk. Once the message is written to the leader of the partition, the broker examines the acks configuration (if 0 or 1, the broker will respond immediately, if 'all', the request will be stored in a buffer called purgatory until the leader observes that the follower replicas replicated the message, at which point a response is sent to the client)

#### Fetch Requests
The client sends a request, asking the broker to send messages from a list of topics, partitions, and offsets. Clients specify a limit to how much data the broker can return for each partition and allocate memory to handle the response. The broker would read the message for the partition after validating the request up to the limit specified by the client (clients can also set a lower boundary for the data size to receive). Not all the data that exists on the leader of the parti‐ tion is available for clients to read as messages not replicated to enough replicas yet are considered “unsafe”

#### Other Requests
The same binary protocol explaned before is used to communicate between the Kafka brokers themselves (i.e. when the controller announces that a partition has a new leader, it sends a _LeaderAndIsr_ request to the new leader and followers). This includes calls for different clients (the _ApiVersionRequest_ allows clients to ask the broker which versions of each request is supported), different versions of the current requests, etc... 

### Physical Storage
The basic storage unit of Kafka is a partition replica, which cannot be split so the size of a partition is limited by the space available on a single mount point.  

#### Partition Allocation
When kafka allocates partitions, it does so by taking in consideration:

    * To spread replicas evenly among brokers
    * To make sure that for each partition, each replica is on a different broker
    * If the brokers have rack information (from version 0.10.0 and higher), then assign the replicas for each partition to different racks if possible
    
To decide in which directory new partitions are stored, we count the number of partitions on each directory and add the new partition to the directory with the fewest partitions. This means that if you add a new disk, all the new partitions will be created on that disk.

#### File Management
Partitions are splitted in segments so it is easier to apply retention policies. The current segment is called _active segment_ and is never deleted. A Kafka broker will keep an open file handle to every segment in every partition, even inactive segments.

#### File Format
Each segment is stored in a single data file. Inside the file, we store Kafka messages and their offsets. The format of the data on the disk is identical to the format of the messages that we send from the producer to the broker and later from the broker to the consumers. Each message contains—in addition to its key, value, and offset—things like the message size, checksum code that allows us to detect corruption, magic byte that indicates the version of the message format, compression codec (Snappy, GZip, or LZ4), and a timestamp (from v 0.10.0).
Kafka brokers ship with the DumpLogSegment tool, which allows you to check the offset, checksum, magic byte, size, and compression codec for each message. You can run the tool using: `bin/kafka-run-class.sh kafka.tools.DumpLogSegments`.

#### Indexes
In order to help brokers quickly locate the message for a given offset, Kafka maintains an index for each partition (broken in segments as well). 

#### Compaction
Kafka supports retention policy in a topic to be of type _delete_ which deletes events older than the retention time and _compact_ which stores the most recent value for each key in the topic (only for non null keys). Each log is viewed as split into two portions:

    * Clean: This section contains only one value for each key, which is the latest value at the time of the previous compaction
    * Dirty: Messages that were written after the last compaction.

If compaction is enabled when Kafka starts (by _log.cleaner.enabled_ configuration), each broker will start a compaction manager thread and _n_ compaction threads.

#### Deleted Events
In order to delete a key from the system completely, not even saving the last message, the application must produce a message that contains that key and a null value (called tombstone) which would be around for a configurable amount of time. During this time, consum‐ ers will be able to see this message and know that the value is deleted.

#### When Are Topics Compacted?
The compact policy never compacts the current segment. In version 0.10.0 and older, Kafka will start compacting when 50% of the topic contains dirty records. In future versions, the plan is to add a grace period during which we guarantee that messages will remain uncompacted.

## Chapter 6: Reliable Data Delivery<a name="Chapter6"></a>
### Reliability Guarantees
The same way RDBMS comes with ACID (atomicity, consistency, isolation, and durability) reliability guarantees, kafka has its own: 

    * Kafka provides order guarantee of messages in a partition
    * Produced messages are considered “committed” when they were written to the partition on all its in-sync replicas (but not necessarily flushed to disk) 
    * Messages that are committed will not be lost as long as at least one replica remains alive
    * Consumers can only read messages that are committed
    
### Replication
An in-sync replica that is slightly behind can slow down producers and consumers— since they wait for all the in-sync replicas to get the message before it is committed. Once a replica falls out of sync, we no longer wait for it to get messages. However, fewer in-sync replicas, the effective replication factor of the partition is lower and therefore there is a higher risk for downtime or data loss.

### Broker configuration
Many broker configuration variables can apply at the broker level, and at the topic level.

#### Replication Factor
The topic-level configuration is _replication.factor_. At the broker level, you control the _default.replication.factor_ for automatically created topics. The default is 3, which is good enough for most applications. Try to use the _broker.rack_ broker configuration parameter to configure the rack name for each broker.

#### Unclean Leader Election
Set at broker level through _unclean.leader.election.enable_ (defaults true). When no in-sync replicas exists except for the leader that just became unavailable there is two ways to handle this scenario:

    * If we don’t allow the out-of-sync replica to become the new leader, the partition will remain offline until we bring the old leader back online
    * If we do allow the out-of-sync replica to become the new leader, we are going to lose all messages that were written to the old leader while that replica was out of sync and also cause some inconsistencies in consumers. If the old leader is recovered after, it will delete any messages it got that are ahead of the current leader. Those messages will not be available to any consumer in the future

Unclean leader election is disabled in systems where data quality and consistency is critical and is enabled when availability is more important.

#### Minimum In-Sync Replicas
Both the topic and the broker-level configuration are called _min.insync.replicas_, the parameter controls how many replicas needs to be in-sync to write to a partition. If the number of in-sync replicas fall below this number, the brokers will no longer accept produce requests, throwing a _NotEnoughReplicasException_. Consumers can still read existing data.

### Using Producers in a Reliable System
There are two important things that everyone who writes applications that produce to Kafka must pay attention to:
    
    * Use the correct acks configuration to match reliability requirements
    * Handle errors correctly both in configuration and in code
    
#### Configuring Producer Retries
The producer can handle retriable errors that are returned by the broker for you. Kafka’s cross-DC replication tool is configured by default to retry endlessly (i.e. retries = MAX_INT). Retrying to send a failed message often includes a small risk of creating duplicate messages. Retries and careful error handling can guarantee that each message will be stored at least once, but we can't guarantee it will be stored exactly once. Applications are often designed to create idempotent messages or include message IDs to deal with this risk.

#### Additional Error Handling
As a developer, you must still be able to handle other types of errors. These include:
    
    *Nonretriable broker errors such as errors regarding message size, authorization errors, etc.
    *Errors that occur before the message was sent to the broker—for example, serialization errors
    *Errors that occur when the producer exhausted all retry attempts or when the available producer's memory is filled to store messages while retrying
    
### Using Consumers in a Reliable System
Consumers must make sure they keep track of which messages they've or haven't read. For each partition it is consuming, the consumer stores its current location, so they or another consumer will know where to continue after a restart. The main way consumers can lose messages is when committing offsets for events they've read but didn't completely process yet. 

#### Important Consumer Configuration Properties for Reliable Processing
There are 4 important consumer configurations to take into account:
 
    *  group.id: Two consumers with the same group id subscribed to the same topic would get assignet a subset of the partitions in the topic, to process all messages in a topic, a consumer must have a unique id
    * auto.offset.reset: Controls what the consumer will do when no offsets were committed or when the consumer asks for offsets that don’t exist in the broker. If you choose earliest, the consumer will start from the beginning of the partition whenever it doesn’t have a valid offset. If you choose latest, the consumer will start at the end of the partition
    * enable.auto.commit: The automatic offset commit guarantees you will never commit an offset that you didn't process (might lead to duplicates)
    * auto.com mit.interval.ms: This configuration lets you configure how frequently they will be committed if commits are automatic (default 5s)
    
#### Explicitly Committing Offsets in Consumers
Some recommendations in case the consumer is designed to commit its offsets:

    * Always commit offsets after events were processed: You can use the auto-commit config‐ uration or commit events at the end of the poll loop
    * Commit frequency is a trade-off between performance and number of duplicates in the event of a crash (committing has some performance overhead)
    * Make sure you know exactly what o sets you are committing (it is critical to always commit offsets for messages after they were processed)
    * Rebalances: Remember you need to handle rebalances properly, usually involves committing offsets before partitions are revoked and cleaning any state you maintain when you are assigned new partitions
    * Consumers may need to retry: Unlike traditional pub/sub messaging systems, you commit offsets and not ack individual messages. One option, when you encounter a retriable error, is to commit the last record you processed successfully. Then store the records that still need to be processed in a buffer and keep trying to process the records, or else, write it to a separate topic and continue
    * Consumers may need to maintain state: If you need to maintain state, one way to do this is to write the latest accumulated value to a “results” topic at the same time you are committing the offset
    * Handling long processing times: For long running calls, even if you don’t want to process additional records, you must continue polling so the client can send heartbeats to the broker. A common pattern in these cases is to hand off the data to a thread-pool when possible with multiple threads to speed things up a bit by processing in parallel while the consumer keeps polling without fetching any data
    * Exactly-once delivery: A way to achieve this is to write results to a system that has some support for unique keys (making it idempotent). You can also write to a system that supports transactions, records and offsets would be written in the same transation so they'll be in sync
    
### Validating System Reliability
#### Validating Configuration
It is easy to test the broker and client configuration in isolation from the application logic. Kafka includes two important tools to help with this validation. The _org.apache.kafka.tools_ package includes _VerifiableProducer_ and _VerifiableConsumer_ classes (can be run from command tools or embedded in the code).
The idea is that the verifiable producer produces a sequence of messages containing numbers from 1 to a value you choose. When you run it, it will print success or error for each message sent to the broker, based on the acks received. You can run different tests like leader election, controller election, rolling restart or unclean leader election. The Apache Kafka source repository includes an extensive test suite.

#### Validating Applications
Check your application provides the guarantees you need. A recommendation is to test at least the following scenarios:

    * Clients lose connectivity to the server
    * Leader election
    * Rolling restart of brokers
    * Rolling restart of consumers
    * Rolling restart of producers
    
#### Monitoring Reliability in Production
Kafka’s Java clients include JMX metrics that allow monitoring client-side status and events. For the producers, the two metrics most important for reliability are error-rate and retry-rate per record. On the consumer side, the most important metric is consumer lag (how far the consumer is from the latest message committed). Keep an eye also on the time a message takes to be processed by checking the timestamp (v >= 1.10.0) or adding a timestamp to the message (v < 1.10.0)
 
## Chapter 7: Building Data Pipelines<a name="Chapter7"></a>
The main value Kafka provides to data pipelines is its ability to serve as a very large, reliable buffer between various stages in the pipeline, effectively decoupling producers and consumers of data within the pipeline.

### Considerations When Building Data Pipelines
#### Timeliness
Kafka, can be used to support anything from near-real-time pipelines to hourly batches. Kafka acts as a giant buffer that decouples the time-sensitivity requirements between producers and consumers.

#### Reliability
To avoid single points of failure and allow for fast and automatic recovery Kafka’s Connect APIs make it easier for connectors to build an end-to-end exactly-once pipeline by providing APIs for integrating with the external systems when handling offsets. 

#### High and Varying Throughput
With kafka there is no need to couple consumer and producer throughput, Kafka is a high-throughput distributed system capable of processing hundreds of megabytes per second on even modest clusters. The Kafka Connect API focuses on parallelizing the work and not just scaling it out. 

#### Data Formats
Kafka itself and the Connect APIs are completely agnostic when it comes to data formats. Many sources and sinks have a schema; we can read the schema from the source with the data, store it, and use it to validate compatibility or even update the schema in the sink database, a generic data integration frame‐ work should also handle differences in behavior between various sources and sinks.

#### Transformations
There are generally two schools of building data pipelines: ETL and ELT, depending on the flexibility and the burden we might want to give to the target system.

#### Security
In terms of data pipelines, the main security concerns are:

    * Can we make sure the data going through the pipe is encrypted? (for data pipelines that cross datacenter boundaries)
    * Who is allowed to make modifications to the pipelines?
    * If the data pipeline needs to read or write from access-controlled locations, can it authenticate properly?

Kafka allows encrypting data on the wire from sources to Kafka and from Kafka to sinks. It also supports authentication (via SASL) and authorization. Kafka also provides an audit log to track access—unauthorized and authorized.

#### Failure Handling
Because Kafka stores all events for long periods of time, it is possible to go back in time and recover from errors when needed.

#### Coupling and Agility
There are multiple ways accidental coupling can happen:

    * Ad-hoc pipelines: Building a custom pipeline for each pair of applications tightly couples the data pipeline to the specific end points and creates a mess of integration points
    * Loss of metadata: If the data pipeline doesn’t preserve schema metadata and does not allow for schema evolution, you end up tightly coupling the software producing the data at the source and the software that uses it at the destination
    * Extreme processing: Some processing of data is inherent to data pipelines. The recommendation is to preserve as much of the raw data as possible and allow downstream apps to make their own decisions regarding data processing and aggregation

### When to Use Kafka Connect Versus Producer and Consumer
When writing or reading from Kafka, you have the choice between using traditional producer and consumer clients, or using the Connect APIs and the connectors. Use Kafka clients when you can modify the code of the application that you want to connect an application to and when you want to either push data into Kafka or pull data from Kafka. Use Connect to connect Kafka to datastores that you did not write and whose code you cannot or will not modify.
If you need to connect Kafka to a datastore and a connector does not exist yet, you can choose between writing an app using the Kafka clients or the Connect API (recommended).

### Kafka Connect
Kafka Connect is a part of Apache Kafka and provides a scalable and reliable way to move data between Kafka and other datastores. Kafka Connect runs as a cluster of worker processes. You install the connector plugins on the workers and then use a REST API to configure and manage connectors, which run with a specific configuration. Connectors start additional tasks to move large amounts of data in parallel and use the avail‐ able resources on the worker nodes more efficiently.

#### Running Connect
Kafka Connect ships with Apache Kafka. For production use you should run Connect on separate servers. In this case, install Apache Kafka on all the machines, and simply start the brokers on some servers and start Connect on other servers. To start a connect worker do: `bin/connect-distributed.sh config/connect-distributed.properties`
Configuration properties to configure Connect:

    * bootstrap.servers: A list of Kafka brokers that Connect will work with
    * group.id: All workers with the same group ID are part of the same Connect cluster
    * key.converter and value.converter: The two configurations set the converter for the key and value part of the message that will be stored in Kafka (defaults to JSON)
    * key.converter.schema.enable and value.converter.schema.enable: Defines if the converter supports JSON messages with or without schema
    * key.converter.schema.registry.url and value.converter.schema.registry.url: Location of the schema registry
    * rest.host.name and rest.port: host and name of the Rest api
    
#### Connector Example: File Source and File Sink
To create a connector, we wrote a JSON that includes a connector name, load-kafka- config, and a connector configuration map, which includes the connector class, the file we want to load, and the topic we want to load the file into. Example:
`echo '{"name":"load-kafka-config", "config":{"connector.class":"FileStreamSource","file":"config/server.properties","topic":"kafka-config-topic"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"`
We can read back the example file shown before as:
`echo '{"name":"dump-kafka-config", "config":{"connector.class":"FileStreamSink","file":"copy-of-server-properties","topics":"kafka-config-topic"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"`
Which would store a file called 'copy-of-server-properties' in our local. To delete the connector simply do:
`curl -X DELETE http://localhost:8083/connectors/dump-kafka-config`

#### A Deeper Look at Connect
To understand how Connect works, you need to understand three basic concepts and how they interact

    * Connectors and tasks: Connector plugins implement the connector API, which includes two parts:
        - Connectors: The connectors determines how many tasks will run for the connector, decides how to split the data-copying work between the tasks and gets configurations for the tasks from the workers and passing it along
        - Tasks: Tasks are responsible for actually getting the data in and out of Kafka
    * Workers: Kafka Connect’s worker processes are the “container” processes that execute the connectors and tasks. They handle the HTTP requests that define connectors and their configuration, stores the connector configuration, starts the connectors and their tasks, and passes the appropriate configurations along. They are also responsible for automatically committing offsets for both source and sink connectors and for handling retries when tasks throw errors
    * Converters and Connect’s data model: Connect APIs includes a data API, which includes both data objects and a schema that describes that data. At the moment Avro object, JSON object, or a string formats are supported.
    * Offset management: The workers does the offset management for the connectors, the connectors use APIs provided by Kafka to maintain information on which events were already processed. Partitions and offsets are tied to the source type, these are not kafka specific.

#### Alternatives to Kafka Connect
Other alternatives to connectors might include:

    * Ingest Frameworks for Other Datastores: Hadoop and Elasticsearch has their own tools like Flume or Logstash
    * GUI-Based ETL Tools:  Informatica, Talend, Pentaho, Apache NiFi and StreamSets, support Apache Kafka as both a data source and a destination
    * Stream-Processing Frameworks: Almost all stream-processing frameworks include the ability to read events from Kafka and write them to a few other systems

## Chapter 8: Cross-Cluster Data Mirroring<a name="Chapter8"></a>
