# partitions

+ Offset only have a meaning for a specific partition
  + E.g, offset 3 in partition 0 doesn't represent the same data as offset 3 in partition 1
+ Order is guaranteed only within a partition(not across partitions)
+ Data is kept only for a limited time(default is one week)
+ Once the data is written to a partition, it can't be changed(immutability)
+ data is assigned randomly to a partition unless a key is provided


# Brokers

+ A Kafka cluster is composed of multiple borkers(servers)
+ Each broker is identified with its ID(integer)
+ Each broker contains certain topic partitions
+ After connecting to any broker(called a bootstrap broker), you will be connected to the entire cluster
+ A good number to get started is 3 borkers, but some big clusters have over 100 brokers
+ Topics should have a replication factor > 1(usually between 2 and 3)
+ This way if a borker is down, another broker can serve the data
+ Example: Topic-A with 2 partitions and replication factor of 2

## Leader for a Partition

+ At any time only ONE broker can be a leader for a given partition
+ Only that leader can receive and serve data for a partition
+ the other brokers will synchronous the data
+ therefore each partition has one leader and multiple ISR)(in-sync replica)


## Producers

+ Producers write data to topics(which is made of partitions)
+ Producers automatically know to which broker and partition to write to
+ In case of Broker failures, Producers will automatically recover
+ Producers can choose to receive acknowledgment of data writes
  + acks=0: Producer won't  wait for acknowledgment(possible data loss)
  + ack=1: Producer will wait for leader acknowledgment(limited data loss)
  + ack=all: Leader + replicas acknowledgment(no data loss)


## Producers: Message Keys

Producers can choose to send a key with the message(string, number, etc)
If key = null, data is sent round robin(borker 101 then 102 then 103)
If key is dent, then all messages for that key will always go to the same partition
A key is basically sent if you need message ordering for a specific field(ex: truck_id)


## Consumers

Consumers read data from a topic(identified b y name)
Consumers know which broker to read from
In case of broker failures, consumers know how to recover
Data is read in order within each partitions

## consumers Groups

Consumers read data in consumer groups
Each consumer within a group read from exclusive partitions
If ou have more consumers than partitions, some consumers will be inactive

Note: Consumers will automaticlly use a GroupCoordinator and a ConsumerCoordinator to assign a consumer to a partition



## Consumer Offsets

Kafka stores the offsets at which a consumer group has been read
the offsets committed live in a kafka topic named __consumer_offsets
When a consumer in a group has processe data received from kafka, it should be commiting the offsets
if a consumer dies, it will be able to read to read back from where it left, off thanks to the committed consumer offsets!

### Delivery semantics for consumers

consumers choose when to commit offsets
There are 3 delivery semantics:

At most once:
  offsets are committed as soom as the message is received
  if the processing goes wrong ,the message will  be lost(it won't be read again)
At least once(usually preferred):
  offsets are committed after the message is processed
  If the processing goes wrong, the message will be read again
  This can result in duplicate processing of messages, Make sure your processing is idempotent(i.e processing again the messages won't impact your systems)
Exactly once
  Can be achived for Kafka => Kafka workflows using Kafka Streams API
For Kafka => external System workflow, use an idempotent consumer

## Kafka Broker Discovery 

Every kafka broker is also called a "bootstrap server"
That means that you only need to connect to one broker and you will be connected to the entire cluster
Each broker knows about all brokers, topics and partitions(metadata)

# Zookeeper

Zookeeper manages brokers(keeps a list of them)
Zookeeper helps in performing leader election for partitions
Zookeeper send dnotifications to Kafka in case of changes(e.g, new topic, broker dies, broker comes up, delete topcs, etc...)
Kafka can't work without Zookeeper
Zookeeper by design operates with an odd number of servers(3, 5, 7)
Zookeeper has a leader(handle writes) the rest of the servers are followers(handl reads)
)Zookeeper does NOT store consumere offsets with Kafka > v0.10)

## Kafka Guarantees

Messages are appended to a topic-partition in the order the are sent
Consumers read messages in the order stored in a topic-partition
With a replication factor of N, producers and consumers can tolerate up to N-1 brokers being down
This is why a replication factor off 3 is a good idea:
  Allows for one broker to be taken down for maintence
  Allows for another broker to be taken down unexpectedly
As long as the number of partitions remains constant for a topic(no new partition), the same key will always go the the same partition

## Install

> wget https://dlcdn.apache.org/kafka/3.2.0/kafka_2.13-3.2.0.tgz

brew install kafka
brew cask install java8


> sudo apt install openjdk-8-jdk


## basic operations
> kafka-console-producer.sh --zookeeper  localhost:2181 --topic first_topic --from-beginning
> kafka-console-consumer --broker-list localhost:9092 --topic first_topic --producer-property acks=all

## Consumer Group

> kafka-console-consumer --broker-list localhost:9092 --topic first_topic --producer-property acks=all --groujp my-first-appliction

> ./bin/kafka-console-consumer.sh --consumer.config config/consumer.properties   --topic third_topic --zookeeper localhost:2181
> ./bin/kafka-consumer-group.sh --list --zookeeper localhost:2181
> ./bin/kafka-console-consumer.sh   --topic third_topic --zookeeper localhost:2181   --consumer.config config/consumer2.properties  --from-beginning  --delete-consumer-offsets
> kafka-consumer-groups.sh  --zookeeper localhost:2181  -describe --group test-consumer-group

## UI Tools

Kafka Tool


## Client Bi-Directional Compatibility 

As of kafka 0.10.2(introudce in July 2017), your clients & Kafka Brokers have a capability called bi-directional Compatibility (because API calls are now versioned)

This means

An OLDER client(ex 1.1) can talk to a NEWER broker(2.0)
A NEWER client(ex 2.0) can talk to an OLDER broker(1.1)
Bottom Line: always use the latest client library version if you can
> https://www.confluent.io/blog/upgrading-apache-kafka-clients-just-got-easier/


## Producer Acks Deep Dive

ack = 0 (no acks)
No response is request
If the broker goes offline or an exception, hadppends, we won't know adn will lose data
Useful for data where its' okay to potentially lose messages
  metricx collection
  Log collection

asks = 1(leader acks, default)

Leader response is requested, but replication is not a guarantee(happends in the background)
If an ack is not received, the producer may retry

If the leader broker goes offline but replicas haven't replicated the the data ye , we have a data lose

acks = all(replica acks)

Leader + rReplicas ack requested
Added latency and safety
No data loss if enough replicas
Necessary setting if you don't want to lose data
Acks=all must be used in conjunction with min.insync.replicas
min.insync.replicas can be set at the broker or topic level(override)
min.insync.replicas-2 implies that at least 2 brokers that are ISR (including leader)must respond that they have the data,
that means if you use replication.factor=3, min.insync=2, acks=all, you can only tolerate 1 broker going down, otherwise the producer will receive an exception on send.(No_ENOUGH_REPLICAS)

## Producer retries

In case of transient failures, developers are expected to handle exceptions, otherwise the datta will be lost.
Example of transient failures:
    NotEnoughtReplicasException
There is a "retries" setting
    default to 0
    You can increase to a high number ,ex integer.MAX_VALUE

In case of retries, by default, there is a change that messages will bbe sent out of order(if a batch has failed to be sent)
If you rely on key-based ordering, that can be an issue
for this ,you can set the setting while controls how many produce requests can be made in parallel: max.in.flight.requests.per.connection
    Default 5
    Set it to 1 if you need to ensuere ordering(may inpace throughput)
In Kafka >= 1.0.0, there's a better solution!

## Idempotent(幂等) Producer

Here's the problem, the Producer can introduce duplidate messages in Kafka due to network errors
In Kafka >= 0.11.1, you can efine a "idempotent producer" which won't introduce duplicates on network error
> produce_id
idmpotent producers are great to guarantee a stable and safe pipoeline!
They come with:
    retries=Integer.MAX_VALUE(2*31-1 = 2147483647)
    max.in.flight.request=1(Kafka >= 0.1 & < 1.1) or
    max.in.flight.requests=5(Kafka >=1.1 - higher performance)
    acks=all
Just set:
    producerProps.put("idempotentence", true);


## Message Compression

Producer usually send dat athat is text-based, for example with JSON data
In this case ,it is important to apply compression  to the producer
Compressing is enable at the Producer level and doesn't require any configuration change in the Brokers or in the Consumers
"Compression-type" can be "none"(default) "gzip", 'Az4", "snappy"
Compresion is more effective the bigger the batch of message being send to Kafka!

benchmarks here:  https://blog.cloudflare.com/squeezing-the-firehose/


Teh compressed batch has the following advantage:
    Much smaller producer request size(compression ratio up to 4x!)
    Faster to transfer data over the network => less latance
    Better throughput
    Better disk utilisation in Kafka(stored messages on disk are smaller)
Disdavantages(very minor):
    Producers muct commit some CPU cycle to compression
    Consumers must commit some CPu cycles to decompression
OVerall:
    consider testing snappy or lz4 for optional speed/ compression ration


Message Compression Recommendations:

    Find a compression algorithm that gives you the best performance for your specific data, Test all of them!
    Always use compression in production and especially if you have high throughput
    considering tweaking linger.ms and batch.size to have bigger batches, and therefore more compression and higher throughput.

## Linger.ms & batch.size

By default, Kafka tries to send records as soon as possible
It will have up to 5 requests in flight, meaning up to 5 messages individually send at the same time
After this, if more messages have to be send while others are in flight, Kafka is smart and will start batching them while they wait to send them all at once

This smart batching allows Kafka to increase throughput while maintaining very low latency
Batch have higher compression ratio so better effciency

Linger.ms: Number of milliseconds a producer is willing to wait before sending a batch out.(default 0)
by introducing some lag(for example linger.ms=5), we increase the chances of messages being sent together in a batch
So at the expense of introducing a small delay, we can increase throughput, compression and efficiency of our producer
If a batch is full(see batch.size) before the end of the liger.ms period, iw it will be send to Kafka right awa


batchsize: Maximum number of bytes that will be included in a batch, The default is 16KB

Increasing a batch size to something like 32KB or 64KB can help increasing thet compression, throughput, and efficiency of requests
Any message that is bigger than the batch size will not be batched
A batch is allocated per partition, so make sure that don't set it to a number that's too high, other wise you'll run waste memory!
(Note: You can monitor the average batch size metric using kafka Producer Metrics)


## High throughput Producer Demo

we'll add snappy message compression in our producer
snappy is very helpful if your messages are text based, for example log lines or JSON documents
snappy made by goole
snappy has a good balance of CPU / compression ratio
We'll also increase the batch.size to 32JB and introduce a small delay through linger.ms(20 ms)


## Producer Default partitioner and how keys are hashed

By default, your keys are hashed using the "murmur2" algorithm
It is most likely perfered to not override the behavior of the partitioner, but it is possible to do so (partitioner.class)
the formula is :
targetPartition = Utils.abs(Utils.murmer2(record.key()) % numPartitions
This means that same key will go to the same partition(we already know this), and adding partitions to a topic will completely alter the formula

## Max.block.ms & buffer.emory

If the producer produces faster than the broker can tke ,the records will be buffered in memory
buffer.memory=33554432(32MB): the size of the send buffer
That buffer will fill  up over time and fill back down when the throughput to the broker increases
If that buffer is full(all 32MB), then the send() method will start to block(won't return right away)
max.block.ms=6000; the tiem the send() will block untill throwing an exception, Exceptios are basically thrown then 
    The producer has filled up its buffer
    The broker is not accepting any new data
    60 seconds has elapsed
If you hit an exception hit that usually means your brokers are down, or overloaded as they can't respond to requests

## elastic search and Kafka
> https://wwwbonsai.io/

> GET /
> GET /_cat/health?v
> GET /_cat/nodes?v
> GET /_cat/indices?v

> PUT /customer/?pretty
> PUT /twitter/tweets/

{
    "source": "Kafka for Beginners",
    "Instructor": "Stephene Maarek",
    "Module": "ElasticSearch",
}

>
> GET /twitter/tweets/1

> DELETE /twitter/

> GET localhost:9200/twitter/tweets/fR3m-4ABruG2PoHQEZSX

## Delivery Semantic At Most Once
+ At most once: offsets are committed as soon as the message batch is received, if the processing goes wrong, the message will be lost,(it won't be read again)

## Delivery Semantic At Least  Once
+ At least once: offsets are committed after the message is prcessed. If the processing  goes wrong, the message will be read again, This can result in duplicate processing of messages. Make sure your processing is idempotent.(i.e processing againg the messages won't impact your systems)

## Delivery Semantic Exactly once:
+ Exactly once: Can be achieved for Kafka => Kafka workflows using Kafka Streams API, For Kafka => Sink workflows, use an idempotent consumer.

# Bottom Line: for most applications you should use at least once processing and ensure your transformations/processing are idempotent





> ssh -vvv git@github.com

# Consumer Poll Behavior

+ Kafka Comsumers have a "poll" model, while many other messaging, but in enterprises have a "push" model
+ This allows consumers to control where in the log they want to consume, how fast, and gives them the ability to replay eventsw

### Consumer Poll behavior


+ Fetch.min.bytes(default 1);
  + Controls how much data you want to pull at least on each request
  + Helps improving throughput and decreasing request number
  + At the cost of latency
+ Max.poll.records(default 500):
  + Controls how many records to receive per poll request
  + Increase if your messages are very small and have a lot of avaiable RAM
  + Good to monitor how many records are polled per request
+ Max.partitions.fetch.bytes(default 1MB):
  + Maximum data returned by the broker per partition
  + If you read from 100 partitions, you'll need a lot of memory(RAM)
+ Fetch.max.bytes(default 50MB):
  + Maximum data returned fro each fetch request(covers multiple partitions)
  + The consumer performs multiple fetches in parallel

> Change these settings only if your consumer maxes out on throughput already
o


## Consumer offset Commits Strategies

+ There are tow most common paatterns for committing offsets in a consumer aapplication

2 strategies:

+ (easy) enable auto.commit = true & synchronous processing of batches
+ (medium) enable.auto.commit = false & manual commit of offsets

'''
while(true) {
    List<Recores> batch = consumer.poll(Duration.ofMillis(100))
    doSomethingSynchronous(batch)
}


'''


+ With auto-commit, iffsets will be committed automatically for you at regualr interval(auto.commit.interval.ms=5000 by default0 every-time you call .poll()
+ If you dno't use synchronous processing, you will be in "at-mose-once" behaviuor because offsets will be committed before your data is processed



'''
> enable.auto.commit = false & synchronous processing of batches
while(true) {
    batch += consumer.poll(Duration.ofMillis(100))
    if (isReady(batch)) {
        doSomethingSynchronous(batch);
        consumer.commitSync();
    }
}

+ You control when you commit offsets and what's the condition for committing them.

Examples: accumulating recordes into a buffer and then flushing the buffer to a database + commiting offsets then

## Consumer Offset Reset behavior

The behavor for the consumer is to then use:

+ auto.offset.reset=latest; will read from the end of the log
+ auto.offset.reset=earliest;; will read from the start of the log
+ auto.offset.reset=none;; will throw exception of no offset is found

### additionally, consumer offsets can be lost:

+ If a consumer hasn't read new data in 1 dat(Kafka < 2.0)
+ If a consumer hasn't read new data in 7 dat(Kafka >= 2.0)

This can be controlled by the broker setting offset.retention.minutes

## Replaying data for Consummers:

+ To replay data for a consumer group:
  + Take all the consumers from a specific group down
  + Use "kafka-consumer-groups" command to set offset to what you want
  + Restart consumers
+ bottom line:
  + Set proper data retention period & offset retention period
  + Ensure the auto offset reset behavior is the one you expect/want
  + Use replay capability in case of unexpected behavior

> kafka-consumer-groups --bootstrap-server localhost:9092 --group groupname --reset-offsets --execute --to-earliest --toipc topic_name # reset topic offsets

## Controlling Consumer Liveliness

+ Consumers in a Group talk to a Consumer Groups Coordinator
| To detect consumers that are ""down"" there is a ""heartbeat" "mechanism and a ""poll"" mechanism
+ To avoid issues, consumers are encouraged to process data fast and poll offen

### consumer Heartbeat Thread

+ Session.timeout.ms(default 10 seconds):
  + Heartbeats are sent periodically to the broker
  + If no heartbeat is sent during that period, the consumer is considered dead
  + Set even lower to faster consumer rebalances
+ Heartbeat.interval.ms(default 3 seconds):
  + How often to send heartbeats
  + Usually set to 1/3rd of session.timeout.ms
+ Take-away: This mechanism is used to detect a consumer application being down

### ConsumerPoll Thread

+ max.poll.interval.ms(default 5 minutes):
  + maximum amount of time between two poll()) calls before declaring the consumer dead
  + This is particularly relevant for Big Data fraeworks like Spark in case the processing takes time
+ Take-away: This mechanism is used to detect a data processing issue with the consumer

