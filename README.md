# decommission-kafka-broker
scratch repo to test manual decommission of a kafka broker


## Cheat sheet

```
docker run --rm -it --network=decommission-kafka-broker_default \
    edenhill/kcat:1.7.1 -b broker2:9092 -L
```