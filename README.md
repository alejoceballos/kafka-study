# Kafka Study

Based on Udemy's [Apache Kafka Series - Learn Apache Kafka for Beginners v3](https://www.udemy.com/course/apache-kafka/) 
course.

## Index
- [What is Kafka?](#what-is-kafka)
- [Topics](#topics)
- [Partitions](#partitions)
- [Offsets](#offsets)
- [Brokers](#brokers)
- [Producers](#producers)
    - [Partition Logic](#partition-logic)
- [Consumers](#consumers)
  - [Consumer Group](#consumer-group)
  - [Consumer Offsets](#consumer-offsets)
- [Messages](#messages)
  - [Producer Message](#producer-message)
  - [Messages Serializer](#message-serializer)

## What is Kafka?

What is Kafka? A "middleware" tool between sources and targets that centralizes the data transfer between them in the 
form of streams that is manageable and can be monitored.

![Kafka Overview](./README.files/Kafka-Study-Overview.png)

### Topics

**What are topics?** Topics are the main mechanism of communication used to store messages, in the form of streams, 
created by producers and used by consumers. Summarizing, topics hold data.

Comparing with relational databases, you can say that a topic is a table, an insert statement is a producer, and a 
select statement is a consumer. But of course, with a lot of extra features in the between.

Kafka topics are **immutable**, it means that once data is written to a topic's partition, it cannot be changed.

Data in topics is kept for a limited amount of time. The default is 1 week.

Topics are split in partitions.

### Partitions

**What are partitions?** Partitions are logical and physical ways to split the data being handled. This allows load 
balancing and automatic recovery in case of failure. 

Each message is indexed in its partition, this indices are called _offsets_.

Data written to a topic's partition is immutable. You cannot delete or update this data.

Messages are ordered inside each partition, but not across partitions. Although there are mechanisms to achieve ordering 
between partitions.

Data can be randomly assigned to the partition (using a round-robin algorithm). To prevent this behavior a key can be 
assigned to a partition and identified when adding data. Messages with the same key will end up in the same partition 
thanks to a hash calculation.

There is no limit how many partitions can fit into a topic.

### Offsets

**What are offsets?** Offset is the message's incremental ID in each partition.

Different messages in different partitions of the same topic may have the same offset value, but they are not related.

Offsets are not reused, even if its related message had been removed.  

### Brokers

**What are brokers?** They are Kafka SERVERS.

They can fail, but Kafka allows producers to recover from brokers failures.

### Producers

**What are producers?** Is a program that writes/produces data to a topic.

Producers must know in advance:
- Which partition to write data to (??? but when explaining partitions the author leads us to believe that the default 
    behavior is Kafka assign the message to a pa0rtition randomly)
- Which broker has this partition

In case a broker is down, Kafka allows the producer to recover from it.

Producers can assign a key to a message, this key will be related to a specific partition. If no key is assigned, the 
partition will be chosen by a round-robin mechanism (partition 0, then 1, the 2, ...) 

#### Partition Logic

When sending a message to a broker, the producer uses a hash algorithm 
([mumur2](https://en.wikipedia.org/wiki/MurmurHash)) applied to the already serialized key to identify the partition it 
will send the message to.

![Producer Serialization and Partition Identification](./README.files/Kafka-Study-Producer.png)

### Consumers

**What are consumers?** Consumers are responsible for pulling (read/consume) the data from topics.

Since consumers know the topics they are consuming from, they have the information regarding the broker (server).

Consumers don't stop working in case a broker fails.

Messages are consumed in the order they were inserted in the partition (FIFO), but not in the order they were inserted 
between partitions. 

So if: 
```
a) message 1 is pushed first to partition 1, offset 1
b) message 2 is pushed to partition 1, offset 2
c) message 3 is pushed to partition 2, offset 1
```
It is guaranteed that message 1 will be read before than message 2, but not before message 3.

After grabbing the message, the consumer deserializes the Key and Value, so it must know in advance what type of 
deserializer it must use. 

**NOTE:** The consumer knows what to expect because it is subscribed to a topic of this message. The topic deals with a 
single type of messages, so producers and consumers know what type of serializers and deserializers must use when 
dealing with the topic.

The topic must not change the type of the Key and Message after created. Doing so will prevent consumers from being able
to deserialize the messages. In case of changing topic's messages types, a new topic must be created and only then 
redirect the consumers to consume from this new topic.

#### Consumer Group

A consumer group is a group of consumers responsible for reading from a topic where each of the consumers is responsible 
for one or more partitions to read. Within a partition group, two consumers cannot read from the same partition. 
However, it is possible to have multiple partition groups consuming from the same topic, and in this case it is possible 
that consumers from different groups consume from the same partition. If there are too many consumers in the group for 
the amount of partitions in a topic, then the extra consumer becomes **inactive**.

![Consumer Group](./README.files/Kafka-Study-Consumer-Group.png)

Set the group the consumer belongs by setting its "group.id" property.

Why have multiple consumer groups? Responsibility segregation. The same data can be used for different types of 
services. For example: a truck tracking system can have two services, notification and location, reading from the truck 
location topic. The first is used to notify in case it arrives at some checkpoint, the other to plot the truck in a map 
in real time.

#### Consumer Offsets

Consumer groups periodically "commit" the offsets it has completed reading in order to be able to continue from the last 
read offset in case of any error. Kafka keeps track of each consumer group offset reading storing this checkpoint in a 
specific topic called _\_\_consumer_offsets_.

### Messages

The data that navigates from producers to consumers.

Messages in each partition are ordered (indexed).

Messages can have keys assigned by producers. These keys are associated to a specific partition.

Messages with the same key will end up in the same partition thanks to a hashing calculation.

#### Producer message

Messages created by producers:
- Key: binary (nullable)
- Value or message content: binary (nullable)
- Compression type: gzip, snappy, lz4, zstd (can be none)
- Headers: key/value (optional)
- Partition + Offset 
- Timestamp 

#### Message Serializer

Kafka does not accept other values than binary, so the Key and Value are serialized by the producer.

There are several serializers in Kafka responsible to transform integers, strings and so on into binary, like the basic 
ones (String, Integer, Float, etc.), and the more complex ones (Apache Avro, Google Protocol Buffers/a.k.a. Protobuf, 
etc.). 

