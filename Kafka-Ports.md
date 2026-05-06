## Kafka Default Ports (Complete Quick Revision)

### Core Apache Kafka / Broker

| Component                              | Default Port | Purpose                                           |
| -------------------------------------- | -----------: | ------------------------------------------------- |
| Kafka Broker                           |         9092 | Client connections (Producer / Consumer / Admin)  |
| Kafka Broker (SSL)                     |         9093 | SSL/TLS secured listener (common convention)      |
| Kafka Broker (Internal / Inter-broker) |        9094+ | Broker-to-broker communication (depends on setup) |

---

### ZooKeeper Mode (Legacy)

| Component               | Default Port | Purpose                                 |
| ----------------------- | -----------: | --------------------------------------- |
| ZooKeeper Client Port   |         2181 | Brokers connect to ZooKeeper            |
| ZooKeeper Follower Port |         2888 | ZooKeeper server-to-server sync         |
| ZooKeeper Election Port |         3888 | Leader election between ZooKeeper nodes |

---

### KRaft Mode (No ZooKeeper)

| Component             |  Default Port | Purpose                            |
| --------------------- | ------------: | ---------------------------------- |
| Controller Listener   | 9093 / custom | Controller quorum communication    |
| Broker Listener       |          9092 | Client traffic                     |
| Inter-broker Listener | 9094 / custom | Replication + broker communication |

> In KRaft, ZooKeeper ports disappear completely.

---

### Confluent Platform Components

| Component                |      Default Port | Purpose                |
| ------------------------ | ----------------: | ---------------------- |
| Schema Registry          |              8081 | Schema management      |
| Kafka Connect            |              8083 | Connect REST API       |
| KSQL Server              |              8088 | ksqlDB REST API        |
| Confluent Control Center |              9021 | UI monitoring          |
| REST Proxy               |              8082 | REST access to Kafka   |
| Replicator               |      Uses Connect | Runs via Kafka Connect |
| Cluster Linking          | Uses broker ports | No separate port       |

---

### Monitoring / Metrics

| Component               |  Default Port | Purpose          |
| ----------------------- | ------------: | ---------------- |
| JMX                     | 9999 (common) | Java monitoring  |
| Prometheus JMX Exporter | 7071 / custom | Metrics scraping |

---

### Security Related

| Component      | Common Port | Purpose                 |
| -------------- | ----------: | ----------------------- |
| SASL_PLAINTEXT |        9092 | Auth without encryption |
| SASL_SSL       |        9093 | Auth + encryption       |
| SSL            |        9093 | Encryption              |

---

## Interview One-Liner 🎯

**9092 = Kafka**
**2181 = ZooKeeper**
**8081 = Schema Registry**
**8083 = Connect**
**9021 = Control Center**
**8088 = ksqlDB**
**8082 = REST Proxy**

These are the ones most interviewers expect immediately.
