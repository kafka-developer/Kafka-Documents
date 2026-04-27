# Diagnose and Remediation of Message Delivery in Kafka

This is primarily about understanding **Kafka Delivery Semantics** and identifying why messages are being lost, duplicated, skipped, delayed, or delivered out of order.

The goal is to diagnose the root cause and apply the correct remediation using Producer, Broker, and Consumer configurations.

----------

# What They Are Testing

Interviewers want to know if you can identify:

-   Why messages get lost
    
-   Why duplicate messages happen
    
-   Why consumers skip records
    
-   Why ordering breaks
    
-   Why retries create duplicates
    
-   Why producer says success but consumer never gets data
    
-   How acknowledgments affect durability
    
-   How offset commits affect reliability
    
-   How to guarantee safe message delivery
    

This directly maps to Kafka Delivery Semantics.

----------

# Kafka Delivery Semantics

## 1. At Most Once

### Meaning

Message is delivered **zero or one time**

No duplicates, but message loss is possible.

If failure happens, Kafka will not retry.

### Example

Consumer commits offset first and then crashes before processing.

Kafka thinks message is already consumed.

That message is permanently lost.

### Common Cause

-   Auto commit enabled
    
-   Offset committed too early
    
-   `acks=0` or weak producer acknowledgment
    

### Remediation

-   Disable auto commit
    
-   Use manual offset commit after successful processing
    
-   Use stronger producer acknowledgment like:
    

```properties
acks=all

```

----------

## 2. At Least Once

### Meaning

Message is delivered **one or more times**

No message loss, but duplicates can happen.

Kafka retries until success.

### Example

Producer sends message → broker stores it → acknowledgment is delayed → producer retries → duplicate message created.

### Common Cause

-   Producer retries
    
-   Consumer crashes before committing offset
    
-   Network timeout causing resend
    

### Remediation

-   Use idempotent producer
    

```properties
enable.idempotence=true

```

-   Use consumer deduplication logic
    
-   Proper retry strategy
    

----------

## 3. Exactly Once

### Meaning

Message is delivered **exactly one time**

No duplicates and no message loss.

This is the strongest guarantee.

### Example

Payment processing system where duplicate transaction is unacceptable.

### Common Cause

Requires proper configuration across Producer + Broker + Consumer.

Not enabled automatically.

### Remediation

Use:

```properties
enable.idempotence=true
acks=all
max.in.flight.requests.per.connection=5
transactional.id=payment-service

```

Also use transactional consumer logic where needed.

----------

# Diagnosis Scenarios

----------

# Scenario 1 — Duplicate Messages

## Problem

Same message appears multiple times in consumer.

## Root Cause

Usually happens in **At Least Once Delivery**

Producer retries or consumer reprocesses same record.

### Common Reasons

-   Producer retry after timeout
    
-   Consumer crash before offset commit
    
-   Network issue causing resend
    
-   Retry topic reprocessing
    

## Remediation

### Producer

```properties
enable.idempotence=true
acks=all
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5

```

### Consumer

-   Commit offset only after successful processing
    
-   Use idempotent business logic
    
-   Use unique transaction IDs
    

----------

# Scenario 2 — Message Loss

## Problem

Producer says message sent, but consumer never receives it.

## Root Cause

Usually happens in **At Most Once Delivery**

Weak durability settings.

### Common Reasons

-   `acks=0`
    
-   `acks=1` and leader crashes before replication
    
-   Unclean leader election
    
-   Offset committed before processing
    

## Remediation

### Producer

```properties
acks=all

```

### Broker

```properties
min.insync.replicas=2
unclean.leader.election.enable=false

```

### Consumer

-   Manual offset commit only after processing
    

----------

# Scenario 3 — Consumer Skips Records

## Problem

Consumer restarts and some records are missing.

## Root Cause

Offset commit problem.

### Common Reasons

-   Auto commit commits too early
    
-   Consumer crashes after commit but before processing
    

## Remediation

```properties
enable.auto.commit=false

```

Use:

-   Manual commit
    
-   Commit only after successful processing
    

----------

# Scenario 4 — Ordering Problems

## Problem

Messages arrive out of order.

## Root Cause

Parallel retries or multiple partitions.

### Common Reasons

-   High `max.in.flight.requests.per.connection`
    
-   Multiple partitions
    
-   Multiple producers
    

## Remediation

```properties
max.in.flight.requests.per.connection=1

```

or safely:

```properties
enable.idempotence=true
max.in.flight.requests.per.connection=5

```

Also ensure ordering-sensitive messages use same partition key.

----------

# Scenario 5 — Producer Success but Data Missing

## Problem

Producer logs success but data disappears.

## Root Cause

Leader acknowledged before full replication.

### Common Reasons

-   `acks=1`
    
-   Broker leader crash
    
-   ISR too small
    

## Remediation

```properties
acks=all
min.insync.replicas=2
replication.factor=3

```

----------

# Scenario 6 — Consumer Lag Causing Delivery Delay

## Problem

Messages are not lost but delayed heavily.

## Root Cause

Consumer cannot keep up.

### Common Reasons

-   Slow processing
    
-   Large messages
    
-   Small fetch size
    
-   Low partitions
    
-   Poll configuration issues
    

## Remediation

-   Increase partitions
    
-   Optimize consumer fetch configs
    
-   Scale consumer group
    
-   Improve processing speed
    

Example:

```properties
fetch.min.bytes=1048576
max.poll.records=500
max.partition.fetch.bytes=1048576

```

----------

# Important Producer Properties for Message Delivery Reliability

| Property | Purpose |
|----------|---------|
| acks | Controls durability guarantee and decides when producer considers message successfully written |
| enable.idempotence | Prevents duplicate message writes during retries |
| retries | Retries failed sends to avoid message loss |
| max.in.flight.requests.per.connection | Controls ordering during retries and prevents out-of-order delivery |
| compression.type | Improves throughput and reduces network usage |
| linger.ms | Waits briefly to create bigger batches for better throughput |
| batch.size | Controls how much data is grouped before sending |

# Important Consumer Properties for Reliable Message Consumption

| Property | Purpose |
|----------|---------|
| enable.auto.commit | Controls whether offsets are committed automatically or manually |
| max.poll.records | Limits how many records consumer reads per poll |
| fetch.min.bytes | Minimum amount of data broker collects before sending to consumer |
| fetch.max.wait.ms | Maximum time broker waits before sending fetch response |
| max.partition.fetch.bytes | Maximum data fetched per partition in one request |
| auto.offset.reset | Decides where consumer starts if no committed offset exists |
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwNDQ1MDU0MTVdfQ==
-->