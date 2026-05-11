# Kafka Interview Questions & Answers — Final Version

# ZooKeeper

## 1. What are ZooKeeper “other 2 ports” different from 2181 and what are they used for?

**Answer —**

ZooKeeper mainly uses 3 important ports:

| Port | Purpose                          |
| ---- | -------------------------------- |
| 2181 | Client communication             |
| 2888 | Follower-to-Leader communication |
| 3888 | Leader election                  |

Example:

```properties
clientPort=2181

server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

### Port Usage

```text
2181 → Kafka brokers connect to ZooKeeper
2888 → ZooKeeper followers sync with leader
3888 → ZooKeeper servers elect leader
```

---

## 2. What is the use of ZooKeeper in Kafka?

**Answer —**

In older Kafka architecture, ZooKeeper manages cluster metadata and coordination.

Kafka uses ZooKeeper for:

```text
Broker registration
Controller election
Cluster membership
Topic metadata
Partition metadata
ISR tracking
Configuration metadata
ACL storage in older setups
```

Important point:

```text
ZooKeeper does NOT directly manage partition leaders.
Kafka Controller manages partition leaders.
ZooKeeper helps elect the Kafka Controller.
```

---

## 3. Which internal broker layer does ZooKeeper connect to?

**Answer —**

ZooKeeper communication comes into the broker through Kafka’s networking and request-processing layers.

Flow:

```text
ZooKeeper Client inside Broker
        ↓
SocketServer / Network Threads
        ↓
Request Queue
        ↓
KafkaRequestHandler Threads
        ↓
Controller / Metadata logic
```

Important broker configs:

```properties
num.network.threads=3
num.io.threads=8
queued.max.requests=500
```

At networking level:

```text
TCP Layer 4
```

Inside Kafka broker:

```text
SocketServer
NetworkProcessor Threads
KafkaRequestHandler Threads
```

---

## 4. What are zNodes? Explain ZooKeeper leader election.

**Answer —**

A zNode is a metadata object inside ZooKeeper’s hierarchical tree structure.

Example:

```text
/brokers
/brokers/ids/1
/controller
/admin
/config
```

Kafka stores metadata in zNodes.

---

### Types of zNodes

| Type       | Meaning                       | Kafka Example         |
| ---------- | ----------------------------- | --------------------- |
| Persistent | Exists until deleted          | `/brokers/topics`     |
| Ephemeral  | Exists while session is alive | `/brokers/ids/1`      |
| Sequential | Auto sequence numbering       | Coordination/election |

---

### Kafka Broker Registration

Kafka broker registers itself using an ephemeral zNode.

Example:

```text
/brokers/ids/1
```

If broker crashes:

```text
ZooKeeper session expires
Ephemeral zNode disappears
Kafka Controller detects broker failure
```

---

### ZooKeeper Internal Leader Election

ZooKeeper servers elect their own leader.

Example:

```text
zk1
zk2
zk3
```

One becomes:

```text
Leader
```

Others become:

```text
Followers
```

Ports used:

```text
2888 → follower sync
3888 → election
```

---

### Kafka Controller Election

Kafka brokers compete to create:

```text
/controller
```

Whichever broker creates it first becomes Kafka Controller.

Example:

```text
Broker 1 creates /controller
Broker 1 becomes Kafka Controller
```

If controller broker dies:

```text
/controller zNode disappears
Another broker creates it
New controller elected
```

Important interview point:

```text
ZooKeeper helps elect Kafka Controller.
Kafka Controller performs partition leader election.
```

---

# KRaft

## 5. Which is better: KRaft or ZooKeeper and why?

**Answer —**

KRaft is the modern Kafka architecture and is preferred for new deployments.

Benefits:

```text
No external ZooKeeper dependency
Simpler architecture
Better scalability
Faster metadata operations
Easier operations
Better failover handling
Kafka manages its own metadata quorum
```

Old Architecture:

```text
Kafka Brokers → ZooKeeper
```

KRaft Architecture:

```text
Kafka Brokers → Kafka Controller Quorum
```

---

## 6. What are KRaft ports?

**Answer —**

Typical KRaft ports:

| Port | Purpose                         |
| ---- | ------------------------------- |
| 9092 | Client/Broker traffic           |
| 9093 | Controller quorum communication |

Example:

```properties
listeners=PLAINTEXT://broker1:9092,CONTROLLER://broker1:9093

controller.listener.names=CONTROLLER

controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
```

---

## 7. Which internal broker layer does KRaft connect to?

**Answer —**

KRaft controller traffic flows through Kafka network and request-processing layers.

Flow:

```text
Controller Listener
        ↓
SocketServer
        ↓
Network Threads
        ↓
Request Queue
        ↓
KafkaRequestHandler Threads
        ↓
Metadata / Controller Logic
```

Broker configs:

```properties
num.network.threads=3
num.io.threads=8
```

---

## 8. What happens when there is only one KRaft controller instead of a quorum?

**Answer —**

Cluster works but has no fault tolerance.

If single controller fails:

```text
Metadata operations stop
Leader elections fail
Topic creation fails
Partition reassignment fails
Cluster becomes unstable
```

Production recommendation:

```text
3 controllers minimum
5 for larger production clusters
```

---

# Kafka Broker

## 9. What is IBP, Inter-Broker Protocol, and why is it important?

**Answer —**

IBP controls protocol compatibility between brokers.

Example:

```properties
inter.broker.protocol.version=3.6
```

Purpose:

```text
Safe rolling upgrades
Mixed-version broker compatibility
Prevents newer brokers from using unsupported features
```

---

## 10. What is log format version and why is it important?

**Answer —**

Controls message format stored on disk.

Example:

```properties
log.message.format.version=3.6
```

Important because:

```text
Older consumers/brokers may not understand newer message formats
Important during upgrades/downgrades
Affects on-disk record compatibility
```

---

## 11. What is unclean leader election and when is it needed?

**Answer —**

Allows Kafka to elect out-of-sync replica as leader.

Config:

```properties
unclean.leader.election.enable=true
```

Benefit:

```text
Partition availability improves
```

Risk:

```text
Data loss
```

Use only when:

```text
Availability more important than consistency
Emergency recovery
Non-critical workloads
```

---

## 12. What is the difference between ACL and RBAC?

**Answer —**

### ACL

ACL controls:

```text
Who can do what on which resource from where
```

ACL components:

| Component       | Meaning         |
| --------------- | --------------- |
| Principal       | User/service    |
| Operation       | Read/Write/etc  |
| Resource        | Topic/group/etc |
| Permission Type | Allow/Deny      |
| Host            | Source host/IP  |

Example:

```bash
kafka-acls --bootstrap-server broker:9092 \
  --add \
  --allow-principal User:app1 \
  --operation Read \
  --topic orders
```

---

### RBAC

RBAC uses predefined roles.

Common Confluent RBAC Roles:

| Role           | Purpose                      |
| -------------- | ---------------------------- |
| SystemAdmin    | Full platform control        |
| UserAdmin      | Manage users/roles           |
| ClusterAdmin   | Kafka cluster administration |
| DeveloperRead  | Read topics/groups           |
| DeveloperWrite | Write to topics              |
| ResourceOwner  | Full resource ownership      |
| SecurityAdmin  | Security management          |
| Operator       | Operational tasks            |
| AuditAdmin     | Audit log access             |
| MetricsViewer  | View metrics                 |

---

### Difference

| ACL                        | RBAC                          |
| -------------------------- | ----------------------------- |
| Resource-level permissions | Role-based permissions        |
| Granular                   | Easier centralized management |
| More difficult at scale    | Better for enterprises        |

---

## 13. How do you put all messages in a specified partition?

**Answer —**

Specify partition manually in producer.

Java example:

```java
ProducerRecord<String,String> record =
new ProducerRecord<>("orders", 2, "key1", "message");
```

Or use same key:

```text
Same key → same partition
```

Kafka hashes key to determine partition.

---

## 14. Where does Broker perform better: bare metal vs VM?

**Answer —**

Kafka performs better on bare metal.

Benefits:

```text
Lower latency
Better disk I/O
Better network throughput
Predictable performance
No virtualization overhead
```

VMs work fine if:

```text
Dedicated storage
Strong networking
No CPU/memory overcommit
```

---

## 15. Which disk is better suited for Broker: SSD vs HDD?

**Answer —**

SSD is preferred.

Benefits:

```text
Faster replication
Faster recovery
Better random I/O
Lower latency
Better rebalance performance
```

HDD is mostly used for:

```text
Cheap long-term retention
Archive workloads
```

---

## 16. If disk is getting full frequently then how would you address this?

**Answer —**

Check:

```text
Retention
Replication factor
Topic count
Consumer lag
Large messages
Unused topics
```

Commands:

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

Fixes:

```text
Reduce retention.ms
Reduce retention.bytes
Add brokers
Increase storage
Delete unused topics
Enable compression
Reassign partitions
```

Example:

```bash
kafka-configs --bootstrap-server localhost:9092 \
 --alter \
 --entity-type topics \
 --entity-name orders \
 --add-config retention.ms=604800000
```

---

## 17. What are mandatory properties to create a topic?

**Answer —**

Minimum:

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

---

# Kafka Producer

## 18. What are producer settings to maximize throughput?

**Answer —**

```properties
batch.size=65536
linger.ms=5
compression.type=lz4
buffer.memory=67108864
acks=all
enable.idempotence=true
```

Purpose:

```text
Larger batching
Better compression
Higher throughput
Reduced network overhead
```

---

## 19. What are producer settings to minimize latency?

**Answer —**

```properties
linger.ms=0
batch.size=16384
compression.type=none
acks=1
```

Tradeoff:

```text
Lower latency usually reduces throughput/durability
```

---

## 20. What property helps in tackling message ordering?

**Answer —**

```properties
max.in.flight.requests.per.connection=1
```

Also:

```text
Same key → same partition
Ordering guaranteed only within partition
```

---

## 21. How do we tackle message duplication?

**Answer —**

Enable idempotence.

```properties
enable.idempotence=true
acks=all
retries=2147483647
```

For full exactly-once:

```properties
transactional.id=my-tx-app
```

---

# Kafka Consumer

## 22. What is consumer rebalance? How does it work?

**Answer —**

Kafka redistributes partitions among consumers in same group.

Triggers:

```text
Consumer joins
Consumer leaves
Consumer crashes
Partitions added
Session timeout
```

Flow:

```text
Group Coordinator pauses consumers
Partitions reassigned
Consumers resume processing
```

---

## 23. How does Kafka determine if consumer is dead?

**Answer —**

Using heartbeats.

Configs:

```properties
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=300000
```

If heartbeat stops:

```text
Broker marks consumer dead
Rebalance starts
```

---

## 24. What is subscribe() vs assign()?

**Answer —**

### subscribe()

Kafka manages partition assignment automatically.

```java
consumer.subscribe(Arrays.asList("orders"));
```

### assign()

Manual partition assignment.

```java
consumer.assign(Arrays.asList(
 new TopicPartition("orders",0)
));
```

---

## 25. What is exactly-once semantics? How is it different from idempotence?

**Answer —**

### Idempotence

Prevents duplicate producer writes.

```properties
enable.idempotence=true
```

### Exactly-once semantics

Guarantees:

```text
Consume
Process
Produce
Commit offset
```

occur exactly once as transaction.

Uses:

```properties
transactional.id=my-app
```

---

## 26. When does consumer lose data?

**Answer —**

When offset committed before processing finishes.

Bad flow:

```text
Poll
Commit offset
Processing fails
```

Safer:

```properties
enable.auto.commit=false
```

Commit after successful processing.

---

## 27. When does consumer get duplication?

**Answer —**

When processing succeeds but offset commit fails.

Flow:

```text
Process succeeds
Consumer crashes before commit
Message re-read
```

---

## 28. What are consumer settings to maximize throughput?

**Answer —**

```properties
fetch.min.bytes=1048576
fetch.max.wait.ms=500
max.poll.records=500
max.partition.fetch.bytes=1048576
receive.buffer.bytes=65536
```

---

## 29. What are consumer settings to minimize latency?

**Answer —**

```properties
fetch.min.bytes=1
fetch.max.wait.ms=50
max.poll.records=100
```

---

## 30. What property helps in tackling message ordering?

**Answer —**

```text
Same key → same partition
One consumer per partition in group
```

Producer ordering property:

```properties
max.in.flight.requests.per.connection=1
```

---

# Kafka Security

## 31. What are 5 ACL pillars?

**Answer —**

| ACL Pillar      | Meaning             |
| --------------- | ------------------- |
| Principal       | User/service        |
| Operation       | Read/Write/etc      |
| Resource        | Topic/group/cluster |
| Permission Type | Allow/Deny          |
| Host            | Source IP/host      |

---

## 32. What are listeners and advertised.listeners?

**Answer —**

### listeners

Broker binds/listens on these addresses.

```properties
listeners=PLAINTEXT://0.0.0.0:9092
```

### advertised.listeners

Broker tells clients to connect using these addresses.

```properties
advertised.listeners=PLAINTEXT://broker1.company.com:9092
```

---

## 33. What is the use of JAAS file?

**Answer —**

JAAS provides SASL authentication configuration.

Used for:

```text
SASL/PLAIN
SASL/SCRAM
Kerberos
OAuth
```

Broker example:

```properties
KafkaServer {
 org.apache.kafka.common.security.scram.ScramLoginModule required
 username="broker"
 password="broker-secret";
};
```

Client example:

```properties
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
 username="app1" \
 password="secret";
```

---

## 34. What are high-level steps to setup SSL in Broker?

**Answer —**

```text
1. Create CA
2. Create broker keystore
3. Generate CSR
4. Sign certificate
5. Import certificates
6. Configure SSL properties
7. Configure clients
8. Restart brokers
9. Test connectivity
```

---

## 35. What is 2-way and 3-way handshake?

**Answer —**

### TCP 3-Way Handshake

```text
SYN
SYN-ACK
ACK
```

Establishes TCP connection.

---

### One-Way SSL

```text
Client verifies broker certificate
```

---

### Two-Way SSL / Mutual TLS

```text
Client verifies broker
Broker verifies client
```

---

## 36. What is session.timeout? How does broker handle client connections?

**Answer —**

Controls how long broker waits before declaring consumer dead.

```properties
session.timeout.ms=45000
```

Broker flow:

```text
Heartbeat received → consumer alive
Heartbeat stops → consumer removed
Rebalance triggered
```

---

## 37. What is quota?

**Answer —**

Kafka quotas limit client resource usage.

Can limit:

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
 --entity-type clients \
 --entity-name app1 \
 --add-config producer_byte_rate=1048576
```

---

## 38. How can you increase client connection duration in Kafka?

**Answer —**

Tune:

```properties
connections.max.idle.ms=900000
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=600000
request.timeout.ms=120000
```

---

# Kafka Schema Registry

## 39. What is schema compatibility? What are compatibility types?

**Answer —**

Schema compatibility controls whether new schema versions safely work with old data.

Types:

| Type                | Meaning                            |
| ------------------- | ---------------------------------- |
| BACKWARD            | New consumers read old data        |
| FORWARD             | Old consumers read new data        |
| FULL                | Both directions                    |
| NONE                | No checks                          |
| BACKWARD_TRANSITIVE | Compatible with all older versions |
| FORWARD_TRANSITIVE  | Forward across all versions        |
| FULL_TRANSITIVE     | Both across all versions           |

Safe change:

```text
Add optional field with default
```

Unsafe:

```text
Rename field
Change datatype
Remove required field
```

---

## 40. What is high-level process of introducing schema to topic first time?

**Answer —**

```text
1. Create schema
2. Choose format (Avro/JSON/Protobuf)
3. Set compatibility mode
4. Register schema
5. Configure producer
6. Producer serializes message
7. Consumer fetches schema ID
8. Consumer deserializes message
```

---

## 41. What is schema registration?

**Answer —**

Schema is stored in Schema Registry under a subject.

Example:

```text
orders-value
```

Schema Registry assigns schema ID.

Producer message contains:

```text
Magic byte
Schema ID
Payload
```

---

## 42. How do you check schema compatibility?

**Answer —**

```bash
curl -X POST \
 http://schema-registry:8081/compatibility/subjects/orders-value/versions/latest
```

Set compatibility:

```bash
curl -X PUT \
 http://schema-registry:8081/config/orders-value \
 -d '{"compatibility":"BACKWARD"}'
```

---

# Kafka Connect

## 43. How do you add worker node to existing Connect cluster?

**Answer —**

New worker must use same:

```properties
group.id=connect-cluster
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
```

Then start worker.

Cluster automatically rebalances tasks.

---

## 44. What is high-level process of installing new connector?

**Answer —**

```text
1. Download connector
2. Place in plugin.path
3. Restart workers
4. Verify plugin
5. Create JSON config
6. Submit connector
7. Verify status
8. Monitor tasks/logs
```

Check plugins:

```bash
curl http://connect:8083/connector-plugins
```

---

## 45. How do you improve connector performance?

**Answer —**

### Increase parallelism

```json
"tasks.max":"6"
```

Requires:

```text
Enough topic partitions
Connector supports parallel tasks
External system supports parallelism
```

---

### Increase partitions

```bash
kafka-topics --alter \
 --topic orders \
 --partitions 12
```

---

### Add more workers

Same group.id and internal topics.

---

### Tune batch settings

JDBC Sink:

```json
"batch.size":"3000"
```

JDBC Source:

```json
"batch.max.rows":"5000"
```

S3 Sink:

```json
"flush.size":"10000"
```

---

### Producer tuning for source connectors

```properties
producer.override.batch.size=65536
producer.override.linger.ms=5
producer.override.compression.type=lz4
```

---

### Consumer tuning for sink connectors

```properties
consumer.override.max.poll.records=1000
consumer.override.fetch.min.bytes=1048576
```

---

### Reduce SMT/converter overhead

Too many SMTs reduce performance.

---

### Check external bottlenecks

```text
Database
API rate limits
Indexes
Network latency
Disk
```

---

## 46. How does Kafka Connect ensure workers know about each other’s configurations?

**Answer —**

Using internal Kafka topics:

```properties
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
```

Workers share same:

```properties
group.id=connect-cluster
```

---

## 47. Where does Connect store committed offset?

**Answer —**

Distributed mode:

```properties
offset.storage.topic=connect-offsets
```

Standalone mode:

```properties
offset.storage.file.filename=/tmp/connect.offsets
```

---

# Kafka On-Prem

## 48. Which one will you choose: bare metal vs VM?

**Answer —**

For production Kafka:

```text
Prefer bare metal
```

Because:

```text
Better disk I/O
Better network
Lower latency
Predictable performance
```

---

## 49. How is On-Prem capacity planning different from Confluent Cloud?

**Answer —**

On-Prem:

```text
You manage:
CPU
Memory
Disk
Network
Brokers
Controllers
Monitoring
Patching
Scaling
Rack awareness
```

Confluent Cloud:

```text
Infrastructure managed by Confluent
Focus mostly on throughput/storage/cost
```

---

## 50. Which one do you prefer SSD or HDD?

**Answer —**

SSD preferred because:

```text
Faster replication
Faster recovery
Lower latency
Better throughput
```

---

## 51. What is Zero Copy transfer?

**Answer —**

Kafka uses OS `sendfile()` to transfer data directly from disk/page cache to network socket.

Without Zero Copy:

```text
Disk → Kernel → JVM Memory → Socket → Network
```

With Zero Copy:

```text
Disk/Page Cache → Socket → Network
```

Benefits:

```text
Less CPU usage
Less memory copying
Higher throughput
Faster fetches/replication
```

Important:

```text
Mostly helps consumer fetches and broker replication traffic.
```

---

## 52. How does Kafka client connect to server? Which internal layer handles it?

**Answer —**

Flow:

```text
Client TCP connection
        ↓
Broker Listener
        ↓
SocketServer
        ↓
Network Threads
        ↓
Request Queue
        ↓
KafkaRequestHandler Threads
        ↓
Kafka APIs
        ↓
ReplicaManager / GroupCoordinator / Controller
```

Important configs:

```properties
num.network.threads=3
num.io.threads=8
queued.max.requests=500
```

---

## 53. What is Rack Awareness?

**Answer —**

Rack awareness helps Kafka distribute replicas across different racks/availability zones/datacenters.

Purpose:

```text
Prevent all replicas from sitting in same rack
Improve fault tolerance
Avoid rack-level failure causing data loss
```

Example:

```text
Rack A → Broker 1
Rack B → Broker 2
Rack C → Broker 3
```

Kafka tries to place replicas across racks.

Broker config:

```properties
broker.rack=rack-a
```

Example:

```properties
broker.rack=dc1-rack1
```

### Why Important

Without rack awareness:

```text
All replicas may land in same rack
Rack failure may lose partition availability
```

With rack awareness:

```text
Replica spread improves resilience
```

### Example

Topic replication factor:

```text
RF=3
```

Kafka tries:

```text
Replica 1 → Rack A
Replica 2 → Rack B
Replica 3 → Rack C
```

### Benefits

```text
Better HA
Better DR
Survives rack/power/network failure
```

Important interview line:

```text
Rack awareness ensures Kafka replicas are distributed across failure domains instead of being concentrated in same physical infrastructure.
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MjEwNTcyMzVdfQ==
-->