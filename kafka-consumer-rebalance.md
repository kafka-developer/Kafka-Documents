# Diagnose and Remediate Kafka Consumer Rebalance Scenarios

> Consumer rebalance happens when Kafka changes partition ownership among consumers in the same **Consumer Group**.

### This usually causes:
- Temporary stop in message consumption
- Duplicate processing
- Increased lag
- Consumer downtime

The goal is to identify **why rebalance happened** and apply the correct fix.

---

## Table of Contents

1. [What is Consumer Rebalance](#1-what-is-consumer-rebalance)
2. [Common Rebalance Scenarios](#2-common-rebalance-scenarios)
3. [Diagnosis Method](#3-diagnosis-method)
4. [Remediation Steps](#4-remediation-steps)
5. [Important Consumer Properties](#5-important-consumer-properties)
6. [Quick Interview Summary](#6-quick-interview-summary)

---

## 1. What is Consumer Rebalance

A rebalance happens when:

- A new consumer joins the group
- A consumer leaves the group
- Consumer crashes
- Session timeout occurs
- Poll timeout occurs
- Topic partitions increase
- Broker leadership changes
- Coordinator failure happens

Kafka then reassigns partitions across consumers.

**During this time:**

- Consumers stop processing
- Partitions are revoked
- New assignment happens
- Consumption resumes

This creates lag and performance issues.

---

## 2. Common Rebalance Scenarios

| Scenario | Root Cause | Impact | Fix |
|---|---|---|---|
| Frequent Consumer Restarts | Application crash / unstable pod | Continuous rebalance | Fix app stability |
| `max.poll.interval.ms` Exceeded | Consumer processing too slow | Kafka removes consumer | Reduce processing time |
| `session.timeout.ms` Exceeded | Heartbeats missed | Consumer marked dead | Tune heartbeat/session |
| New Consumer Added | Scale up event | Partition reassignment | Normal behavior |
| Consumer Removed | Scale down / crash | Reassignment triggered | Normal behavior |
| Large Batch Processing | Poll not called fast enough | Poll timeout rebalance | Reduce batch size |
| Long GC Pause | JVM freeze | Heartbeats stop | JVM tuning |
| Network Issues | Consumer cannot reach broker | Session timeout | Fix network |
| Topic Partition Increase | Admin changed partitions | Rebalance triggered | Expected behavior |
| Coordinator Broker Failure | Group coordinator changed | Temporary rebalance | Broker recovery |

---

## 3. Diagnosis Method

### Step 1 — Check Consumer Logs

Look for:

```text
Revoking previously assigned partitions
Successfully joined group
Attempt to heartbeat failed
Member removed from group
Rebalance in progress
```

These confirm rebalance activity.

---

### Step 2 — Check Consumer Lag

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-group \
  --describe
```

Look for:

- Sudden lag spikes
- Consumers disappearing
- Frequent ownership changes

---

### Step 3 — Check Broker Logs

Look for:

```text
Preparing to rebalance group
Stabilized group
Member failed
Heartbeat session expired
```

This identifies the reason.

---

### Step 4 — Check JVM / Pod / Infra

Check:

- OOM kill
- CPU spikes
- GC pauses
- Kubernetes restarts
- Node failures

> Often rebalance is infra-related.

---

## 4. Remediation Steps

### Scenario 1 — Slow Processing

**Problem**

Consumer takes too long to process records.

Kafka expects:

```
poll() → process → poll()
```

If processing exceeds `max.poll.interval.ms`, Kafka removes the consumer.

**Fix**

- Reduce processing time
- Reduce `max.poll.records`
- Move heavy logic to async workers
- Increase `max.poll.interval.ms`

---

### Scenario 2 — Heartbeat Failure

**Problem**

Consumer misses heartbeat. Kafka assumes consumer is dead.

**Fix**

Tune:

```properties
session.timeout.ms
heartbeat.interval.ms
```

Rule:

```
heartbeat.interval.ms < session.timeout.ms
```

> Best practice: `heartbeat.interval.ms` = 1/3 of `session.timeout.ms`

---

### Scenario 3 — Frequent Deployments

**Problem**

Kubernetes rolling deployments trigger constant rebalances.

**Fix**

Use:

- Graceful shutdown
- Static membership (`group.instance.id`)
- Cooperative Sticky Assignor (`CooperativeStickyAssignor`)

This reduces full rebalances.

---

### Scenario 4 — Large Poll Batch

**Problem**

Too many records fetched → slow processing → poll timeout.

**Fix**

Reduce:

```properties
max.poll.records
max.partition.fetch.bytes
fetch.max.bytes
```

> Smaller batches = fewer poll timeout issues.

---

### Scenario 5 — Network Instability

**Problem**

Consumer loses broker connectivity.

**Fix**

Check:

- DNS resolution
- Firewall rules
- Load balancer health
- Broker reachability
- TLS/certificate issues

---

## 5. Important Consumer Properties

| Property | Purpose |
|---|---|
| `session.timeout.ms` | How long Kafka waits before declaring consumer dead |
| `heartbeat.interval.ms` | How often consumer sends heartbeat |
| `max.poll.interval.ms` | Maximum allowed time between poll calls |
| `max.poll.records` | Number of records returned per poll |
| `fetch.min.bytes` | Minimum bytes broker waits before sending fetch |
| `fetch.max.wait.ms` | Max wait time for fetch response |
| `partition.assignment.strategy` | Controls partition assignment behavior |
| `group.instance.id` | Enables static membership |

---

## 6. Quick Interview Summary

> **"Consumer rebalance happens when consumer group membership changes or Kafka thinks a consumer is unhealthy.**
>
> I first check consumer logs for rebalance messages, then consumer lag using `kafka-consumer-groups`, then broker logs for session timeout or coordinator events.
>
> Most common reasons are slow processing causing `max.poll.interval.ms` timeout, heartbeat failures causing session timeout, application restarts, long GC pauses, or frequent deployments.
>
> To fix this, I tune heartbeat/session settings, reduce batch size, optimize processing time, use static membership, and cooperative sticky assignor to reduce unnecessary full rebalances."

---

*This is a strong senior Kafka engineer answer.*
