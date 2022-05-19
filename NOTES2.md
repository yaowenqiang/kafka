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


