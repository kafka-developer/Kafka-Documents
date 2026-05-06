# How You Create the Cluster

## 1. Plan Capacity First

Before installing anything, ask:  I would not randomly create brokers. I’d start with throughput, retention, replication factor, disk, network, and consumer parallelism requirements.”
## 1. Where do you get these numbers from?

In real-world Kafka implementations, customers usually do not provide exact Kafka metrics like partition count or broker sizing upfront. They typically provide business-level requirements such as expected transaction volume, retention expectations, number of applications, downstream consumers, availability expectations, and whether the workload is production-critical.

For example, they may say this is for payments, fraud detection, or replacing an existing MQ platform and it must be highly available with minimal downtime.

From that business context, we work with application teams, MQ teams, database teams, and infrastructure teams to estimate throughput, message volume, peak load, retention, and consumer patterns.

Based on those estimates, we decide:

-   number of brokers
    
-   number of partitions
    
-   replication factor
    
-   storage sizing
    
-   disk and network capacity
    
-   HA and DR strategy
    

So customers provide workload expectations, and we convert that into Kafka infrastructure sizing.

----------

## 2. What is the first step of Capacity Planning?

The first step of Kafka capacity planning is requirement discovery and workload estimation.

Before deciding brokers or ZooKeeper/KRaft nodes, we first understand:

-   expected producer throughput
    
-   expected consumer throughput
    
-   retention period
    
-   message size
    
-   number of topics
    
-   number of consumers or consumer groups
    
-   availability requirements
    
-   growth forecast for future scaling
    

In many cases, customers do not know exact numbers, so we start with assumptions based on business volume and validate those assumptions through load testing or pilot traffic.

Once workload is understood, we calculate:

-   storage requirements
    
-   partition count
    
-   network requirements
    
-   broker count
    
-   replication strategy
    
-   failover planning
    

Broker sizing comes after workload estimation, not before.

----------

## 3. How do we decide the number of Brokers?

Broker count is decided based on partition requirements, storage requirements, throughput requirements, replication factor, high availability, and future growth.

----------

### a) Partition Calculation

First, we estimate the required partitions using producer and consumer throughput.

We use:

```text
Required Partitions = max (Producer Throughput / Partition Throughput,
                           Consumer Throughput / Partition Throughput)

```

For example:

If customer requirement is 1 TB per day, we first convert that into throughput.

Average throughput for 1 TB/day is roughly 12 MB/sec, but in production we usually size for peak traffic, which may be 3x to 5x higher depending on the workload.

So we may design for around 50 MB/sec peak throughput.

If one partition can safely handle around 10 MB/sec, then:

```text
50 MB/sec ÷ 10 MB/sec = 5 partitions

```

So we would need at least 5 partitions for that workload.

Partition sizing is also influenced by consumer parallelism and future scaling.

----------

### b) Broker Calculation

After partition sizing, we calculate broker count.

Broker count depends on:

-   total partition replicas
    
-   storage requirement
    
-   disk capacity
    
-   network throughput
    
-   replication factor
    
-   fault tolerance requirements
    

For example:

If customer has:

-   100 topics
    
-   5 partitions per topic
    
-   replication factor = 3
    

Then:

```text
Total replicas = Topics × Partitions × Replication Factor
               = 100 × 5 × 3
               = 1500 replicas

```

These replicas are distributed across brokers.

If we start with a 3-broker cluster:

```text
1500 ÷ 3 = 500 replicas per broker

```

This is manageable.

For production environments, especially in banking, we usually prefer at least 3 brokers for RF=3, but often 5 or more brokers are better for maintenance, scaling, and fault tolerance.

The final broker count is always driven by the highest requirement among storage, throughput, HA, and future growth.

----------

## Strong Interview Closing Line

In Kafka capacity planning, I do not size brokers first. I first understand the business workload, convert that into throughput and storage estimates, calculate partition requirements, and then decide broker count based on replication, HA, and future growth. The goal is not just to make Kafka work, but to ensure it remains stable, scalable, and fault tolerant in production.

## 2. Install Kafka / Confluent Platform On-Prem
 Each broker runs on a Linux VM or physical server.
Typical components:

> confluent-server / kafka broker
schema-registry
kafka-connect
ksqldb-server
control-center
prometheus exporter / JMX exporter

## Producer throughput
```bash 

Main properties:

batch.size=65536    #65kb
linger.ms=5   # How long the producer waits before sending messages to Kafka.
compression.type=lz4  #message compression, Snappy, lz4, gzip & zstd(best overall)
acks=all   # Acknowledgment level for writes.
enable.idempotence=true  # Prevents duplicate message writes caused by retries. Required for exactly-once style safety at producer level.
max.in.flight.requests.per.connection=5  # Maximum number of unacknowledged requests allowed on a single connection. With idempotence=true, Kafka allows up to 5 safely. Higher throughput without breaking ordering guarantees.
buffer.memory=67108864  # Total memory (in bytes) available for the producer to buffer records  waiting to be sent to Kafka. If buffer fills up, producer blocks or throws exception depending on timeout. Larger buffer helps during traffic spikes.
```
Explain:
>For producer throughput, I increase batching using `batch.size` and small `linger.ms`, enable compression like `lz4` or `snappy`, and ensure enough producer buffer memory. For durability, I usually keep `acks=all` with idempotence enabled.

## Consumer throughput

```bash
Main properties:

fetch.min.bytes=1048576    # Minimum amount of data (in bytes) the broker should collect before responding to a consumer fetch request.
fetch.max.wait.ms=500   # Maximum time (in milliseconds) the broker waits to satisfy fetch.min.bytes before sending data anyway.
max.partition.fetch.bytes=1048576   # Maximum amount of data (in bytes) fetched from a single partition per request.
max.poll.records=500  # Maximum number of records returned in a single poll() call.
receive.buffer.bytes=65536   # Size of the TCP receive buffer (SO_RCVBUF)used by the consumer socket.
```
Explain:
> For consumer throughput, I increase fetch size so the consumer pulls more data per request. I tune `fetch.min.bytes`, `fetch.max.wait.ms`, `max.partition.fetch.bytes`, and `max.poll.records` based on processing speed.

## Producer latency
Main properties:
```bash
linger.ms=0
batch.size=16384
compression.type=none
acks=1
```
But for production banking, be careful with `acks=1`.

> For low latency, I reduce `linger.ms`, avoid oversized batches, keep producers close to brokers network-wise, and tune compression carefully. I do not blindly reduce `acks` in critical workloads because that can reduce durability.

## Consumer latency

Main properties:
```bash
fetch.min.bytes=1
fetch.max.wait.ms=50
max.poll.records=100
```
Explain:
>For low-latency consumers, I reduce `fetch.min.bytes` and `fetch.max.wait.ms` so consumers do not wait too long to receive data. I also keep `max.poll.records` reasonable so each poll finishes quickly.

## Longer client connection / avoid consumer timeout
Main properties:
```bash
session.timeout.ms=45000
heartbeat.interval.ms=15000
max.poll.interval.ms=600000
request.timeout.ms=120000
connections.max.idle.ms=900000
```
Explain:
> To avoid consumer timeout, I tune `session.timeout.ms`, `heartbeat.interval.ms`, and especially `max.poll.interval.ms`. If message processing takes longer, `max.poll.interval.ms` must be increased so Kafka does not consider the consumer dead and trigger a rebalance.

Important rule:
> heartbeat.interval.ms should be around 1/3 of session.timeout.ms
Example:
```bash
session.timeout.ms=45000
heartbeat.interval.ms=15000
```





<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MTUzNzA5MDcsMzM0MTUzODQ3XX0=
-->