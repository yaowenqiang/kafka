> Add-hoc consumer

> bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic first  -partitions 2 ---repliction-factor 1

> bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic first  

> bin/kafka-console-producer.sh --broker-list localhost:9001  --topic first
>
> bin/kafka-console-consumer --zookeeper localhost:2181 --topic first
> bin/kafka-console-consumer --zookeeper localhost:2181 --topic first --from-beginning

