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






