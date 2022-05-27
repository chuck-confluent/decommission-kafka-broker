# decommission-kafka-broker
scratch repo to test manual decommission of a kafka broker using only apache kafka tools. Then, we would compare to cluster expansion/shrink in the Confluent Cloud Console.


## Cheat sheet

### List topics
```
docker compose exec broker1 \
    kafka-topics --bootstrap-server broker1:9091 \
    --list
```

## Steps to decommission a broker

### Pick a broker to decommission (don't kill the controller!)

### Find out the partitions for which that broker is the leader

### Reassign Partitions (sucks for any new partitions getting created -- they may be unavailable)

### Gracefully kill broker
