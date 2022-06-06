Work in progress

# Kafka Elasticity

Apache Kafka supports adding and subtracting brokers to respond to changes in demand. However, this must be done carefully. This repo demonstrates the steps required to expand and shrink a Kafka cluster.

Confluent has re-architected Apache Kafka to offer cloud-native elasticity in Confluent Cloud. Compare the steps here to the experience of using Confluent Cloud, which supports autoscaling for basic and standard clusters, and a 1-click scale up/down experience with dedicated clusters.


## Initialize Cluster and Topic

### Start Cluster

```
docker compose up zookeeper broker1 broker2 broker3 -d
```

### Create Topic

Create topic `test`.
```
docker compose exec broker1 \
    kafka-topics \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --create \
        --topic test \
        --partitions 8 \
        --replication-factor 3
```

Describe the topic.

```
docker compose exec broker1 \
    kafka-topics \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --describe \
        --topic test
```
### Produce

Produce to topic `test` in a terminal on the left.

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
Consume from topic `test` in a terminal on the right.

```
docker compose exec broker1 \
    kafka-console-consumer \
        --topic test \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093
```


## Steps to Add a Broker

### Start Broker

Usually this would be on a fresh machine with Kafka installed and properly configured. The broker would start with a `kafka-server-start` or `systemctl start confluent-kafka` command.
```
docker-compose up -d broker4
```

### Manually Reassign Partitions (with throttle)

Check where partitions are placed. Notice Broker 4 doesn't serve any of this traffic.
```
docker compose exec broker1 \
    kafka-topics \
    --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
    --describe --topic test
```

Create a json file to specify topics you want to rebalance. Notice only one topic is specified (what if you have 1000 topics?).
```
echo '{"topics": [{"topic": "test"}],"version":1}' > reassignment-files/topics-to-move.json
```

Generate a reassignment plan. Notice you don't know the load is evenly balanced amongst partitions, so you would need to measure that to make smarter decisions about partition placement.
```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --topics-to-move-json-file /tmp/reassignment-files/topics-to-move.json \
        --broker-list "1,2,3,4" \
        --generate \
    > ./reassignment-files/reassignment.json
```

The reassignment plan `./reassignment-files/reassignment.json` should look like this:
```
Current partition replica assignment
{"version":1,"partitions":[{"topic":"test","partition":0,"replicas":[1,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":1,"replicas":[2,3,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[3,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":3,"replicas":[1,3,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":4,"replicas":[2,1,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":5,"replicas":[3,2,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":6,"replicas":[1,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":7,"replicas":[2,3,1],"log_dirs":["any","any","any"]}]}

Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"test","partition":0,"replicas":[1,4,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":1,"replicas":[2,1,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[3,2,4],"log_dirs":["any","any","any"]},{"topic":"test","partition":3,"replicas":[4,3,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":4,"replicas":[1,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":5,"replicas":[2,3,4],"log_dirs":["any","any","any"]},{"topic":"test","partition":6,"replicas":[3,4,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":7,"replicas":[4,1,2],"log_dirs":["any","any","any"]}]}
```

Open the file  `./reassignment-files/reassignment.json` and edit it so it only includes the “Proposed partition reassignment configuration" in JSON (i.e., just the last non-empty line).

The resulting file should look like this:
```
{"version":1,"partitions":[{"topic":"test","partition":0,"replicas":[1,4,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":1,"replicas":[2,1,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[3,2,4],"log_dirs":["any","any","any"]},{"topic":"test","partition":3,"replicas":[4,3,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":4,"replicas":[1,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":5,"replicas":[2,3,4],"log_dirs":["any","any","any"]},{"topic":"test","partition":6,"replicas":[3,4,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":7,"replicas":[4,1,2],"log_dirs":["any","any","any"]}]}
```

Execute the reassignment plan with throttling set to 1MBps.

```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --reassignment-json-file /tmp/reassignment-files/reassignment.json \
        --execute \
        --throttle 1000000
```

Verify the throttle has been set.
```
docker compose exec broker1 \
    kafka-configs \
        --describe \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --entity-type brokers
```

Check the progress of the migration until it is complete and the throttles have been removed.

```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --reassignment-json-file /tmp/reassignment-files/reassignment.json \
        --verify
```

Verify that the partitions have been migrated. Note broker 4 is now the leader for some of the partitions.

```
docker compose exec broker1 \
    kafka-topics \
    --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
    --describe --topic test
```

## Steps to Decommission a broker
### Pick a broker to decommission (don't kill the controller!)

Check which broker is the Controller. You don't want to kill the controller because it may result in degraded cluster performance or even downtime while re-elections occur. This effect is pronounced when there are more partitions in the cluster. We will assume broker 1 is the controller and proceed to decommission broker 4.

```
docker compose exec zookeeper \
    zookeeper-shell localhost get /controller
```

### Reassign Partitions and Replicas Off of the Broker

We are decommissioning broker 4. We essentially do the same process as before, but in reverse. First, we use the `kafka-reassign-partitions` tool to generate a reassignment plan that moves partition leaders and replicas off of broker 4, this time generating the file `decommission.json`. Note we are using the same `topics-to-move.json` file from before as input, which only moves the `test` topic. In general we would first have to use the `kafka-topics` command to see all the topics that have partition leaders or replicas on broker 4 and include those in `topics-to-move.json`.

```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --topics-to-move-json-file /tmp/reassignment-files/topics-to-move.json \
        --broker-list "1,2,3" \  
        --generate \
    > ./reassignment-files/decommission.json
```

Again edit `./reassignment-files/decommission.json` so only the proposed partition reassignment json remains (e.g. only the last non-empty line). It should resemble:

```
{"version":1,"partitions":[{"topic":"test","partition":0,"replicas":[3,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":1,"replicas":[1,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[2,3,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":3,"replicas":[3,2,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":4,"replicas":[1,3,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":5,"replicas":[2,1,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":6,"replicas":[3,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":7,"replicas":[1,2,3],"log_dirs":["any","any","any"]}]}
```
Notice there are no replicas planned to be placed on broker 4.

Execute the reassignment with a throttle of 1MBps.
```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --reassignment-json-file /tmp/reassignment-files/decommission.json \
        --execute \
        --throttle 1000000
```

Check that the migration was successful and the throttles were removed.

```
docker compose exec broker1 \
    kafka-reassign-partitions \
        --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
        --reassignment-json-file /tmp/reassignment-files/decommission.json \
        --verify
```

Verify that broker 4 no longer holds replicas for the topic.

```
docker compose exec broker1 \
    kafka-topics \
    --bootstrap-server broker1:29091,broker2:29092,broker3:29093 \
    --describe --topic test
```

### Gracefully kill broker

Usually this would be a `kafka-server-stop` or `systemctl stop confluent-kafka` command.
```
docker compose rm --stop --force broker4
```
