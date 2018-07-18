# Running Chaos engineering

To get this working properly you must add the following lines to your /etc/hosts file
```
127.0.0.1       kafka-1
127.0.0.1       kafka-2
127.0.0.1       kafka-3
127.0.0.1       zookeeper-1
127.0.0.1       zookeeper-2
127.0.0.1       zookeeper-3
```

You also need the Kafka Cli tools found here - http://kafka.apache.org/
Just download it for the .sh cli tools and add them to the path

## Example scenario
Create a topic with a replication factor of two using brokers 1,2, initiate a producer and a consumer for it 

```
./kafka-topics.sh --create --zookeeper localhost:22181 --topic test_kafka_1 --config retention.ms=604800000 --replica-assignment 1:2,1:2,1:2,1:2
./kafka-topics.sh --describe --zookeeper localhost:22181 --topic test_kafka_1
 Topic:test_kafka_1     PartitionCount:4        ReplicationFactor:2     Configs:retention.ms=604800000
        Topic: test_kafka_1     Partition: 0    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: test_kafka_1     Partition: 1    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: test_kafka_1     Partition: 2    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: test_kafka_1     Partition: 3    Leader: 1       Replicas: 1,2   Isr: 1,2

./kafka-producer-perf-test.sh --topic test_kafka_1 --num-records 100000 --record-size 1000 --throughput 1 --producer-props acks=1 bootstrap.servers=localhost:19092,localhost:29092,localhost:39092 buffer.memory=67108864 compression.type=none batch.size=8196
./kafka-console-consumer --bootstrap-server localhost:19092  --topic test_kafka_1 --from-beginning --group cnsgroup1
```
Check kafka_topic_partition_under_replicated_partition metric

```
docker kill kafka-2
```

Check kafka_topic_partition_under_replicated_partition metric again
Kafka_topic_partition_under_replicated_partition spiked (one node is missing)

```
docker-compose -f docker-compose.yml up -d
```

Check kafka_topic_partition_under_replicated_partition metric again
Kafka_topic_partition_under_replicated_partition returns to 0

```
docker kill kafka-2
```

Check kafka_topic_partition_under_replicated_partition metric
```
./kafka-topics.sh --describe --zookeeper localhost:22181 --topic test_kafka_2
 Topic:test_kafka_2	PartitionCount:4	ReplicationFactor:2	Configs:retention.ms=604800000
	Topic: test_kafka_2	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: test_kafka_2	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: test_kafka_2	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: test_kafka_2	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1
```

Kafka is not self healing.... if we want to repair that we need to create a JSON reassignment file - 
```
{"version":1,
 "partitions":[{"topic":"test_kafka_1","partition":0,"replicas":[1,3]},
               {"topic":"test_kafka_1","partition":1,"replicas":[1,3]},
               {"topic":"test_kafka_1","partition":2,"replicas":[1,3]},
               {"topic":"test_kafka_1","partition":3,"replicas":[1,3]}]
}
```
And then issue the following command -

```
./kafka-reassign-partitions.sh --zookeeper localhost:22181 --reassignment-json-file reassignment.json --execute
./kafka-topics.sh --describe --zookeeper localhost:22181 --topic test_kafka_1

 Topic:test_kafka_1	PartitionCount:4	ReplicationFactor:2	Configs:retention.ms=604800000
	Topic: test_kafka_1	Partition: 0	Leader: 1	Replicas: 1,3	Isr: 1,3
	Topic: test_kafka_1	Partition: 1	Leader: 1	Replicas: 1,3	Isr: 1,3
	Topic: test_kafka_1	Partition: 2	Leader: 1	Replicas: 1,3	Isr: 1,3
	Topic: test_kafka_1	Partition: 3	Leader: 1	Replicas: 1,3	Isr: 1,3
```

Check kafka_topic_partition_under_replicated_partition metric
