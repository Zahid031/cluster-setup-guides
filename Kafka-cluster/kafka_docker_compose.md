# Kafka Cluster with Docker Compose

A simple 3-node Kafka cluster setup using Docker Compose with KRaft mode (no ZooKeeper required). Includes both Confluent and Apache Kafka options.

## Prerequisites

- Docker and Docker Compose installed
- Update `192.168.169.XX` or `10.70.34.181` with your actual IP address
- Basic understanding of Kafka and containerization

## Quick Start

1. Create a directory for your Kafka cluster:
```bash
mkdir kafka-cluster
cd kafka-cluster
```

2. Save one of the `docker-compose.yml` files below (Confluent or Apache)
3. Replace `192.168.169.XX` or `10.70.34.181` with your actual server IP
4. Create data directories:
```bash
mkdir -p data/kafka1 data/kafka2 data/kafka3
```

5. Start the cluster:
```bash
docker-compose up -d
```

6. Access Kafka UI at `http://YOUR_IP:8080`

## Option 1: docker-compose.yml (Confluent)

```yaml
version: '3.8'

services:
  kafka1:
    image: confluentinc/cp-kafka:7.9.4
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      CLUSTER_ID: "83fuOzneRyCWp1SAjgV0HA"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka1:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka1:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:29092,EXTERNAL://192.168.169.XX:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka1:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka2:
    image: confluentinc/cp-kafka:7.9.4
    container_name: kafka2
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 2
      CLUSTER_ID: "83fuOzneRyCWp1SAjgV0HA"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka2:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka2:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka2:29092,EXTERNAL://192.168.169.XX:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka2:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka3:
    image: confluentinc/cp-kafka:7.9.4
    container_name: kafka3
    ports:
      - "9096:9092"
    environment:
      KAFKA_NODE_ID: 3
      CLUSTER_ID: "83fuOzneRyCWp1SAjgV0HA"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka3:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka3:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka3:29092,EXTERNAL://192.168.169.XX:9096
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka3:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka-ui:
    image: redpandadata/console:v3.2.2
    container_name: redpanda-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_BROKERS: 192.168.169.XX:9092,192.168.169.XX:9094,192.168.169.XX:9096
      SERVER_LISTEN_PORT: 8080
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - kafka-net

networks:
  kafka-net:
    driver: bridge
```

## Option 2: docker-compose.yml (Apache Kafka)

```yaml
services:
  kafka1:
    image: apache/kafka:4.0.1
    container_name: kafka1
    restart: always
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_CLUSTER_ID: "QJRAqIIQQziNKl2McGkGYg"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka1:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka1:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:29092,EXTERNAL://192.168.169.23:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_METADATA_LOG_DIR: /var/lib/kafka/metadata
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka1/data:/var/lib/kafka/data
      - ./data/kafka1/metadata:/var/lib/kafka/metadata
    networks:
      - kafka-net

  kafka2:
    image: apache/kafka:4.0.1
    container_name: kafka2
    restart: always
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_CLUSTER_ID: "QJRAqIIQQziNKl2McGkGYg"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka2:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka2:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka2:29092,EXTERNAL://192.168.169.23:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_METADATA_LOG_DIR: /var/lib/kafka/metadata
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka2/data:/var/lib/kafka/data
      - ./data/kafka2/metadata:/var/lib/kafka/metadata
    networks:
      - kafka-net

  kafka3:
    image: apache/kafka:4.0.1
    container_name: kafka3
    restart: always
    ports:
      - "9096:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_CLUSTER_ID: "QJRAqIIQQziNKl2McGkGYg"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka3:29092,EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka3:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka3:29092,EXTERNAL://192.168.169.23:9096
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_METADATA_LOG_DIR: /var/lib/kafka/metadata
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
    volumes:
      - ./data/kafka3/data:/var/lib/kafka/data
      - ./data/kafka3/metadata:/var/lib/kafka/metadata
    networks:
      - kafka-net

  kafka-ui:
    image: redpandadata/console:v3.2.2
    container_name: redpanda-ui
    restart: always
    ports:
      - "8080:8080"
    environment:
      KAFKA_BROKERS: 192.168.169.23:9092,192.168.169.23:9094,192.168.169.23:9096
      SERVER_LISTEN_PORT: 8080
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - kafka-net

networks:
  kafka-net:
    driver: bridge
```

## Directory Creation
```bash
mkdir -p ./data/kafka1/data ./data/kafka1/metadata \
         ./data/kafka2/data ./data/kafka2/metadata \
         ./data/kafka3/data ./data/kafka3/metadata

chmod 777 -R data
```
## Common Commands

Stop the cluster:
```bash
docker-compose down
```

View logs:
```bash
docker-compose logs -f kafka1
```

Create a topic:
```bash
docker exec kafka1 kafka-topics --bootstrap-server kafka1:9092 --create --topic my-topic --partitions 3 --replication-factor 3
```

List topics:
```bash
docker exec kafka1 kafka-topics --bootstrap-server kafka1:9092 --list
```

## Important Notes

- Replace `192.168.169.XX` in Option 1 with your actual server IP
- Replace `10.70.34.181` in Option 2 with your actual server IP
- Use KRaft mode (no ZooKeeper needed)
- All three nodes act as both brokers and controllers
- Data persists in `./data` directories
- Replication factor is 3 for high availability





