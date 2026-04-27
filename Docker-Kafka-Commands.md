# Docker Kafka Cluster — Command Summary

## 1. Start the cluster

```bash
docker compose up -d

```

Starts Kafka broker, Schema Registry, ksqlDB Server, and ksqlDB CLI in background.

----------

## 2. Stop the cluster

```bash
docker compose down

```

Stops all containers.

----------

## 3. Stop and remove volumes

```bash
docker compose down -v

```

Deletes containers + persistent volumes (fresh restart).

----------

## 4. View running containers

```bash
docker ps

```

Shows all running Docker containers.

Cleaner view:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"

```

Best clean view:

```bash
docker ps --format "table {{.Names}}"

```

----------

## 5. View all containers (including stopped)

```bash
docker ps -a

```

----------

## 6. Enter Kafka broker container

```bash
docker exec -it broker bash

```

Access Kafka broker shell.

----------

## 7. Enter Schema Registry container

```bash
docker exec -it schema-registry bash

```

Access Schema Registry shell.

----------

## 8. Enter ksqlDB CLI

```bash
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088

```

Access ksqlDB interactive shell.

----------

## 9. View Kafka CLI tools

Inside broker container:

```bash
cd /opt/kafka/bin
ls

```

Shows:

-   kafka-topics
    
-   kafka-console-producer
    
-   kafka-console-consumer
    
-   kafka-configs
    
-   kafka-consumer-groups
    
-   kafka-acls
    
-   kafka-storage
    
-   kafka-metadata-quorum
    
-   etc.
    

----------

## 10. Kafka command help

```bash
kafka-topics --help

```

Shows command usage.

----------

## 11. List Kafka topics

```bash
docker exec -it broker kafka-topics \
--bootstrap-server broker:29092 \
--list

```

Shows all topics.

----------

## 12. Create topic

```bash
docker exec -it broker kafka-topics \
--bootstrap-server broker:29092 \
--create \
--topic orders \
--partitions 3 \
--replication-factor 1

```

Creates topic.

----------

## 13. Describe topic

```bash
docker exec -it broker kafka-topics \
--bootstrap-server broker:29092 \
--describe \
--topic orders

```

Shows topic details.

----------

## 14. Produce messages

```bash
docker exec -it broker kafka-console-producer \
--bootstrap-server broker:29092 \
--topic orders

```

Send messages manually.

----------

## 15. Consume messages

```bash
docker exec -it broker kafka-console-consumer \
--bootstrap-server broker:29092 \
--topic orders \
--from-beginning

```

Reads messages.

----------

## 16. Check Schema Registry

```bash
curl http://localhost:8081/subjects

```

Verifies Schema Registry is working.

Expected:

```json
[]

```

----------

## 17. ksqlDB commands

Inside ksqlDB CLI:

```sql
SHOW TOPICS;
SHOW STREAMS;
SHOW TABLES;

```

View Kafka topics, streams, and tables.

----------

# Quick Memory Rule

## Docker level

```bash
docker ps
docker exec
docker compose

```

## Kafka level

```bash
kafka-topics
kafka-console-producer
kafka-console-consumer

```

## SQL level

```bash
ksql

```

## Schema level

```bash
curl localhost:8081

```

This is enough for strong interview discussion 🔥
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ0MzcwNDE3OF19
-->