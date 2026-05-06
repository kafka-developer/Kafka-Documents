# Kafka Interview Revision — Simple Answers With Commands

## Zookeeper

### 1. What are Zookeeper “other 2 ports” different from 2181 and what are they used for?

**Answer —** Zookeeper usually uses **3 important ports**:

```text
2181 = Client port
2888 = Follower-to-leader communication port
3888 = Leader election port
```

Example:

```properties
clientPort=2181

server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

**2181** is used by Kafka brokers to connect to Zookeeper.
**2888** is used between Zookeeper servers after leader election.
**3888** is used by Zookeeper nodes to elect a leader.

---

### 2. What is the use of Zookeeper in Kafka?

**Answer —** In older Kafka architecture, Zookeeper manages Kafka cluster metadata.

It tracks:

```text
Broker registration
Controller election
Topic metadata
Partition leadership
ISR information
Cluster membership
ACLs in older setups
```

Kafka brokers depend on Zookeeper to know who is alive, who is controller, and which broker owns which partition.

---

### 3. Which layer of Broker OSI model does Zookeeper connect to?

**Answer —** Zookeeper communication uses **TCP**, so it works at **OSI Layer 4 — Transport Layer**.

Kafka/Zookeeper application logic is Layer 7, but the actual network connection is TCP Layer 4.

```text
Broker ↔ Zookeeper = TCP connection
Port = 2181
OSI Layer = Transport Layer
```

---

### 4. What are zNodes? Explain Zookeeper leader election.

**Answer —** zNodes are like files/directories inside Zookeeper’s internal metadata tree.

Example:

```text
/brokers/ids/1
/brokers/topics/orders
/controller
/admin/delete_topics
```

Kafka stores cluster metadata in these zNodes.

Zookeeper leader election works like this:

```text
1. Multiple Zookeeper nodes start.
2. Each node votes.
3. The node with the best/latest transaction ID usually wins.
4. One node becomes Leader.
5. Others become Followers.
6. If Leader dies, another election happens.
```

Zookeeper should normally run with an odd number of nodes:

```text
3 nodes = can tolerate 1 failure
5 nodes = can tolerate 2 failures
```

---

# Kraft

### 5. Which is better: Kraft or Zookeeper and why?

**Answer —** **KRaft is better for modern Kafka** because Kafka no longer needs external Zookeeper.

KRaft benefits:

```text
No Zookeeper dependency
Faster metadata handling
Better scalability
Simpler architecture
Kafka manages its own metadata
Controller quorum replaces Zookeeper
```

Old model:

```text
Kafka Broker → Zookeeper
```

New KRaft model:

```text
Kafka Broker → Kafka Controller Quorum
```

For new Kafka deployments, KRaft is preferred.

---

### 6. What are KRaft ports?

**Answer —** KRaft usually has:

```text
9092 = Broker listener/client traffic
9093 = Controller listener/KRaft quorum traffic
```

Example:

```properties
process.roles=broker,controller
node.id=1

listeners=PLAINTEXT://broker1:9092,CONTROLLER://broker1:9093
advertised.listeners=PLAINTEXT://broker1:9092

controller.listener.names=CONTROLLER
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
```

---

### 7. Which layer of Broker OSI model does KRaft connect to?

**Answer —** KRaft controller communication also uses **TCP**, so it is **OSI Layer 4 — Transport Layer**.

```text
Broker ↔ Controller quorum = TCP
Usually port = 9093
```

---

### 8. What happens when there is only one KRaft controller instead of a quorum?

**Answer —** It works, but it has **no high availability**.

With one controller:

```text
Controller alive = cluster works
Controller down = metadata operations fail
```

Problems:

```text
No fault tolerance
No majority voting
No controller failover
Not good for production
```

Production should use **3 or 5 controllers**.

---

# Kafka Broker

### 9. What is IBP, Inter-Broker Protocol, and why is it important?

**Answer —** **Inter-Broker Protocol** controls the Kafka protocol version used between brokers.

It matters during upgrades.

Example:

```properties
inter.broker.protocol.version=3.6
```

Why important:

```text
Allows rolling upgrades
Prevents new broker from using features old brokers cannot understand
Keeps mixed-version clusters stable during upgrade
```

---

### 10. What is log format version and why is it important?

**Answer —** Log format version controls the message format written to Kafka logs on disk.

Example:

```properties
log.message.format.version=3.6
```

Why important:

```text
Controls physical message format
Important during upgrades/downgrades
Older consumers or brokers may not understand newer format
```

In modern Kafka, this is less commonly changed manually, but still important during compatibility planning.

---

### 11. What is unclean leader election and when is it needed?

**Answer —** Unclean leader election allows Kafka to elect a replica that is **not fully in-sync** as leader.

Config:

```properties
unclean.leader.election.enable=false
```

If set to `true`, Kafka may choose an out-of-sync replica.

Benefit:

```text
Partition becomes available
```

Risk:

```text
Data loss
```

Use only when:

```text
Availability is more important than data correctness
Emergency recovery situation
Non-critical data
```

For banking/financial systems, usually:

```properties
unclean.leader.election.enable=false
```

---

### 12. What is the difference between ACL and RBAC?

**Answer —**

| Item        | ACL                          | RBAC                          |
| ----------- | ---------------------------- | ----------------------------- |
| Full form   | Access Control List          | Role-Based Access Control     |
| Level       | Resource permission          | Role permission               |
| Example     | User A can READ topic orders | User A has DeveloperRead role |
| Common in   | Apache Kafka                 | Confluent Platform            |
| Granularity | Very detailed                | Easier centralized management |

ACL example:

```bash
kafka-acls --bootstrap-server broker:9092 \
  --add \
  --allow-principal User:app1 \
  --operation Read \
  --topic orders
```

RBAC example concept:

```text
Assign role DeveloperRead to user/app on Kafka cluster/topic
```

---

### 13. How do you put all messages in a specified partition?

**Answer —** Use a **custom partitioner** or explicitly send records to a partition in producer code.

Java example:

```java
ProducerRecord<String, String> record =
    new ProducerRecord<>("orders", 2, "key1", "message-value");

producer.send(record);
```

This sends the message to partition `2`.

Command-line testing:

```bash
kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --property parse.key=true \
  --property key.separator=:
```

Kafka will hash the key and send same key to same partition.

---

### 14. Where does Broker perform better: bare metal vs virtual machine?

**Answer —** Kafka usually performs better on **bare metal** because it gets direct access to CPU, memory, disk, and network.

Bare metal benefits:

```text
Lower latency
Better disk I/O
Better network throughput
Less virtualization overhead
More predictable performance
```

Virtual machines are still common, but need proper sizing.

For heavy on-prem Kafka:

```text
Bare metal is preferred
VMs are acceptable if storage/network are strong
```

---

### 15. Which disk is better suited for Broker: SSD vs HDD?

**Answer —** **SSD is better** for Kafka brokers when performance matters.

SSD benefits:

```text
Faster reads/writes
Better random I/O
Faster recovery
Better consumer catch-up
Better broker restart performance
```

HDD can work for low-cost, high-retention workloads, but SSD is preferred for production performance.

---

### 16. If disk is getting full frequently, how would you address this?

**Answer —** Check retention, topic size, replication factor, and consumer lag.

Useful commands:

```bash
kafka-topics --bootstrap-server localhost:9092 --describe
```

Check topic configs:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --describe
```

Fix options:

```text
Reduce retention.ms
Reduce retention.bytes
Increase disk capacity
Add brokers
Reassign partitions
Delete unused topics
Compress producer messages
Check replication factor
Check large messages
Check consumer lag
```

Example reduce retention:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.ms=604800000
```

That sets retention to 7 days.

---

### 17. What are the mandatory properties to create a topic?

**Answer —** Minimum required:

```text
Topic name
Partitions
Replication factor
Bootstrap server
```

Command:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 6 \
  --replication-factor 3
```

Important optional configs:

```text
retention.ms
retention.bytes
cleanup.policy
min.insync.replicas
segment.bytes
```

Example:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000
```

---

# Kafka Producer

### 18. What are the producer settings to maximize throughput?

**Answer —** Throughput means sending more data per second.

Important settings:

```properties
batch.size=65536
linger.ms=5
compression.type=lz4
acks=all
buffer.memory=67108864
max.in.flight.requests.per.connection=5
enable.idempotence=true
```

Explanation:

```text
batch.size = larger batch
linger.ms = wait a little to collect more records
compression.type = reduce network size
buffer.memory = more producer memory
acks=all = safer but slightly slower
```

---

### 19. What are producer settings to minimize latency?

**Answer —** Latency means send messages as fast as possible.

Settings:

```properties
linger.ms=0
batch.size=16384
compression.type=none
acks=1
max.in.flight.requests.per.connection=1
```

Tradeoff:

```text
Lower latency usually means lower throughput
acks=1 is faster but less durable than acks=all
```

For critical systems, do not reduce durability blindly.

---

### 20. What property helps in tackling message ordering?

**Answer —**

```properties
max.in.flight.requests.per.connection=1
```

This gives strongest ordering because only one request is in flight.

With idempotence enabled, Kafka allows safe ordering with:

```properties
enable.idempotence=true
max.in.flight.requests.per.connection=5
```

Also, same key must go to same partition.

```text
Ordering is guaranteed only within a partition.
```

---

### 21. How do we tackle message duplication?

**Answer —** Use idempotent producer.

```properties
enable.idempotence=true
acks=all
retries=2147483647
max.in.flight.requests.per.connection=5
```

Idempotence prevents duplicate writes caused by producer retries.

For full exactly-once processing, use transactions:

```properties
transactional.id=my-producer-1
enable.idempotence=true
```

---

# Kafka Consumer

### 22. What is consumer rebalance? How does it work?

**Answer —** Consumer rebalance happens when Kafka redistributes partitions among consumers in the same consumer group.

It happens when:

```text
New consumer joins
Existing consumer dies
Consumer leaves
New partitions are added
Consumer takes too long to poll
```

Example:

```text
Topic has 6 partitions
Consumer group has 2 consumers
Each gets 3 partitions

If 1 more consumer joins:
Kafka rebalances
Now each may get 2 partitions
```

---

### 23. How does Kafka determine if a consumer is dead or does not exist?

**Answer —** Kafka uses heartbeats.

Important configs:

```properties
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=300000
```

If broker does not receive heartbeat within `session.timeout.ms`, consumer is considered dead.

If consumer does not call `poll()` within `max.poll.interval.ms`, Kafka removes it from the group.

---

### 24. What is subscribe() vs assign()?

**Answer —**

| Method        | Meaning                                                       |
| ------------- | ------------------------------------------------------------- |
| `subscribe()` | Kafka automatically assigns partitions through consumer group |
| `assign()`    | You manually assign specific partitions                       |

Example `subscribe()`:

```java
consumer.subscribe(Arrays.asList("orders"));
```

Use when:

```text
You want Kafka group management
You want automatic rebalancing
```

Example `assign()`:

```java
TopicPartition partition = new TopicPartition("orders", 0);
consumer.assign(Arrays.asList(partition));
```

Use when:

```text
You want full manual control
No group rebalance needed
```

---

### 25. What is exactly-once semantics? How is it different from idempotence?

**Answer —** Idempotence prevents duplicate writes from a producer.

Exactly-once semantics means:

```text
Consume message
Process message
Produce output
Commit offset
```

All happen safely as one transaction.

Difference:

| Concept                | Meaning                                                         |
| ---------------------- | --------------------------------------------------------------- |
| Idempotence            | Prevents duplicate producer writes                              |
| Exactly-once semantics | Prevents duplicate processing/output in read-process-write flow |

Producer idempotence:

```properties
enable.idempotence=true
```

Transaction:

```properties
transactional.id=my-app-1
```

---

### 26. When does consumer lose data?

**Answer —** Consumer can lose data when offset is committed before processing is complete.

Bad flow:

```text
Poll message
Commit offset
Processing fails
Message is not processed again
Data lost
```

Risky config:

```properties
enable.auto.commit=true
```

Safer approach:

```properties
enable.auto.commit=false
```

Then manually commit after processing.

---

### 27. When does a consumer get duplication?

**Answer —** Duplication happens when message is processed but offset is not committed.

Flow:

```text
Poll message
Process message successfully
Consumer crashes before commit
Consumer restarts
Same message is read again
```

This is common in **at-least-once** processing.

---

### 28. What are consumer settings to maximize throughput?

**Answer —**

```properties
fetch.min.bytes=1048576
fetch.max.wait.ms=500
max.partition.fetch.bytes=1048576
max.poll.records=500
receive.buffer.bytes=65536
```

Explanation:

```text
fetch.min.bytes = broker waits until enough data is available
fetch.max.wait.ms = max wait time for fetch
max.poll.records = more records per poll
max.partition.fetch.bytes = larger fetch per partition
```

---

### 29. What are consumer settings to minimize latency?

**Answer —**

```properties
fetch.min.bytes=1
fetch.max.wait.ms=50
max.poll.records=100
```

Explanation:

```text
Consumer fetches quickly
Does not wait for large batches
Processes smaller batches faster
```

Tradeoff:

```text
Lower latency usually reduces throughput
```

---

### 30. What property helps in tackling message ordering?

**Answer —** Ordering is handled by **partitioning**.

Key rule:

```text
Same key → same partition → ordered processing
```

Producer side:

```text
Use message key
```

Consumer side:

```text
One partition is consumed by only one consumer in a group at a time
```

Kafka guarantees order only inside a partition, not across the whole topic.

---

# Kafka Security

### 31. What are 5 ACL pillars?

**Answer —** Kafka ACLs are built around:

```text
Principal
Operation
Resource Type
Resource Name
Permission Type
Host
```

Example:

```bash
kafka-acls --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:app1 \
  --operation Read \
  --topic orders \
  --group app1-group
```

Common operations:

```text
Read
Write
Create
Delete
Alter
Describe
ClusterAction
```

---

### 32. What are listeners and advertised.listeners?

**Answer —**

`listeners` means where broker binds/listens.

```properties
listeners=PLAINTEXT://0.0.0.0:9092
```

`advertised.listeners` means what address clients should use to connect.

```properties
advertised.listeners=PLAINTEXT://broker1.company.com:9092
```

Simple difference:

```text
listeners = broker opens this port
advertised.listeners = broker tells clients to use this address
```

Wrong advertised listener is a very common Kafka connection issue.

---

### 33. What is the use of JAAS file?

**Answer —** JAAS file stores authentication configuration for SASL.

Example for SCRAM:

```properties
KafkaServer {
 org.apache.kafka.common.security.scram.ScramLoginModule required
 username="broker"
 password="broker-secret";
};
```

Used for:

```text
SASL/PLAIN
SASL/SCRAM
Kerberos
Broker authentication
Client authentication
```

---

### 34. What are the high-level steps to set up SSL in Broker?

**Answer —**

Steps:

```text
1. Create Certificate Authority
2. Create broker keystore
3. Create broker certificate signing request
4. Sign broker certificate
5. Import CA certificate into truststore
6. Import signed broker certificate into keystore
7. Configure broker SSL properties
8. Configure client truststore/keystore
9. Restart broker
10. Test connection
```

Broker config:

```properties
listeners=SSL://broker1:9093
advertised.listeners=SSL://broker1:9093
security.inter.broker.protocol=SSL

ssl.keystore.location=/etc/kafka/secrets/broker.keystore.jks
ssl.keystore.password=password
ssl.key.password=password

ssl.truststore.location=/etc/kafka/secrets/broker.truststore.jks
ssl.truststore.password=password

ssl.client.auth=required
```

---

### 35. What is 2-way and 3-way handshake?

**Answer —** In Kafka security, people usually mean SSL/TLS handshake.

**One-way SSL:**

```text
Client verifies broker certificate
Broker does not verify client certificate
```

**Two-way SSL / mutual TLS:**

```text
Client verifies broker certificate
Broker verifies client certificate
```

**TCP 3-way handshake** is different:

```text
SYN
SYN-ACK
ACK
```

That happens before TCP connection is established.

---

### 36. What is session.timeout? How does broker handle client connections?

**Answer —** `session.timeout.ms` controls how long broker waits before considering consumer dead.

Example:

```properties
session.timeout.ms=45000
heartbeat.interval.ms=3000
```

Broker expects heartbeats. If heartbeats stop, broker removes consumer from group and triggers rebalance.

---

### 37. What is a quota?

**Answer —** Quota limits how much resource a client can use.

Kafka quotas can limit:

```text
Producer bandwidth
Consumer bandwidth
Request rate
Controller mutation rate
```

Example:

```bash
kafka-configs --bootstrap-server localhost:9092 \
  --alter \
  --add-config 'producer_byte_rate=1048576,consumer_byte_rate=2097152' \
  --entity-type clients \
  --entity-name app1
```

---

### 38. How can you increase client connection duration in Kafka?

**Answer —** Tune timeout and idle configs.

Useful properties:

```properties
connections.max.idle.ms=600000
request.timeout.ms=60000
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=600000
```

For consumers, most important:

```properties
session.timeout.ms
heartbeat.interval.ms
max.poll.interval.ms
```

If processing takes long, increase:

```properties
max.poll.interval.ms
```

---

# Kafka Schema Registry

### 39. What is schema compatibility? What are different compatibility types? How do we make schema compatible?

**Answer —** Schema compatibility controls whether a new schema version can safely work with old producers/consumers.

Types:

| Type                | Meaning                                          |
| ------------------- | ------------------------------------------------ |
| BACKWARD            | New consumer can read old data                   |
| FORWARD             | Old consumer can read new data                   |
| FULL                | Both backward and forward                        |
| NONE                | No compatibility check                           |
| BACKWARD_TRANSITIVE | New schema compatible with all previous versions |
| FORWARD_TRANSITIVE  | Old schemas compatible with all future versions  |
| FULL_TRANSITIVE     | Both ways across all versions                    |

Safe change example:

```text
Add new optional field with default value
```

Unsafe changes:

```text
Rename field
Remove required field
Change datatype
```

Avro safe example:

```json
{
  "name": "customer_email",
  "type": ["null", "string"],
  "default": null
}
```

---

### 40. What is the high-level step-by-step process of introducing a schema to a topic for the first time?

**Answer —**

```text
1. Decide data format: Avro, Protobuf, or JSON Schema
2. Create schema
3. Choose subject naming strategy
4. Set compatibility mode
5. Register schema in Schema Registry
6. Configure producer with Schema Registry URL
7. Producer sends serialized message
8. Consumer fetches schema ID and deserializes message
```

Register schema:

```bash
curl -X POST http://schema-registry:8081/subjects/orders-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"}]}"
  }'
```

---

### 41. What is schema registration?

**Answer —** Schema registration means storing a schema in Schema Registry under a subject.

Example subject:

```text
orders-value
```

Schema Registry assigns a schema ID.

Producer message contains:

```text
Magic byte
Schema ID
Serialized payload
```

Consumer uses schema ID to fetch schema and deserialize data.

---

### 42. How do you check schema compatibility?

**Answer —** Use Schema Registry compatibility endpoint.

Command:

```bash
curl -X POST http://schema-registry:8081/compatibility/subjects/orders-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":[\"null\",\"double\"],\"default\":null}]}"
  }'
```

Response:

```json
{"is_compatible":true}
```

Check current compatibility:

```bash
curl http://schema-registry:8081/config/orders-value
```

Set compatibility:

```bash
curl -X PUT http://schema-registry:8081/config/orders-value \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"compatibility":"BACKWARD"}'
```

---

# Kafka Connect

### 43. How do you add a worker node to an existing Connect cluster?

**Answer —** Add another worker with the same Connect cluster configs.

Important configs must match:

```properties
group.id=connect-cluster
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
bootstrap.servers=broker1:9092,broker2:9092
```

Then start the new worker.

Kafka Connect automatically joins the group and rebalances connector tasks.

---

### 44. What is the step-by-step high-level process of installing a new connector?

**Answer —**

```text
1. Download connector plugin
2. Place plugin under plugin.path
3. Restart Connect workers
4. Verify plugin is available
5. Create connector JSON config
6. Submit connector using REST API
7. Check connector status
8. Monitor tasks and logs
```

Config:

```properties
plugin.path=/usr/share/java,/opt/connectors
```

Check plugins:

```bash
curl http://connect:8083/connector-plugins
```

Create connector:

```bash
curl -X POST http://connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector.json
```

Check status:

```bash
curl http://connect:8083/connectors/my-connector/status
```

---

### 45. How do you improve connector performance?

**Answer —**

Options:

```text
Increase tasks.max
Increase topic partitions
Tune producer/consumer settings
Tune batch size
Tune external system performance
Add more Connect workers
Use SMTs carefully
Avoid slow transformations
Check network and database bottlenecks
```

Example:

```json
{
  "tasks.max": "6"
}
```

But `tasks.max` only helps if:

```text
Topic has enough partitions
External system supports parallelism
Connector supports multiple tasks
```

---

### 46. What are the mandatory configurations for Connect JSON file to create a connector?

**Answer —** Minimum:

```json
{
  "name": "my-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "orders"
  }
}
```

Common mandatory fields:

```text
name
connector.class
tasks.max
topics or topics.regex
Connector-specific connection properties
Key/value converter configs if not inherited
```

For JDBC Sink, also need:

```json
{
  "connection.url": "jdbc:postgresql://db:5432/app",
  "connection.user": "user",
  "connection.password": "password"
}
```

---

### 47. How does Kafka Connect ensure that workers know about each other’s configurations?

**Answer —** Connect workers use internal Kafka topics.

Important topics:

```text
config.storage.topic
offset.storage.topic
status.storage.topic
```

Example:

```properties
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
```

Workers also share same:

```properties
group.id=connect-cluster
```

That allows them to coordinate and rebalance tasks.

---

### 48. Where does Connect store its committed offset?

**Answer —** Kafka Connect stores offsets in the internal offset topic.

Config:

```properties
offset.storage.topic=connect-offsets
```

For standalone mode, offsets can be stored in a file:

```properties
offset.storage.file.filename=/tmp/connect.offsets
```

Distributed mode uses Kafka topic.

---

# Kafka On-Prem Specific

### 49. Which one will you choose: bare metal vs virtual machine?

**Answer —** For high-throughput production Kafka, I would prefer **bare metal**.

Reason:

```text
Better disk I/O
Better network performance
Lower latency
Predictable performance
Less noisy-neighbor risk
```

But VMs are acceptable if:

```text
Storage is dedicated
Network is strong
CPU/memory are reserved
Monitoring is strong
```

Interview answer:

```text
For critical on-prem Kafka, I prefer bare metal. But if the company standard is VM, I make sure the VM has dedicated resources, fast disks, proper network bandwidth, and no overcommitment.
```

---

### 50. How is on-prem capacity planning different from Confluent Cloud capacity planning?

**Answer —** In Confluent Cloud, Confluent manages brokers, storage, scaling, patching, and infrastructure.

In on-prem, we manage everything.

On-prem capacity planning includes:

```text
Broker count
Disk size
CPU
Memory
Network
Rack awareness
Replication factor
Retention
Partition count
Controller/Zookeeper sizing
Disaster recovery
Monitoring
Patch/upgrade planning
```

Confluent Cloud planning focuses more on:

```text
Throughput
Storage
Partitions
Connectors
Networking
Cost
Environment/cluster type
```

---

### 51. Which one do you prefer: SSD or HDD?

**Answer —** I prefer **SSD** for Kafka brokers.

Reason:

```text
Faster recovery
Better consumer catch-up
Better disk I/O
Lower latency
Better performance during rebalancing and replication
```

HDD can be used for cheap long-term retention, but SSD is better for active Kafka workloads.

---

### 52. What is Zero Copy transfer?

**Answer —** Zero Copy means Kafka sends data from disk to network without copying data multiple times through application memory.

Normal flow:

```text
Disk → Kernel → Application → Kernel → Network
```

Kafka Zero Copy flow:

```text
Disk → Kernel → Network
```

Kafka uses operating system feature like `sendfile()`.

Benefit:

```text
Less CPU usage
Faster transfer
Higher throughput
```

This is one big reason Kafka is very fast.

---

### 53. How does Kafka client connect to the server? At what OSI layer?

**Answer —** Kafka clients connect to brokers using **TCP**.

```text
Producer/Consumer/Admin Client → Broker
Protocol = Kafka protocol over TCP
OSI Layer = Layer 4 Transport
Application protocol = Layer 7
```

Connection flow:

```text
1. Client connects to bootstrap server
2. Broker returns cluster metadata
3. Client learns broker and partition leaders
4. Client connects directly to leader brokers
5. Producer sends data / consumer fetches data
```

Example client config:

```properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
security.protocol=PLAINTEXT
```

For SSL:

```properties
security.protocol=SSL
ssl.truststore.location=/path/client.truststore.jks
ssl.truststore.password=password
```
