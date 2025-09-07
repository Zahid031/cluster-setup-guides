# Apache Kafka KRaft Cluster Setup Guide

This guide walks you through setting up a 3-node Apache Kafka cluster using KRaft mode (Kafka Raft metadata mode), which eliminates the need for Apache ZooKeeper.

## Prerequisites

- 3 Ubuntu/Debian VMs or servers
- Network connectivity between all nodes
- Sudo access on all machines
- At least 2GB RAM per node (recommended)

## Cluster Architecture

| Node | IP Address | Role | Ports |
|------|------------|------|-------|
| kafka-1 | 192.168.56.51 | Broker + Controller | 9092 (Broker), 9093 (Controller) |
| kafka-2 | 192.168.56.52 | Broker + Controller | 9092 (Broker), 9093 (Controller) |
| kafka-3 | 192.168.56.53 | Broker + Controller | 9092 (Broker), 9093 (Controller) |

## Step 1: Initial System Setup

Run the following commands on **all three nodes**:

### 1.1 Configure Host Names

```bash
# Add host entries for cluster communication
echo "192.168.56.51 kafka-1
192.168.56.52 kafka-2
192.168.56.53 kafka-3" | sudo tee -a /etc/hosts
```

### 1.2 Install Java

```bash
# Update package list and install OpenJDK 21
sudo apt update
sudo apt install openjdk-21-jdk -y

# Verify Java installation
java -version
```

**Expected output:**
```
openjdk version "21.0.x" 2024-xx-xx
OpenJDK Runtime Environment (build 21.0.x+xx-Ubuntu-xxubuntuxx)
OpenJDK 64-Bit Server VM (build 21.0.x+xx-Ubuntu-xxubuntuxx, mixed mode, sharing)
```

## Step 2: Kafka Installation

Run on **all three nodes**:

### 2.1 Create Kafka User and Directories

```bash
# Create dedicated kafka user (system user with no login shell)
sudo useradd -r -m -d /var/lib/kafka -s /usr/sbin/nologin kafka || true

# Create necessary directories
sudo mkdir -p /opt/kafka /var/lib/kafka/{data,meta,logs}
sudo chown -R kafka:kafka /opt/kafka /var/lib/kafka
```

### 2.2 Download and Install Kafka

```bash
# Download Kafka 4.0.0
cd /tmp
curl -LO https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz

# Extract to /opt/kafka
sudo tar -xzf kafka_2.13-4.0.0.tgz -C /opt/kafka --strip-components=1

# Set ownership
sudo chown -R kafka:kafka /opt/kafka

# Create configuration directory
sudo mkdir -p /etc/kafka
sudo chown -R kafka:kafka /etc/kafka
```

## Step 3: Node-Specific Configuration

Create the configuration file on each node with node-specific settings:

### 3.1 Kafka-1 Configuration

```bash
sudo vi /etc/kafka/server.properties
```

**Content for kafka-1:**
```properties
# --- Identity & roles ---
node.id=1
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://kafka-1:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093

# --- Storage (separate data & metadata) ---
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta

# --- Broker defaults ---
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
```

### 3.2 Kafka-2 Configuration

**Content for kafka-2** (change only node.id and advertised.listeners):
```properties
# --- Identity & roles ---
node.id=2
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://kafka-2:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093

# --- Storage (separate data & metadata) ---
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta

# --- Broker defaults ---
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
```

### 3.3 Kafka-3 Configuration

**Content for kafka-3** (change only node.id and advertised.listeners):
```properties
# --- Identity & roles ---
node.id=3
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://kafka-3:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093

# --- Storage (separate data & metadata) ---
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta

# --- Broker defaults ---
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
```

## Step 4: Initialize Cluster Storage

Run on **all three nodes**:

### 4.1 Generate Cluster UUID

```bash
# Generate a random UUID for the cluster
sudo -u kafka /opt/kafka/bin/kafka-storage.sh random-uuid
```

**Example output:**
```
kyLP6L_AThKpnaxwecuTsw
```

⚠️ **Important:** Use the same UUID on all nodes. Copy the generated UUID and use it in the next step.

### 4.2 Format Storage Directories

```bash
# Format the storage using the generated UUID
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t kyLP6L_AThKpnaxwecuTsw \
  -c /etc/kafka/server.properties
```

**Expected output:**
```
Formatting /var/lib/kafka/meta with metadata.version 4.0-IV0
Formatting /var/lib/kafka/data with metadata.version 4.0-IV0
```

## Step 5: Create Systemd Service

Run on **all three nodes**:

### 5.1 Create Service File

```bash
sudo vi /etc/systemd/system/kafka.service
```

**Service file content:**
```ini
[Unit]
Description=Apache Kafka (KRaft)
After=network-online.target
Wants=network-online.target

[Service]
User=kafka
Group=kafka
Environment=KAFKA_HEAP_OPTS=-Xms1g -Xmx1g
Environment=KAFKA_OPTS=
ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=100000
WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
```

### 5.2 Start Kafka Service

```bash
# Reload systemd and enable/start Kafka
sudo systemctl daemon-reload
sudo systemctl enable --now kafka

# Check service status
sudo systemctl status kafka --no-pager
```

**Expected output:**
```
● kafka.service - Apache Kafka (KRaft)
     Loaded: loaded (/etc/systemd/system/kafka.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2025-08-31 10:00:00 UTC; 10s ago
   Main PID: 12345 (java)
      Tasks: 50 (limit: 4915)
     Memory: 512.0M
        CPU: 5.234s
     CGroup: /system.slice/kafka.service
             └─12345 java -Xms1g -Xmx1g -server [...]
```

## Step 6: Cluster Verification

Run these commands from **any node** to verify the cluster:

### 6.1 Check KRaft Quorum Status

```bash
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server kafka-1:9092 describe --status
```

**Expected output:**
```
ClusterId:              kyLP6L_AThKpnaxwecuTsw
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          1
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [1, 2, 3]
CurrentObservers:       []
```

### 6.2 List Available Brokers

```bash
/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server kafka-1:9092
```

**Expected output:**
```
kafka-1:9092 (id: 1 rack: null) -> (
	Produce(0): 0 to 11 [usable: 11],
	Fetch(1): 0 to 16 [usable: 16],
	...
)
kafka-2:9092 (id: 2 rack: null) -> (
	...
)
kafka-3:9092 (id: 3 rack: null) -> (
	...
)
```

## Step 7: Testing the Cluster

### 7.1 Create a Test Topic

```bash
cd /opt/kafka/bin

# Create a test topic with 3 partitions and replication factor 3
./kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server kafka-1:9092 \
  --partitions 3 \
  --replication-factor 3
```

**Expected output:**
```
Created topic test-topic.
```

### 7.2 Verify Topic Creation

```bash
# Describe the topic to see partition distribution
./kafka-topics.sh --describe --topic test-topic --bootstrap-server kafka-1:9092
```

**Expected output:**
```
Topic: test-topic	TopicId: abc123def456	PartitionCount: 3	ReplicationFactor: 3	Configs: 
	Topic: test-topic	Partition: 0	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: test-topic	Partition: 1	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: test-topic	Partition: 2	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
```

### 7.3 Test Message Production and Consumption

**Terminal 1 - Producer:**
```bash
# Start producer (type messages and press Enter)
./kafka-console-producer.sh --topic test-topic --bootstrap-server kafka-1:9092
```

**Terminal 2 - Consumer:**
```bash
# Start consumer from beginning
./kafka-console-consumer.sh --topic test-topic --bootstrap-server kafka-2:9092 --from-beginning
```

**Example interaction:**
```
# In producer terminal:
> Hello Kafka Cluster!
> This is message number 2
> Testing replication across nodes

# In consumer terminal:
Hello Kafka Cluster!
This is message number 2
Testing replication across nodes
```

## Step 8: Advanced Operations

### 8.1 Topic Management Examples

```bash
# List all topics
./kafka-topics.sh --list --bootstrap-server kafka-1:9092

# Create topic with custom configuration
./kafka-topics.sh --create \
  --topic user-events \
  --bootstrap-server kafka-1:9092 \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config segment.ms=86400000

# Modify topic configuration
./kafka-configs.sh --bootstrap-server kafka-1:9092 \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config retention.ms=1209600000

# Delete a topic
./kafka-topics.sh --delete --topic test-topic --bootstrap-server kafka-1:9092
```

### 8.2 Consumer Groups

```bash
# Create consumer with group ID
./kafka-console-consumer.sh \
  --topic user-events \
  --bootstrap-server kafka-1:9092 \
  --group my-consumer-group \
  --from-beginning

# List consumer groups
./kafka-consumer-groups.sh --bootstrap-server kafka-1:9092 --list

# Describe consumer group
./kafka-consumer-groups.sh \
  --bootstrap-server kafka-1:9092 \
  --group my-consumer-group \
  --describe
```

### 8.3 Performance Testing

```bash
# Producer performance test
./kafka-producer-perf-test.sh \
  --topic user-events \
  --num-records 100000 \
  --record-size 1024 \
  --throughput 10000 \
  --producer-props bootstrap.servers=kafka-1:9092,kafka-2:9092,kafka-3:9092

# Consumer performance test
./kafka-consumer-perf-test.sh \
  --topic user-events \
  --messages 100000 \
  --bootstrap-server kafka-1:9092,kafka-2:9092,kafka-3:9092
```

## Configuration Reference

### Key Configuration Parameters

| Parameter | Description | Example Value |
|-----------|-------------|---------------|
| `node.id` | Unique identifier for each node | 1, 2, 3 |
| `process.roles` | Roles this node performs | broker,controller |
| `listeners` | Network interfaces to bind | BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093 |
| `advertised.listeners` | Address clients use to connect | BROKER://kafka-1:9092 |
| `controller.quorum.voters` | All controller nodes in quorum | 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093 |
| `log.dirs` | Data storage location | /var/lib/kafka/data |
| `metadata.log.dir` | Metadata storage location | /var/lib/kafka/meta |

## Monitoring and Troubleshooting

### 8.4 Health Checks

```bash
# Check cluster metadata
./kafka-metadata-quorum.sh --bootstrap-server kafka-1:9092 describe --status

# Check broker logs
sudo journalctl -u kafka -f

# Test cluster connectivity
./kafka-broker-api-versions.sh --bootstrap-server kafka-1:9092,kafka-2:9092,kafka-3:9092
```

### 8.5 Common Issues and Solutions

#### Issue: Node fails to join quorum
**Symptoms:** Broker starts but doesn't appear in cluster
**Solution:**
```bash
# Check network connectivity
telnet kafka-2 9093
telnet kafka-3 9093

# Verify configuration consistency across nodes
grep "controller.quorum.voters" /etc/kafka/server.properties
```

#### Issue: Storage format mismatch
**Symptoms:** "Log directory has not been formatted" error
**Solution:**
```bash
# Re-format with correct UUID (use same UUID on all nodes)
sudo systemctl stop kafka
sudo rm -rf /var/lib/kafka/{data,meta}/*
sudo mkdir -p /var/lib/kafka/{data,meta}
sudo chown -R kafka:kafka /var/lib/kafka
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format -t YOUR_CLUSTER_UUID -c /etc/kafka/server.properties
sudo systemctl start kafka
```

## Useful Commands Reference

```bash
# Service management
sudo systemctl {start|stop|restart|status} kafka

# Cluster status
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server kafka-1:9092 describe --status

# Topic operations
./kafka-topics.sh --bootstrap-server kafka-1:9092 {--list|--create|--delete|--describe}

# Consumer group management
./kafka-consumer-groups.sh --bootstrap-server kafka-1:9092 {--list|--describe}
```

This setup provides a robust foundation for Apache Kafka deployment in production or development environments......