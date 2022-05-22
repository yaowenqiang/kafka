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
+ the other brokers will synchronize the data
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
> kafka-consumer-groups.sh  --zookeeper localhost:2181  --describe --group test-consumer-group

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






