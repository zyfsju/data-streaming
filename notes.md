Q: What is the relationship between topic and partition?

A: [Reference](https://www.instaclustr.com/the-power-of-kafka-partitions-how-to-get-the-most-out-of-your-kafka-cluster/)

-   Each topic can have 1 or more partitions. There is no theoretical upper limit. You can request as many partitions as you like, but there are practical limits.
-   The size (in terms of messages stored) of partitions is limited to what can fit on a single node. If you have more data in a topic than can fit on a single node you must increase the number of partitions. Partitions are spread across the nodes in a Kafka cluster.
-   Message ordering in Kafka is per partition only.
-   If you are using an (optional) message key (required for event ordering within partitions, otherwise events are round-robin load balanced across the partitions – and therefore not ordered), then you need to ensure you have many more distinct keys (> 20 is a good start) than partitions otherwise partitions may get unbalanced, and in some cases may not even have any messages (due to hash collisions).
-   Partitions can have copies to increase durability and availability, and enables Kafka to failover to a broker with a replica of the partition if the broker with the leader partition fails. This is called the Replication Factor, and can be 1 or more.

Q: docker-compose running Kafka?

A: [Reference](https://docs.confluent.io/platform/current/kafka/multi-node.html)

Q: How does the producer decide to which partition to send a message?

A: The producer picks which partition a record goes to. By default, records with the same key get sent to the same partition. The default partition strategy for producers without using a key is round-robin.

Q: How is load balancing implemented on the side of the producer, and the consumer, when a topic has multiple partitions?

### Stream Processing

#### Log-structured Storage

Q: How are append-only logs used in SQL databases?

A: SQL databases use append-only logs to track all creations, updates, and deletes made to the database, so that those changes can be synced to replicas

Append-only logs

-   Append-only logs are text files in which incoming events are written to the end of the log as they are received.
-   This simple concept -- of only ever appending, or adding, data to the end of a log file -- is what allows stream processing applications to ensure that events are ordered correctly even at high throughput and scale.
-   We can take this idea a step farther, and say that in fact, streams are append-only logs.

Log-structured streaming

-   Log-structured streams build upon the concept of append-only logs. One of the hallmarks of log-structured storage systems is that at their core they utilize append-only logs.
-   Common characteristics of all log-structured storage systems are that they simply append data to log files on disk.
-   These log files may store data indefinitely, for a specific time period, or until a specific size is reached.
-   There are typically many log files on disk, and these log files are merged and compacted occasionally.
-   When a log file is merged it means that two or more log files are joined together into one log file.
-   When a log file is compacted it means that data from one or more files is deleted. Deletion is typically determined by the age of a record. The oldest records are removed, while the newest stay.
-   Examples of real world log-structured data stores: Apache HBase, Apache Cassandra, Apache Kafka

### Apache Kafka

Benefits of partitions

-   Increased parallelism for consumers and producers
-   Higher throughput

#### Topic Data Management

#### Producer Configuration

Options

-   All available settings for the confluent_kafka_python library can be found in the [librdkafka configuration options](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md). confluent_kafka_python uses librdkafka under the hood and shares the exact configuration options in this document.
-   It is a good idea to always set the `client.id` for improved logging, debugging, and resource limiting
-   The `retries` setting determines how many times the producer will attempt to send a message before marking it as failed
-   If ordering guarantees are important to your application and you’ve also enabled retries, make sure that you set `enable.idempotence` to true
-   Producers may choose to compress messages with the compression.type setting
    -   Options are none, gzip, lz4, snappy, and zstd
    -   Compression is performed by the producer client if enabled
    -   If the topic has its own compression setting, it must match the producer setting, otherwise the broker will decompress and recompress the message into its configured format.
    -   The `acks` setting determines how many In-Sync Replica (ISR) Brokers need to have successfully received the message from the client before moving on
    -   A setting of `-1` or `all` means that all ISRs will have successfully received the message before the producer proceeds
    -   Clients may opt to set this to 0 for performance reasons
-   The diagram below illustrates how the topic and producer may have different compression settings. However, the setting at the topic level will always be what the consumer sees.

#### Consumer Offsets

Kafka consumers subscribe to one or more topics

-   Kafka keeps track of what data a consumer has seen with offsets
-   Kafka stores offsets in a private internal topic
-   Most client libraries automatically send offsets to Kafka for you on a periodic basis
-   You may opt to commit offsets yourself, but it is not recommended unless there is a specific use-case.
-   Offsets may be sent synchronously or asynchronously
-   Committed offsets determine where the consumer will start up
    -   If you want the consumer to start from the first known message, set `auto.offset.reset` to `earliest`
    -   This will only work the first time you start your consumer. On subsequent restarts it will pick up wherever it left off
    -   If you always want your consumer to start from the earliest known message, you must manually assign your consumer to the start of the topic on boot

Q: Which of the following best describes what a consumer offset is?

A: A number stored in a private Kafka topic which identifies the last consumed message for a consumer.

Q: How does Kafka keep track of consumer offsets?

A: It maintains an internal topic which tracks committed offsets from consumers.

#### Consumer Groups

-   All Kafka Consumers belong to a Consumer group. `group.id` parameter is required. Consumer groups consist of one or more consumers
-   Consumer groups increase throughput and processing speed by allowing many consumers of topic data. However, only one consumer in the consumer group receives any given message
-   Create a consumer group with a single member, if your application needs to inspect every message in a topic
-   Adding or removing consumers causes Kafka to rebalance
    -   During a rebalance, a broker group coordinator identifies a consumer group leader
    -   The consumer group leader reassigns partitions to the current consumer group members
    -   During a rebalance, messages may not be processed or consumed

### Data Schemas and Apache Avro

-   Data schemas make consumers and producers more resilient to change.
-   Schemas decrease coupling between applications
-   create more efficient representations with compression

Use cases:

-   The Hadoop ecosystem uses defined schemas to load data: Avro
-   Kubernetes uses gRPC to communicate with system components

Why schemas matter

-   Data streams are constantly evolving
-   No schema = broken consumer on every data change
-   Schemas allow consumers to function without updates
-   Schemas provide independence and scalability
-   Schemas can communicate version compatibility

Apache Avro is a data serialization system that uses binary compression

Q: Required fields for Avro records?

A: name, fields, type

#### Schema Registry - Summary

-   Provides an HTTP REST API for managing Avro schemas
-   Reduces network overhead, allowing producers and consumers to register schemas one time
-   Uses a Kafka topic to store state
-   Deployed as one or more web servers, with one leader
-   Uses Zookeeper to manage elections

### Kafka Connect and REST Proxy

#### Kafka Connect

web server and framework built on the JVM for integrating Kafka with external data sources such as SQL databases, log files and HTTP endpoints.

Choose to use Kafka Connect rather than a Kafka client library:

-   Saves time
-   Allows users to repeatedly implement similar Kafka integrations
-   Provides an abstraction from Kafka for application code
-   Decreases amount of code to maintain

#### Kafka Connect Architecture

Kafka Connect uses Kafka as its configuration store and uses Framework JARs (jdbc, text, http, s3) to provide functionality

-   Connectors are abstractions for managing tasks
-   Tasks contain the production or consumption code
-   Kafka and target systems often have different formats
-   Converters map data formats to and from Connect

#### Kafka Connect - Summary

-   Can be used to handle common and repeated scenarios
-   web-server written in Java and Scala, and runs on the JVM
-   Has a plugin architecture, meaning that you can easily write your own connectors in addition to using the rich open-source ecosystem of connectors
-   Isolate your application entirely from integrating with a Kafka client library

Q: In which of the following scenarios would you not want to use Kafka Connect?

A: The data needs to be transformed before being stored in Kafka; The data must be sent as it occurs rather than being sent periodically by Kafka Connect

#### The Kafka Connect API

-   CRUD Connector configuration via a REST API
-   Check the status of a task via the API
-   Start, stop and restart Connectors via the API

[See the documentation for more information on any of these actions](https://docs.confluent.io/current/connect/references/restapi.html)

First, we can view connector-plugins:

`curl http://localhost:8083/connector-plugins | python -m json.tool`

or use `jq` to format the JSON output

##### Create a Connector

Lets create a connector. We'll dive into more details on how this works later.

```
curl -X POST -H 'Content-Type: application/json' -d '{
    "name": "first-connector",
    "config": {
        "connector.class": "FileStreamSource",
        "tasks.max": 1,
        "file": "/var/log/journal/confluent-kafka-connect.service.log",
        "topic": "kafka-connect-logs"
    }
  }' \
  http://localhost:8083/connectors
```

##### List connectors

We can list all configured connectors with:

`curl http://localhost:8083/connectors | python -m json.tool`

You can see our connector in the list.

##### Detailing connectors

Let's list details on our connector:

`curl http://localhost:8083/connectors/first-connector | python -m json.tool`

##### Pausing connectors

Sometimes its desirable to pause or restart connectors:

To pause:

`curl -X PUT http://localhost:8083/connectors/first-connector/pause`

To restart:

`curl -X POST http://localhost:8083/connectors/first-connector/restart`

##### Deleting connectors

Finally, to delete your connector:

`curl -X DELETE http://localhost:8083/connectors/first-connector`

#### Kafka Connect Connectors - Summary

-   Kafka Connect supports a number of Connectors for common data sources
    -   File on disk
    -   Amazon S3 and Google Cloud Storage
    -   SQL databases such as MySQL and Postgres
    -   HDFS
-   Kafka Connect has an extensive REST API for managing and creating Connectors

#### REST Proxy Architecture

-   REST Proxy is a web server built in Java and Scala that allows any client capable of HTTP to integrate with Kafka
-   REST Proxy cannot create topics, not topic configuration changes, nor modify existing data
-   Can fetch administrative and metadata information about the cluster
-   Can consume from topics
-   Can produce data to an existing topic
-   Use REST Proxy when working on a frontend client without a native Kafka library, or a legacy application that supports HTTP but can't add new dependencies

Q: In what scenarios would you not want to use REST Proxy?

A:

-   Put logs into Kafka from a server without a Kafka integration - Use Kafka Connect
-   When writing a Python/Java application that can be updated to use new dependencies

#### Using REST Proxy

`POST` data to `/topics/<topic_name>` to produce data

#### Consuming Data with REST Proxy

-   `POST` to `/consumers/<group_name>` to create a consumer group
-   `POST` to `/consumers/<group_name>/instances/<instance_id>/subscriptions` to create a subscription
-   `GET` to `/consumers/<group_name>/instances/<instance_id>/records` to retrieve records

-

### KSQL

### Project

## Spark

Pros

-   Speed
-   Ease of Use
-   Advanced Analytics
-   Dynamic in Nature
-   Multilingual
-   Apache Spark is powerful
-   Increased access to Big data
-   Demand for Spark Developers
-   Open-source community

Cons

-   No automatic optimization process
-   File Management System
-   Fewer Algorithms
-   Small Files Issue
-   Window Criteria
-   Doesn’t suit for a multi-user environment

Tuning:
[https://developer.hpe.com/blog/MPxYKrwlr5SZQGoLnZ2X/performance-tuning-of-an-apache-kafkaspark-streaming-system](https://developer.hpe.com/blog/MPxYKrwlr5SZQGoLnZ2X/performance-tuning-of-an-apache-kafkaspark-streaming-system)

In a Spark Streaming application, the stream is said to be stable if the processing time of each microbatch is equal to or less than the batch time.

### Intro to Spark Streaming

#### Intro to Spark RDDs

-   RDD stands for Resilient Distributed Dataset:
-   Resilient because its fault-tolerance comes from maintaining RDD lineage, so even with loss during the operations, you can always go back to where the operation was lost.
-   Distributed because the data is distributed across many partitions and workers.
-   Dataset is a collection of partitioned data. RDD has characteristics like in-memory, immutability, lazily evaluated, cacheable, and typed (we don't see this much in Python, but you'd see this in Scala or Java).
