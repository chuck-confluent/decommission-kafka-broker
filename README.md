Work in progress

# Kafka Elasticity

Apache Kafka supports adding and subtracting brokers to respond to changes in demand. However, this must be done carefully. This repo demonstrates the steps required to expand and shrink a Kafka cluster.

Confluent has re-architected Apache Kafka to offer cloud-native elasticity in Confluent Cloud. Compare the steps here to the experience of using Confluent Cloud, which supports autoscaling for basic and standard clusters, and a 1-click scale up/down experience with dedicated clusters.


## Cheat sheet

### Start Cluster

```
docker compose up zookeeper broker1 broker2 broker3 -d
```

### List topics
```
docker compose exec broker1 \
    kafka-topics \
    --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
    --list
```

### Produce

Create topic `test`.
```
kafka-topics \
    --bootstrap-server broker1:29091 \
    --create \
    --topic test \      
    --partitions 8 \
    --replication-factor 3
```

Produce to topic `test`.

```
docker compose exec broker1 \
    kafka-producer-perf-test \
        --topic test \
        --num-records 50000000 \
        --record-size 100 \
        --throughput 1 \
        --producer-props bootstrap.servers=broker1:29091,broker2:29092,broker3:29093
```

### Consume

```
docker compose exec broker1 \
    kafka-console-consumer \
        --topic test \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093
```


## Steps to Add a Broker

### Start Broker

```
docker-compose up -d broker4
```

### Manually Reassign Partitions (with throttle)

```
docker compose exec broker1 \
    kafka-topics \
    --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
    --describe --topic test
```

### Remove throttle

## Steps to Decommission a broker
### Pick a broker to decommission (don't kill the controller!)

### Find out the partitions for which that broker is the leader

### Manually Reassign Partitions (with throttle)

### Remove Throttle

### Gracefully kill broker
