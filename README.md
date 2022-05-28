Work in progress

# Kafka Elasticity
scratch repo to test manual scaling of a kafka cluster using only apache kafka tools. Then, we would compare to cluster expansion/shrink in the Confluent Cloud Console.


## Cheat sheet

### Start Cluster

```
docker compose up -d zookeeper broker1 broker2 broker3
```

### List topics
```
docker compose exec broker1 \
    kafka-topics --bootstrap-server broker1:9091 \
    --list
```

## Steps to Decommission a broker

### Pick a broker to decommission (don't kill the controller!)

### Find out the partitions for which that broker is the leader

### Manually Reassign Partitions (with throttle)

### Remove Throttle

### Gracefully kill broker

## Steps to Add a Broker

### Start Broker

### Manually Reassign Partitions (with throttle)

### Remove throttle
