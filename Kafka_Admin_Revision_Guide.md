# Kafka Administrator — Interview Preparation Guide

**Target role:** Kafka Administrator / Software Engineer (Federal, On-Prem)
**Environment:** PeopleSoft HCM integrations · Linux + UNIX/Solaris · Secret-cleared, no-internet enclave
**Prepared for:** Prateek Shukla — Confluent Certified Kafka Administrator & Developer

> **How to use this guide.** This maps directly to the posting. Each section explains what to know, then gives the **actual commands, configs, and procedures** you should be able to run and talk through. Study the **Operations**, **Security**, and **Incident** sections hardest — that is where this role lives. Where your Confluent Cloud background differs from on-prem, the gap is called out so you can pre-empt it.

> **Note on versions.** Commands use the modern `--bootstrap-server` form. Older clusters you may inherit still use `--zookeeper` for some topic/ACL operations — both are shown where it matters. Binary names assume the vanilla Apache distribution (`kafka-topics.sh`); Confluent packaging drops the `.sh` (`kafka-topics`). Use whichever the host has.

---

## Contents

1. [Reading the Role](#1-reading-the-role)
2. [Fundamentals & Architecture](#2-fundamentals--architecture)
3. [Cluster Setup — 3-Broker Cluster (ZooKeeper & KRaft)](#3-cluster-setup--3-broker-cluster)
4. [Install, Configure, Patch & Upgrade](#4-install-configure-patch--upgrade)
5. [Topics, Partitions & Consumer Groups](#5-topics-partitions--consumer-groups)
6. [Security — TLS, SASL, ACLs, Certs](#6-security--tls-sasl-acls-certs)
7. [Monitoring, Metrics & Alerting](#7-monitoring-metrics--alerting)
8. [Incident & Problem Management (Runbooks)](#8-incident--problem-management-runbooks)
9. [HA/DR, Backup, Capacity & Change Control](#9-hadr-backup-capacity--change-control)
10. [Bridging Your Confluent Cloud Background](#10-bridging-your-confluent-cloud-background)
11. [PeopleSoft HCM Integration Context](#11-peoplesoft-hcm-integration-context)
12. [Scripting & Automation](#12-scripting--automation)
13. [Quick Reference (Command Cheat Sheet)](#13-quick-reference-command-cheat-sheet)
14. [Questions to Ask Them](#14-questions-to-ask-them)

---

## 1. Reading the Role

**What this job actually is:** a hands-on platform operations role keeping a secured, on-prem Kafka estate alive in support of PeopleSoft HCM. Not a cloud-streaming architecture role, not a developer role. The interview tests whether you can **install, secure, monitor, troubleshoot, and document** Kafka on bare Linux/Solaris under federal change control.

What they're really screening for:

| Screen | What they want to hear |
|---|---|
| On-prem ops depth | You've run Kafka without a managed control plane — broker installs, configs, OS tuning, manual recovery. |
| Security posture | TLS/SSL, SASL, ACLs, certificate lifecycle, audit/compliance are second nature. |
| Incident discipline | You triage lag, broker failure, disk pressure, ISR shrink methodically, with runbooks. |
| Process maturity | Change management, SOPs, backup/recovery, capacity planning — you operate inside controls. |
| Clearance fit | U.S. citizen, able to hold Secret, comfortable in a restricted enclave. |

**Positioning:** lead with Kafka internals and operations that transfer. Translate cloud reflexes into on-prem equivalents (Section 10). Never imply you'd reach for a managed service when the answer is a manual procedure.

---

## 2. Fundamentals & Architecture

- **Topic** → split into **partitions** (unit of parallelism + ordering).
- **Partition** → ordered, append-only log, replicated across **brokers**.
- **Leader** handles all reads/writes for a partition; **followers** replicate.
- **Consumer group** → partitions distributed across members; each partition consumed by exactly one member in the group.
- **Coordination** → ZooKeeper (legacy) or KRaft (modern, internal Raft quorum).

**Durability triad — memorize this:**

```
Replication Factor (RF)  +  min.insync.replicas (ISR floor)  +  producer acks
```

With `RF=3`, `min.insync.replicas=2`, `acks=all`: survive **one** broker loss with **no data loss** and still accept writes.

**ISR (in-sync replicas):** replicas caught up to the leader within `replica.lag.time.max.ms`. A write commits only when all ISR members have it.

**Cleanup policies:**
- `delete` → drop old segments past `retention.ms` / `retention.bytes` (event streams).
- `compact` → keep latest value per key forever (changelog/state).
- `compact,delete` → compacted but tombstones still age out.

---

## 3. Cluster Setup — 3-Broker Cluster

This is the section to be fluent in. Be able to stand up a 3-broker cluster from scratch, both ways.

### 3a. OS prep (every broker, do first)

```bash
# JDK (Kafka needs Java 11+ / 17+ depending on version)
sudo apt-get install -y openjdk-17-jdk    # or: yum install java-17-openjdk

# Dedicated user + dirs
sudo useradd -r -m -d /opt/kafka kafka
sudo mkdir -p /data/kafka-logs            # data on its OWN disk/volume
sudo chown -R kafka:kafka /data/kafka-logs

# OS tuning — Kafka opens lots of files and mmaps heavily
# /etc/security/limits.conf
kafka  -  nofile  100000

# /etc/sysctl.d/99-kafka.conf
vm.swappiness=1
vm.max_map_count=262144
sudo sysctl --system
```

```bash
# Download + unpack (offline enclave: pull the tarball through approved channels first)
cd /opt/kafka
tar -xzf kafka_2.13-3.7.0.tgz
ln -s kafka_2.13-3.7.0 current
```

### 3b. Option A — ZooKeeper mode (3 ZK + 3 brokers)

Many federal estates still run this. Quorum must be **odd** (3 or 5).

**ZooKeeper config** (`config/zookeeper.properties`, per ZK node):

```properties
dataDir=/data/zookeeper
clientPort=2181
tickTime=2000
initLimit=10
syncLimit=5
# the ensemble — same on all 3 nodes
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

```bash
# On each ZK node, write its id (matches server.N above)
echo 1 | sudo tee /data/zookeeper/myid   # 2 on zk2, 3 on zk3

# Start ZooKeeper
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

**Broker config** (`config/server.properties`) — change `broker.id` per broker:

```properties
broker.id=1                               # 2 and 3 on the others
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://broker1:9092
log.dirs=/data/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181

# durability defaults for a 3-broker cluster
default.replication.factor=3
min.insync.replicas=2
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# rack awareness (spread replicas across failure domains)
broker.rack=rack-a                        # rack-b / rack-c on the others
```

```bash
# Start each broker
bin/kafka-server-start.sh -daemon config/server.properties
```

### 3c. Option B — KRaft mode (no ZooKeeper)

Modern default. The 3 nodes act as combined controller+broker here.

**`config/kraft/server.properties`** (per node, change `node.id`):

```properties
process.roles=broker,controller
node.id=1                                 # 2 and 3 on the others
controller.quorum.voters=1@host1:9093,2@host2:9093,3@host3:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://host1:9092
controller.listener.names=CONTROLLER
inter.broker.listener.name=PLAINTEXT
log.dirs=/data/kafka-logs

default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3
```

```bash
# 1) Generate ONE cluster UUID, reuse it on all nodes
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
echo $KAFKA_CLUSTER_ID

# 2) Format storage on EACH node with that same UUID
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID \
  -c config/kraft/server.properties

# 3) Start each node
bin/kafka-server-start.sh -daemon config/kraft/server.properties
```

### 3d. Run brokers under systemd (production)

`/etc/systemd/system/kafka.service`:

```ini
[Unit]
Description=Apache Kafka
After=network.target

[Service]
User=kafka
Environment="KAFKA_HEAP_OPTS=-Xmx6g -Xms6g"
ExecStart=/opt/kafka/current/bin/kafka-server-start.sh /opt/kafka/current/config/server.properties
ExecStop=/opt/kafka/current/bin/kafka-server-stop.sh
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka
sudo systemctl status kafka
journalctl -u kafka -f          # tail broker logs
```

### 3e. Verify the cluster is healthy

```bash
B=broker1:9092

# Brokers registered?
bin/kafka-broker-api-versions.sh --bootstrap-server $B | grep -c "id"

# Create a test topic and confirm replication
bin/kafka-topics.sh --bootstrap-server $B --create \
  --topic smoke-test --partitions 3 --replication-factor 3

bin/kafka-topics.sh --bootstrap-server $B --describe --topic smoke-test
# Look for: Leader assigned, Replicas: 1,2,3, Isr: 1,2,3 (all three in sync)

# Produce + consume round trip
echo "hello" | bin/kafka-console-producer.sh --bootstrap-server $B --topic smoke-test
bin/kafka-console-consumer.sh --bootstrap-server $B --topic smoke-test \
  --from-beginning --max-messages 1

# Cleanup
bin/kafka-topics.sh --bootstrap-server $B --delete --topic smoke-test
```

---

## 4. Install, Configure, Patch & Upgrade

### 4a. Rolling upgrade (zero downtime)

The single most important operational procedure. **One broker at a time, ISR-gated.**

```bash
# 1) Pin protocol + message format to the CURRENT (old) version, in server.properties:
#    inter.broker.protocol.version=3.6
#    log.message.format.version=3.6

# 2) For EACH broker, one at a time:
#    a. controlled shutdown (moves leadership off cleanly)
sudo systemctl stop kafka

#    b. swap binaries / update the symlink
ln -sfn /opt/kafka/kafka_2.13-3.7.0 /opt/kafka/current

#    c. start, then WAIT for full recovery before the next broker
sudo systemctl start kafka

#    d. GATE: do not proceed until under-replicated partitions == 0
bin/kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions
# (empty output = healthy = safe to move to next broker)

# 3) After ALL brokers are on new binaries, bump the protocol version
#    inter.broker.protocol.version=3.7  → second rolling restart
```

**Rule:** never restart the next broker while any partition is under-replicated.

### 4b. JVM / GC — what to say

```bash
# G1GC, modest heap, rest of RAM goes to OS page cache (NOT heap)
export KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20"
```

Long GC pauses drop a broker out of the ISR and can break its ZK/KRaft session — so heap stays modest and you watch pause time.

---

## 5. Topics, Partitions & Consumer Groups

### 5a. Topic provisioning

```bash
B=broker1:9092

# Create with explicit settings (a real provisioning decision)
bin/kafka-topics.sh --bootstrap-server $B --create \
  --topic hcm.payroll.events \
  --partitions 6 --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete

# List / describe
bin/kafka-topics.sh --bootstrap-server $B --list
bin/kafka-topics.sh --bootstrap-server $B --describe --topic hcm.payroll.events
```

### 5b. Altering topics (mind the catches)

```bash
# Change retention LIVE (no restart)
bin/kafka-configs.sh --bootstrap-server $B --alter \
  --entity-type topics --entity-name hcm.payroll.events \
  --add-config retention.ms=259200000

# Increase partitions — ONE WAY, and it BREAKS key ordering going forward
bin/kafka-topics.sh --bootstrap-server $B --alter \
  --topic hcm.payroll.events --partitions 12
# You cannot decrease. For a keyed/compacted topic this is a real hazard — size up front.

# View current overrides
bin/kafka-configs.sh --bootstrap-server $B --describe \
  --entity-type topics --entity-name hcm.payroll.events
```

### 5c. Consumer group lag — the bread-and-butter diagnostic

```bash
# The command you'll run constantly
bin/kafka-consumer-groups.sh --bootstrap-server $B --describe --group etl-consumer
# Columns: PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID  HOST
#   LAG = LOG-END-OFFSET - CURRENT-OFFSET

# List all groups / their state
bin/kafka-consumer-groups.sh --bootstrap-server $B --list
bin/kafka-consumer-groups.sh --bootstrap-server $B --describe --group etl-consumer --state
```

**Triage flow when lag is high:**
1. Lag on **all** partitions → consumer-side (capacity, slow processing, GC).
2. Lag on **a few** → skewed keys, one slow partition, or a stuck member.
3. Check consumers alive + committing; watch for rebalance storms or a poison message.
4. Check brokers: hot/disk-pressured leader, under-replicated partitions, network.
5. Remediate: scale consumers (≤ partition count), fix processing, raise fetch sizes, rebalance. Offset surgery **last**.

### 5d. Offset reset (group must be STOPPED — data-affecting, change-controlled)

```bash
# Dry run first (always)
bin/kafka-consumer-groups.sh --bootstrap-server $B --group etl-consumer \
  --reset-offsets --to-earliest --topic hcm.payroll.events --dry-run

# Execute (replay from start)
bin/kafka-consumer-groups.sh --bootstrap-server $B --group etl-consumer \
  --reset-offsets --to-earliest --topic hcm.payroll.events --execute

# Other targets: --to-latest  --to-datetime 2026-06-22T00:00:00.000  --shift-by -100
```

### 5e. Partition reassignment / rebalancing (with throttle)

```bash
# 1) Define which topics to move (topics.json)
cat > topics.json <<'EOF'
{"topics":[{"topic":"hcm.payroll.events"}],"version":1}
EOF

# 2) Generate a proposed plan for target brokers
bin/kafka-reassign-partitions.sh --bootstrap-server $B \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3" --generate
# Save the "Proposed partition reassignment" block to reassign.json

# 3) Execute WITH a throttle so replication doesn't starve production
bin/kafka-reassign-partitions.sh --bootstrap-server $B \
  --reassignment-json-file reassign.json --execute \
  --throttle 50000000        # 50 MB/s

# 4) Verify (also removes the throttle when complete)
bin/kafka-reassign-partitions.sh --bootstrap-server $B \
  --reassignment-json-file reassign.json --verify
```

```bash
# Rebalance leadership back to preferred replicas
bin/kafka-leader-election.sh --bootstrap-server $B \
  --election-type PREFERRED --all-topic-partitions
```

---

## 6. Security — TLS, SASL, ACLs, Certs

**The heart of the role.** Three independent layers — name and distinguish all three:

1. **Encryption in transit** → TLS/SSL (client↔broker and inter-broker), often mutual TLS.
2. **Authentication** → SASL (GSSAPI/Kerberos common in federal, or SCRAM) or mTLS client certs.
3. **Authorization** → ACLs via the authorizer.

Plus **encryption at rest** (disk/volume — Kafka has no native at-rest encryption) and **audit logging**.

### 6a. TLS setup — generate keystores/truststores

```bash
# 1) Create a CA (in prod this is your enterprise/federal CA — don't self-sign)
openssl req -new -x509 -keyout ca-key -out ca-cert -days 3650 -nodes \
  -subj "/CN=Kafka-CA"

# 2) Per broker: keystore with its key
keytool -keystore broker1.keystore.jks -alias broker1 -validity 365 -genkey \
  -keyalg RSA -storepass changeit -keypass changeit \
  -dname "CN=broker1" -ext SAN=DNS:broker1

# 3) Sign the broker cert with the CA
keytool -keystore broker1.keystore.jks -alias broker1 -certreq -file cert-req -storepass changeit
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-req -out cert-signed \
  -days 365 -CAcreateserial
keytool -keystore broker1.keystore.jks -alias CARoot -import -file ca-cert -storepass changeit -noprompt
keytool -keystore broker1.keystore.jks -alias broker1 -import -file cert-signed -storepass changeit -noprompt

# 4) Truststore trusts the CA (every broker + client gets this)
keytool -keystore client.truststore.jks -alias CARoot -import -file ca-cert -storepass changeit -noprompt
```

**Broker `server.properties` for TLS:**

```properties
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://broker1:9093
security.inter.broker.protocol=SSL

ssl.keystore.location=/opt/kafka/ssl/broker1.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.location=/opt/kafka/ssl/client.truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required          # required = mutual TLS
```

**Client properties file (`client-ssl.properties`):**

```properties
security.protocol=SSL
ssl.truststore.location=/opt/kafka/ssl/client.truststore.jks
ssl.truststore.password=changeit
ssl.keystore.location=/opt/kafka/ssl/client.keystore.jks   # for mTLS
ssl.keystore.password=changeit
```

```bash
# Verify TLS handshake from a client
openssl s_client -connect broker1:9093 -showcerts </dev/null
bin/kafka-topics.sh --bootstrap-server broker1:9093 \
  --command-config client-ssl.properties --list
```

### 6b. Certificate rotation (zero downtime, rolling)

```
1. Generate new keystore + new CA (if CA is rotating).
2. Import BOTH old and new CA into every truststore (trust overlap).
3. Roll brokers one at a time onto the new keystore; verify handshake each time.
4. Once all brokers + clients use the new CA, remove the old CA from truststores.
5. Track expiry in a cert inventory; alert weeks ahead.
```

```bash
# Check a broker cert's expiry (the check your cron should run)
echo | openssl s_client -connect broker1:9093 2>/dev/null \
  | openssl x509 -noout -enddate
# Inside a JKS:
keytool -list -v -keystore broker1.keystore.jks -storepass changeit | grep "until"
```

### 6c. SASL/SCRAM (simpler than Kerberos to demo)

```bash
# Create a SCRAM credential for a user
bin/kafka-configs.sh --bootstrap-server $B --alter \
  --entity-type users --entity-name peoplesoft-prod \
  --add-config 'SCRAM-SHA-512=[password=secret]'
```

```properties
# broker server.properties
listeners=SASL_SSL://:9094
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-secret";
```

**SASL/Kerberos (GSSAPI) — what to say in federal context:** each broker/client has a **keytab** + service principal; JAAS points to it; clients get a TGT and authenticate via GSSAPI. You manage keytab distribution, principal naming, **clock sync** (Kerberos is time-sensitive), and ticket renewal — owned jointly with the security/identity team.

### 6d. ACLs (deny-by-default, least privilege)

```properties
# Enable the authorizer (server.properties)
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer  # KRaft
# (ZK clusters: kafka.security.authorizer.AclAuthorizer)
allow.everyone.if.no.acl.found=false        # deny by default
super.users=User:admin
```

```bash
# Let the PeopleSoft producer WRITE to one topic
bin/kafka-acls.sh --bootstrap-server $B --add \
  --allow-principal User:peoplesoft-prod \
  --operation Write --operation Describe \
  --topic hcm.payroll.events

# Let the ETL group READ
bin/kafka-acls.sh --bootstrap-server $B --add \
  --allow-principal User:etl-svc \
  --operation Read --operation Describe \
  --topic hcm.payroll.events --group etl-consumer

# Audit: list everything
bin/kafka-acls.sh --bootstrap-server $B --list

# Remove
bin/kafka-acls.sh --bootstrap-server $B --remove \
  --allow-principal User:peoplesoft-prod --operation Write --topic hcm.payroll.events
```

Scope to named topics/groups, grant only needed operations, avoid wildcards, review periodically for audit.

---

## 7. Monitoring, Metrics & Alerting

### 7a. Signals that matter

| Signal | Healthy value / meaning |
|---|---|
| Under-replicated partitions | **0** — non-zero = broker behind/down, durability at risk |
| Offline partitions | **0** — non-zero = partition has no leader, unavailable → page now |
| Active controller count | **exactly 1** cluster-wide |
| Consumer lag | low / not growing — rising = consumers can't keep up |
| ISR shrink/expand rate | low — frequent shrinks = flapping brokers, GC, network |
| Request p99 latency | trending flat — rising = saturation coming |
| Disk per log dir | well under full — a full log dir takes the broker offline |
| JVM heap / GC pause | short pauses — long ones drop brokers from ISR |

### 7b. Collecting metrics on-prem (no SaaS)

```bash
# Kafka exposes metrics via JMX. Enable JMX on the broker:
export JMX_PORT=9999
# Then scrape with the Prometheus JMX exporter as a javaagent:
export KAFKA_OPTS="-javaagent:/opt/kafka/jmx_prometheus_javaagent.jar=7071:/opt/kafka/kafka-jmx.yml"
# → Prometheus scrapes :7071 → Grafana dashboards → Alertmanager
```

```bash
# Quick CLI health checks (no dashboard needed)
bin/kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions
bin/kafka-topics.sh --bootstrap-server $B --describe --unavailable-partitions
df -h /data/kafka-logs            # disk
journalctl -u kafka --since "10 min ago" | grep -i error
```

**Lag nuance:** alert on **sustained, growing** lag, not a single spike (replays/deploys spike then drain). Flat-high lag with healthy consumers = slow-but-keeping-up; growing lag = the real problem.

---

## 8. Incident & Problem Management (Runbooks)

Method first: **detect → triage → mitigate → resolve → RCA → prevent.**

### 8a. Runbook — broker down

```bash
# 1) Scope: which broker, process-down or host-down?
sudo systemctl status kafka
journalctl -u kafka --since "30 min ago" | tail -50

# 2) Is the cluster still serving? (RF=3, min.isr=2 survives one loss)
bin/kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions
bin/kafka-topics.sh --bootstrap-server $B --describe --unavailable-partitions

# 3) Triage root cause
df -h /data/kafka-logs                       # disk full?
grep -i "OutOfMemory\|GC pause\|expired" /opt/kafka/logs/server.log   # OOM / cert?
dmesg | tail                                  # hardware / OOM killer

# 4) Mitigate: free disk / restart / fail the host
sudo systemctl start kafka

# 5) Recover: confirm rejoin + ISR full
bin/kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions
# 6) If leadership didn't move back:
bin/kafka-leader-election.sh --bootstrap-server $B --election-type PREFERRED --all-topic-partitions

# 7) Communicate status, log the change, open an RCA.
```

### 8b. Runbook — disk filling fast

```bash
# Confirm it's the log dirs, find the biggest topics
du -sh /data/kafka-logs/* | sort -rh | head

# Honor retention sooner on the offending topic (ages data out)
bin/kafka-configs.sh --bootstrap-server $B --alter --entity-type topics \
  --entity-name <topic> --add-config retention.ms=3600000

# Or add a second log dir on another volume (log.dirs=/data1,/data2) + restart
# Or reassign partitions OFF the hot broker (throttled — see 5e)
```
**Never** delete log segment files by hand — it corrupts the partition.

### 8c. Runbook — `NOT_ENOUGH_REPLICAS` on produce

Cause: in-sync replicas dropped below `min.insync.replicas` while producer used `acks=all`. A follower fell out of the ISR (broker down, GC, network, disk). Kafka is **correctly** refusing the write to protect durability.

```bash
# Find which partitions are under-replicated and on which broker
bin/kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions
# Fix the underlying broker/replication health to restore the ISR.
# DO NOT "fix" it by lowering min.insync.replicas in prod under pressure.
```

### 8d. Poison message stalling a consumer

Identify the offset/partition where the consumer loops or dies → route to a dead-letter topic, **or** (group stopped, change-controlled) skip via offset shift (`--shift-by 1`) → feed back to the app team so the consumer handles bad records. You enable recovery; the durable fix is in consumer code.

---

## 9. HA/DR, Backup, Capacity & Change Control

### 9a. High availability (within a cluster)

```properties
default.replication.factor=3
min.insync.replicas=2
# producer side: acks=all
broker.rack=rack-a            # rack awareness spreads replicas across failure domains
```
Odd controller quorum (3 or 5). Cluster tolerates broker loss transparently; clients reconnect to new leaders.

### 9b. Disaster recovery — MirrorMaker 2 (cross-site)

Intra-cluster replication ≠ site protection. For cross-site, replicate to a second cluster with **MirrorMaker 2**.

```properties
# mm2.properties
clusters=primary,dr
primary.bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
dr.bootstrap.servers=drbroker1:9092,drbroker2:9092
primary->dr.enabled=true
primary->dr.topics=hcm\..*
replication.factor=3
```
```bash
bin/connect-mirror-maker.sh mm2.properties
```
Tradeoff to mention: offsets don't match 1:1 across clusters (MM2 offset translation); plan and **test** failover runbooks. For federal, both clusters live inside the accredited boundary.

### 9c. Backup & recovery (be honest)

Replication is HA, **not** backup — it won't save you from accidental delete, bad retention, or corruption. Approaches: (1) replicate critical topics to a DR cluster; (2) sink important topics to durable storage via a connector for cold recovery; (3) back up **configs, ACLs, certs/keytabs, cluster metadata**. Kafka "backup" is mostly replication + config/metadata capture — say so plainly.

### 9d. Capacity planning

```
raw storage ≈ ingest_rate (MB/s) × retention_seconds × replication_factor + headroom
```
Add partition growth, consumer fan-out, peak vs average. Track trends so you provision **ahead** of a full disk. Tie it to the change calendar.

### 9e. Change management (federal)

Every prod change — broker config, topic/ACL, patch, cert rotation — goes through a change record with **plan, rollback, test evidence, approval window**. No ad-hoc prod changes. Keep runbooks/SOPs current; maintain configs, ACL lists, and cert inventories for **audit readiness**. Position yourself as comfortable inside controls.

---

## 10. Bridging Your Confluent Cloud Background

Pre-empt: *"You've run managed Kafka — can you run it on bare metal without the control plane?"*

| Confluent Cloud gave you | On-prem, you own it manually |
|---|---|
| Auto provisioning & scaling | Host sizing, binary installs, systemd, manual reassignment (5e) |
| Managed TLS & keys | Keystores/truststores, enterprise CA, rolling cert rotation (6b) |
| Built-in dashboards | Self-hosted JMX exporter + Prometheus/Grafana (7b) |
| Rolling upgrades for you | You run the ISR-gated rolling upgrade (4a) |
| Self-balancing | `kafka-reassign-partitions` with throttles (5e) |
| SLA/DR abstracted | You design RF, rack awareness, MirrorMaker 2, test failover (9) |

**Line to deliver:** *"Confluent Cloud made me fluent in what good looks like — durability settings, security layers, lag and replication health. On-prem I'm comfortable doing those same jobs by hand because I understand the internals under the managed layer, and my Terraform/Kubernetes background means automation and infra discipline come naturally."*

---

## 11. PeopleSoft HCM Integration Context

You won't be the PeopleSoft developer — show you understand what Kafka is **for** here.

- **Role:** Kafka is the integration backbone moving HCM data (employee, payroll, benefits, org events) between PeopleSoft and downstream ETL/warehouse/apps. PeopleSoft emits events/batch messages onto topics; consumers pick them up. You keep it flowing: topic provisioning, per-principal ACLs, fit-for-purpose retention, monitoring so a stalled consumer doesn't silently desync HR data.
- **Batch load:** PeopleSoft batch windows are bursty — a nightly run floods a topic, spiking lag and disk. Size/partition for **peak**, watch lag during batch windows, ensure retention covers a consumer outage without data loss. Coordinate timing with app/ETL teams.
- **Stakeholders (from the posting):** application, middleware, ETL, database, infrastructure, **cybersecurity**. You're the Kafka owner brokering between integration developers (need topics, access, reliability) and security/infra (own certs, network, compliance). It's a cross-team operations seat — emphasize collaboration and clear comms.

---

## 12. Scripting & Automation

The posting names **scripts, automations, runbooks, SOPs** as deliverables — treat them as version-controlled, audit-ready artifacts.

**First script to write — a cluster health one-shot:**

```bash
#!/usr/bin/env bash
# kafka-health.sh — five-second "is the cluster OK?"
set -euo pipefail
B="${1:-broker1:9092}"
K=/opt/kafka/current/bin

echo "== Under-replicated =="
$K/kafka-topics.sh --bootstrap-server "$B" --describe --under-replicated-partitions

echo "== Unavailable (offline) =="
$K/kafka-topics.sh --bootstrap-server "$B" --describe --unavailable-partitions

echo "== Consumer lag (critical groups) =="
for g in etl-consumer payroll-sink; do
  $K/kafka-consumer-groups.sh --bootstrap-server "$B" --describe --group "$g" \
    | awk 'NR==1 || $6>0'        # header + any partition with lag
done

echo "== Disk =="
df -h /data/kafka-logs | tail -1

echo "== Cert expiry =="
echo | openssl s_client -connect "${B%:*}:9093" 2>/dev/null \
  | openssl x509 -noout -enddate 2>/dev/null || echo "  (no TLS listener / check manually)"
```

Tooling: Bash/Python for routine ops (health, lag reports, cert-expiry, topic provisioning with standard defaults); Ansible for broker config + OS tuning; Terraform for declarative provisioning where the platform allows.

---

## 13. Quick Reference (Command Cheat Sheet)

```bash
B=broker1:9092

# --- TOPICS ---
kafka-topics.sh --bootstrap-server $B --create --topic T --partitions 6 --replication-factor 3
kafka-topics.sh --bootstrap-server $B --describe --topic T
kafka-topics.sh --bootstrap-server $B --list
kafka-topics.sh --bootstrap-server $B --alter --topic T --partitions 12     # one-way!
kafka-topics.sh --bootstrap-server $B --delete --topic T

# --- HEALTH ---
kafka-topics.sh --bootstrap-server $B --describe --under-replicated-partitions   # want: empty
kafka-topics.sh --bootstrap-server $B --describe --unavailable-partitions        # want: empty

# --- CONFIGS (live) ---
kafka-configs.sh --bootstrap-server $B --alter --entity-type topics --entity-name T --add-config retention.ms=604800000
kafka-configs.sh --bootstrap-server $B --describe --entity-type topics --entity-name T

# --- CONSUMER GROUPS ---
kafka-consumer-groups.sh --bootstrap-server $B --list
kafka-consumer-groups.sh --bootstrap-server $B --describe --group G
kafka-consumer-groups.sh --bootstrap-server $B --group G --reset-offsets --to-earliest --topic T --dry-run

# --- ACLs ---
kafka-acls.sh --bootstrap-server $B --add --allow-principal User:svc --operation Read --topic T --group G
kafka-acls.sh --bootstrap-server $B --list

# --- REASSIGN / LEADER ---
kafka-reassign-partitions.sh --bootstrap-server $B --reassignment-json-file r.json --execute --throttle 50000000
kafka-leader-election.sh --bootstrap-server $B --election-type PREFERRED --all-topic-partitions

# --- KRaft STORAGE ---
kafka-storage.sh random-uuid
kafka-storage.sh format -t <UUID> -c config/kraft/server.properties
```

**Facts to have at instant recall:**

| Concept | Key fact |
|---|---|
| Durability vs latency knob | producer `acks` (0 / 1 / all) |
| Survive 1 broker loss, no data loss | RF=3, min.insync.replicas=2, acks=all |
| Must be exactly 1 cluster-wide | active controller count |
| Must be 0 | under-replicated + offline partitions |
| Reduce partitions? | No — increase only; breaks key ordering |
| Offsets stored in | internal `__consumer_offsets` topic |
| Cleanup policies | `delete` (retention), `compact` (latest-per-key) |
| Cross-cluster replication | MirrorMaker 2 |
| Replaced ZooKeeper | KRaft (internal Raft metadata quorum) |
| Three security layers | TLS (encrypt), SASL/mTLS (authn), ACLs (authz) |
| `NOT_ENOUGH_REPLICAS` cause | ISR < min.insync.replicas under acks=all |
| Safe patch method | rolling upgrade, ISR-gated, protocol pinned |
| Kafka "backup" really means | replication + DR cluster + config/metadata/cert capture |
| Broker GC | G1GC, modest heap, rest of RAM to page cache |

---

## 14. Questions to Ask Them

- Are the clusters on ZooKeeper or KRaft today, and is a migration planned?
- What's the current monitoring/alerting stack inside the enclave?
- How is DR handled — single site, or cross-site replication?
- What does the change-management cadence look like for production Kafka work?
- How mature are the existing runbooks/SOPs — building from scratch or maintaining?
- Who owns certificates and Kerberos principals — the Kafka team or central security?
- What's the biggest recurring Kafka incident the team faces right now?

---

**Final framing:** you're an operator who understands Kafka deeply enough to run it without a safety net, secures it to federal standards, and documents everything for audit. Lead with operations and security, translate your cloud experience into on-prem competence, and show you're comfortable inside change control. That's the hire.
