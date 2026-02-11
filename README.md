# kafka_homework

## HA Implementation: Multi-Broker Replication

We deployed a 3-broker Kafka cluster (KRaft mode) using Docker Compose.

### Topic Configuration
- Topic: `ha-topic`
- Partitions: 1
- Replication Factor: 3

Initial state:
Replicas: 3,1,2
Isr: 3,1,2

All replicas were in sync.

### Failure Simulation

We stopped broker `kafka2`:

docker stop kafka2

Result:
- Producer could still successfully send messages.
- Consumer could still read all messages.
- ISR changed from `3,1,2` to `3,1`.

This demonstrates:
- Leader election remained stable.
- The cluster removed the failed broker from ISR.
- Service availability was maintained.

Therefore, Kafka High Availability is successfully validated.
