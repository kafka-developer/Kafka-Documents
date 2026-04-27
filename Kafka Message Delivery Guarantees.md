## Kafka Message Delivery Guarantees — Diagnose + Remediation

### 1. Messages Missing

### Why it happens

Producer sends message, but broker does not safely store it before acknowledging.

Common bad configs:

```properties
acks=0

```

or

```properties
acks=1

```

Also happens when:

```properties
replication.factor=1

```

### Example Problem

```text
Producer sends message
Leader broker receives it
Broker crashes before follower replication
Message is lost

```

----------

### Fix

Use strong durability settings:

```properties
acks=all
enable.idempotence=true
retries=Integer.MAX_VALUE

```

Topic settings:

```properties
replication.factor=3
min.insync.replicas=2

```

### Simple meaning

```text
Producer waits until leader + followers confirm write
Only then success is returned

```

----------

## 2. Duplicate Messages

### Why it happens

Producer sends message, broker stores it, but acknowledgment is lost.

Producer thinks message failed and retries.

Same message gets written again.

----------

### Example Problem

```text
Producer sends Order #101
Broker writes Order #101
Ack is lost
Producer retries
Broker writes Order #101 again
Duplicate created

```

----------

### Fix

Use idempotent producer:

```properties
enable.idempotence=true
acks=all
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5

```

### Simple meaning

Kafka adds sequence numbers.

If retry happens, broker knows:

```text
This is same message → do not write twice

```

----------

## 3. Out-of-Order Messages

### Why it happens

Producer sends many requests at same time.

Retry causes later message to arrive first.

----------

### Example Problem

```text
Message A sent
Message B sent

B succeeds first
A retries later

Final order becomes:

B
A

```

Wrong order.

----------

### Fix

```properties
enable.idempotence=true
max.in.flight.requests.per.connection=5

```

Also use same message key:

```text
customer_id
order_id
account_id

```

### Simple meaning

Kafka guarantees order only inside one partition.

Same key = same partition = correct order.

----------

## 4. Message Lost Because Consumer Committed Early

### Why it happens

Consumer commits offset before actual processing.

----------

### Bad Flow

```text
poll message
commit offset
process message
app crashes

```

Kafka thinks message is done.

But business processing never happened.

Message lost forever.

----------

### Bad Config

```properties
enable.auto.commit=true

```

----------

### Fix

```properties
enable.auto.commit=false

```

Correct flow:

```text
poll message
process message
save to DB
commit offset

```

Code example:

```java
consumer.commitSync();

```

Only after success.

----------

## 5. Duplicate Processing by Consumer

### Why it happens

Consumer processes message but crashes before committing offset.

----------

### Example

```text
poll message
process payment
app crashes
offset not committed
consumer restarts
same payment processed again

```

Duplicate business action.

----------

### Fix

Do NOT try to stop replay.

Replay is normal.

Instead make consumer idempotent:

```text
Use event_id
Check if already processed
Use unique constraint
Use UPSERT instead of INSERT

```

Example:

```sql
INSERT INTO payments (event_id, amount)
VALUES ('evt-101', 500)
ON CONFLICT (event_id)
DO NOTHING;

```

Now duplicate replay is safe.

----------

## 6. Kafka Topic → Kafka Topic Duplicate

### Why it happens

Application reads from Topic A and writes to Topic B.

Crash happens in between.

----------

### Example Problem

```text
Read from payments
Write to audit-topic
Crash before offset commit
Restart
Same event written again

```

Duplicate in output topic.

----------

### Fix → Use Transactions

Producer:

```properties
enable.idempotence=true
acks=all
transactional.id=payment-service-1

```

Consumer:

```properties
isolation.level=read_committed
enable.auto.commit=false

```

### Simple meaning

Either both happen:

```text
write output + commit offset

```

or neither happens.

This gives Exactly Once Semantics.

----------

## 7. Long Processing Causes Rebalance + Replay

### Why it happens

Consumer takes too long to process.

Kafka thinks consumer is dead.

Partitions move to another consumer.

Messages replay.

----------

### Example

```text
Consumer polls 5000 records
Processing takes 10 minutes
max.poll.interval.ms expires
Kafka removes consumer
rebalance happens
messages replay

```

----------

### Fix

```properties
max.poll.records=100
max.poll.interval.ms=300000
session.timeout.ms=45000
heartbeat.interval.ms=15000

```

### Simple meaning

Take smaller batches.

Poll faster.

Avoid unnecessary rebalances.

----------

# Final Interview Answer

I diagnose delivery guarantee issues by checking producer, broker, and consumer separately.

Producer side:

```properties
acks
enable.idempotence
retries
max.in.flight.requests.per.connection

```

Broker side:

```properties
replication.factor
min.insync.replicas
unclean.leader.election.enable

```

Consumer side:

```properties
enable.auto.commit
commitSync()
idempotent processing

```

For strongest delivery:

```properties
Producer:
acks=all
enable.idempotence=true

Topic:
replication.factor=3
min.insync.replicas=2

Consumer:
enable.auto.commit=false
commit after success

```

For Kafka-to-Kafka pipelines:

```properties
transactional.id
isolation.level=read_committed

```

That gives:

```text
At most once  → commit first
At least once → process first
Exactly once  → idempotence + transactions

```


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTUxNzEwNzIyNF19
-->