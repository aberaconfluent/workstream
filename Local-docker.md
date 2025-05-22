# Apache Flink and Kafka Setup Guide

This document outlines steps to set up Apache Kafka and Flink, run example jobs, and interact with Flink SQL. It also includes instructions for a Docker-based setup with Confluent Platform.

---

## Tarball with Apache Kafka (Optional, Prerequisites)

```bash
wget [https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz](https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz)
mv ~/Downloads/kafka_2.13-3.7.0.tgz .
tar xf kafka_2.13-3.7.0.tgz
cd kafka_2.13-3.7.0

# Generate a Kafka Cluster ID for KRaft mode
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

# Format the storage directories for KRaft
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

---

## Flink Installation

```bash
wget https://dlcdn.apache.org/flink/flink-1.19.0/flink-1.19.0-bin-scala_2.12.tgz
tar xf flink-1.19.0-bin-scala_2.12.tgz
cd flink-1.19.0

# Start Flink locally
./bin/start-cluster.sh
```

**Validate UI is running:** `http://localhost:8081/#/overview`
**Validate a TaskManager is connected**

---

## Run Example

```bash
./bin/flink run ./examples/streaming/TopSpeedWindowing.jar
```

To list running jobs:

```bash
./bin/flink list
```

To cancel a job:

```bash
./bin/flink cancel <id>
```

**Explore Flink UI further:** `http://localhost:8081/#/overview`

---

## Use Flink SQL Shell (Local Setup)

First, create the Kafka input topic:

```bash
./kafka_2.13-3.7.0/bin/kafka-topics.sh --create --topic flink-input --bootstrap-server localhost:9092
```

Navigate to Flink directory and prepare SQL client:

```bash
cd flink-1.19.0
mkdir sql-lib
cd sql-lib
wget [https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.1.0-1.18/flink-sql-connector-kafka-3.1.0-1.18.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.1.0-1.18/flink-sql-connector-kafka-3.1.0-1.18.jar)
# Optional: If you need Avro with Confluent Schema Registry
# wget [https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-avro-confluent-registry/1.19.0/flink-sql-avro-confluent-registry-1.19.0.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-avro-confluent-registry/1.19.0/flink-sql-avro-confluent-registry-1.19.0.jar)
```

Start the SQL client:

```bash
./flink-1.19.0/bin/sql-client.sh --library ./flink-1.19.0/sql-lib
```

Inside the SQL client, create tables and run queries:

```sql
> CREATE TABLE flinkInput (
>   `raw` STRING,
>   `ts` TIMESTAMP(3) METADATA FROM 'timestamp'
> ) WITH (
>   'connector' = 'kafka',
>   'topic' = 'flink-input',
>   'properties.bootstrap.servers' = 'localhost:9092',
>   'properties.group.id' = 'testGroup',
>   'scan.startup.mode' = 'earliest-offset',
>   'format' = 'raw'
> );
>
> SELECT * FROM flinkInput;
```

Create an aggregate table:

```sql
> CREATE TABLE msgCount (
>   `count` BIGINT NOT NULL
> ) WITH (
>   'connector' = 'kafka',
>   'topic' = 'message-count',
>   'properties.bootstrap.servers' = 'localhost:9092',
>   'properties.group.id' = 'testGroup',
>   'scan.startup.mode' = 'earliest-offset',
>   'format' = 'debezium-json'
> );
>
> INSERT INTO msgCount SELECT COUNT(*) as `count` FROM flinkInput;
>
> DROP TABLE msgCount;
```

---

## Separate Terminal: Produce Data to Kafka (for Local Setup)

Open a new terminal and run:

```bash
> ./kafka_2.13-3.7.0/bin/kafka-console-producer.sh --topic flink-input --bootstrap-server localhost:9092
```

Create `message-count` topic and consume from it:

```bash
> ./kafka_2.13-3.7.0/bin/kafka-topics.sh --create --topic message-count --bootstrap-server localhost:9092
> ./kafka_2.13-3.7.0/bin/kafka-console-consumer.sh --topic message-count --bootstrap-server localhost:9092
```

---

## Docker with Confluent Platform

This setup is based on the repository at: [GitHub - confluentinc/learn-flink-sql-exercises](https://github.com/confluentinc/learn-flink-sql-exercises).

### Create `sql-client` Directory and Dockerfile

```bash
mkdir sql-client
```

Create `Dockerfile` inside the `sql-client` directory:

```dockerfile
# Dockerfile in sql-client/
FROM flink:1.20
RUN wget -P /opt/flink/lib/ [https://repo1.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar](https://repo1.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar); \
    wget -P /opt/flink/lib/ [https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-avro-confluent-registry/1.20.1/flink-sql-avro-confluent-registry-1.20.1.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-avro-confluent-registry/1.20.1/flink-sql-avro-confluent-registry-1.20.1.jar); \
    wget -P /opt/flink/lib/ [https://repo.maven.apache.org/maven2/org/apache/kafka/kafka-clients/3.4.0/kafka-clients-3.4.0.jar](https://repo.maven.apache.org/maven2/org/apache/kafka/kafka-clients/3.4.0/kafka-clients-3.4.0.jar); \
    wget -P /opt/flink/lib/ [https://github.com/knaufk/flink-faker/releases/download/v0.5.2/flink-faker-0.5.2.jar](https://github.com/knaufk/flink-faker/releases/download/v0.5.2/flink-faker-0.5.2.jar);
RUN chown -R flink:flink /opt/flink/lib
```

### Create `docker-compose.yaml`

Create `docker-compose.yaml` in the upper directory (parent of `sql-client`). This concatenates `cp-all-in-one` with Apache's Flink Docker image. Adjust `taskmanager scale` and `taskmanager.numberOfTaskSlots` to tune the number of task managers and slots per.

```yaml
version: '2'
services:
  broker:
    image: confluentinc/cp-kafka:7.6.1
    hostname: broker
    container_name: broker
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@broker:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://broker:29092,CONTROLLER://broker:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
      # Replace CLUSTER_ID with a unique base64 UUID using "bin/kafka-storage.sh random-uuid"
      # See [https://docs.confluent.io/kafka/operations-tools/kafka-tools.html#kafka-storage-sh](https://docs.confluent.io/kafka/operations-tools/kafka-tools.html#kafka-storage-sh)
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.1
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - broker
    ports:
      - "6081:6081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker:29092'
      SCHEMA_REGISTRY_LISTENERS: [http://0.0.0.0:6081](http://0.0.0.0:6081)
  connect:
    image: cnfldemos/cp-server-connect-datagen:0.6.4-7.6.0
    hostname: connect
    container_name: connect
    depends_on:
      - broker
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'broker:29092'
      CONNECT_REST_ADVERTISED_HOST_NAME: connect
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_FLUSH_INTERVAL_MS: 10000
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:6081
      # CLASSPATH required due to CC-2422
      CLASSPATH: /usr/share/java/monitoring-interceptors/monitoring-interceptors-7.6.1.jar
      CONNECT_PRODUCER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor"
      CONNECT_CONSUMER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
      CONNECT_LOG4J_LOGGERS: org.apache.zookeeper=ERROR,org.I0Itec.zkclient=ERROR,org.reflections=ERROR
  control-center:
    image: confluentinc/cp-enterprise-control-center:7.6.1
    hostname: control-center
    container_name: control-center
    depends_on:
      - broker
      - schema-registry
      - connect
    ports:
      - "9021:9021"
    environment:
      CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker:29092'
      CONTROL_CENTER_CONNECT_CONNECT-DEFAULT_CLUSTER: 'connect:8083'
      CONTROL_CENTER_CONNECT_HEALTHCHECK_ENDPOINT: '/connectors'
      CONTROL_CENTER_SCHEMA_REGISTRY_URL: "http://schema-registry:6081"
      CONTROL_CENTER_REPLICATION_FACTOR: 1
      CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
      CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
      CONFLUENT_METRICS_TOPIC_REPLICATION: 1
      PORT: 9021
  rest-proxy:
    image: confluentinc/cp-kafka-rest:7.6.1
    depends_on:
      - broker
      - schema-registry
    ports:
      - 8082:8082
    hostname: rest-proxy
    container_name: rest-proxy
    environment:
      KAFKA_REST_HOST_NAME: rest-proxy
      KAFKA_REST_BOOTSTRAP_SERVERS: 'broker:29092'
      KAFKA_REST_LISTENERS: "[http://0.0.0.0:8082](http://0.0.0.0:8082)"
      KAFKA_REST_SCHEMA_REGISTRY_URL: 'http://schema-registry:6081'
  jobmanager:
    build: sql-client/.
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
  taskmanager:
    build: sql-client/.
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 2 # Adjust this to tune the number of task managers
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2 # Adjust this to tune slots per task manager
  sql-client:
    build: sql-client/.
    command: bin/sql-client.sh
    depends_on:
      - jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        rest.address: jobmanager
```

---

## Run Flink and Kafka (Docker)

```bash
docker-compose up -d
```

**Confluent Control Center is at:** `http://localhost:9021/`
**Apache Flink UI is at:** `http://localhost:8081/`

In Confluent Control Center, create topics called `flink-input` and `message-count`.

---

## Submit A Job (Docker)

Run a shell on the job manager host:

```bash
docker exec -it $(docker ps --filter name=jobmanager --format={{.ID}}) /bin/sh
```

Then:

```bash
./bin/flink run ./examples/streaming/TopSpeedWindowing.jar
```

You can also copy the example jar file locally using `docker cp` and then submit it via the Web UI.
Explore the Flink UI, including the job manager logs.

---

## Manage Jobs (Docker)

From the JobManager shell:

```bash
./bin/flink list
```

You can cancel the job on the web UI or CLI:

```bash
./bin/flink cancel <id>
```

---

## Flink SQL Shell (Docker)

```bash
docker-compose run sql-client
```

Create a table backed by the Kafka topic you created earlier:

```sql
CREATE TABLE flinkInput (
   `raw` STRING,
   `ts` TIMESTAMP(3) METADATA FROM 'timestamp'
) WITH (
   'connector' = 'kafka',
   'topic' = 'flink-input',
   'properties.bootstrap.servers' = 'broker:29092',
   'properties.group.id' = 'testGroup',
   'scan.startup.mode' = 'earliest-offset',
   'format' = 'raw'
);
```

View the data in the table:

```sql
SELECT * FROM flinkInput;
```

while producing data in Confluent Control Center.

Create an aggregate table backed by the `message-count` topic:

```sql
CREATE TABLE msgCount (
  `count` BIGINT NOT NULL
) WITH (
  'connector' = 'kafka',
  'topic' = 'message-count',
  'properties.bootstrap.servers' = 'broker:29092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'debezium-json'
);
```

Followed by:

```sql
INSERT INTO msgCount SELECT COUNT(*) as `count` FROM flinkInput;
```

Continue to add messages to the `flink-input` topic from Confluent Control Center.

```sql
SELECT * FROM msgCount;
```

To get a sense of how Flink is modifying the underlying Kafka topic, view the `message-count` contents in Confluent Control Center.
To match the stream as represented in the Kafka topic, view the results as a changelog in FlinkSQL shell:

```sql
SET 'sql-client.execution.result-mode' = 'changelog';
```sql
SELECT * FROM msgCount;
```
```
