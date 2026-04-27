
# Kafka Delivery Semantics

Delivery semantics in Apache Apache Kafka define **how many times a message is delivered between Producer → Broker → Consumer** and what guarantee Kafka gives if failures happen (broker crash, network issue, consumer restart, retry, etc.).

This is one of the most important interview topics because it directly impacts **data loss vs duplicate data**.

There are **3 delivery semantics**:

----------

# Table of Contents

1.  At Most Once
    
2.  At Least Once
    
3.  Exactly Once
    
4.  Simple Workflow Comparison
    
5.  Producer + Consumer configs for each
    
6.  Interview-ready summary
    

----------

# 1. At Most Once

## Meaning

Message is delivered **0 or 1 time**

→ It may be lost  
→ But it will never be duplicated

This prioritizes **speed over reliability**

----------

## Workflow

Producer sends message → Broker receives → Consumer reads → Consumer commits offset **before processing**

If consumer crashes after offset commit but before processing:

→ Message is gone forever

Because Kafka thinks it was already consumed

----------

## Example

Log monitoring system

If one log line is missed, not a big issue

Speed matters more than perfection

----------

## Risk

### Data Loss = YES

### Duplicate Data = NO

----------

# 2. At Least Once

## Meaning

Message is delivered **1 or more times**

→ No data loss  
→ But duplicates may happen

This prioritizes **reliability over duplicates**

This is the most common real-world setup

----------

## Workflow

Producer sends message → Broker stores → Consumer reads → Consumer processes → Consumer commits offset **after processing**

If consumer crashes after processing but before offset commit:

→ Same message is read again

Result:

→ Duplicate processing

----------

## Example

Bank notification email

Better to send 2 emails than lose 1 payment alert

----------

## Risk

### Data Loss = NO

### Duplicate Data = YES

----------

# 3. Exactly Once

## Meaning

Message is delivered **exactly 1 time**

→ No data loss  
→ No duplicates

This is hardest and most expensive

Used where duplicates are unacceptable

----------

## Workflow

Producer uses:

-   Idempotence
    
-   Transactions
    

Consumer reads only committed data

Offsets and writes happen atomically

Even if retry happens:

→ duplicate write is prevented

----------

## Example

Bank money transfer

If ₹10,000 is transferred twice

→ disaster

Must happen exactly once

----------

## Risk

### Data Loss = NO

### Duplicate Data = NO

----------

# 4. Simple Workflow Comparison

0 \leq \text{deliveries} \leq 1

### At Most Once

Commit first → Process later

Fast but risky

----------

\text{deliveries} \geq 1

### At Least Once

Process first → Commit later

Safe but duplicates possible

----------

\text{deliveries} = 1

### Exactly Once

Transactional processing

Safe + no duplicates

----------

# 5. Producer + Consumer Configs

# At Most Once

## Consumer

```properties
enable.auto.commit=true
auto.commit.interval.ms=1000

```

Offsets committed automatically

May lose data

----------

# At Least Once

## Consumer

```properties
enable.auto.commit=false

```

Manual commit after successful processing

Safer

----------

# Exactly Once

## Producer

```properties
acks=all
enable.idempotence=true
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5
transactional.id=txn-1

```

## Consumer

```properties
isolation.level=read_committed
enable.auto.commit=false

```

This enables transactional guarantees

----------

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNzAzMzk3NTNdfQ==
-->