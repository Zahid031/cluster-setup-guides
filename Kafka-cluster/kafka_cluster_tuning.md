# Kafka KRaft Cluster Setup Guide

Complete setup guide for 3-node Kafka cluster with actual configurations and performance results.

## Cluster Setup

### Node Configuration
| Node | IP Address | Role | Node ID |
|------|------------|------|---------|
| kafka-1 | 192.168.61.146 | broker,controller | 1 |
| kafka-2 | 192.168.61.147 | broker,controller | 2 |
| kafka-3 | 192.168.61.148 | broker,controller | 3 |

## Installation Steps

### 1. Configure Hosts (All Nodes)
```bash
echo "192.168.61.146 kafka-1
192.168.61.147 kafka-2
192.168.61.148 kafka-3" | sudo tee -a /etc/hosts
```

### 2. Install Java (All Nodes)
```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```

### 3. Create User and Directories (All Nodes)
```bash
sudo useradd -r -m -d /var/lib/kafka -s /usr/sbin/nologin kafka
sudo mkdir -p /opt/kafka /var/lib/kafka/{data,meta,logs}
sudo chown -R kafka:kafka /opt/kafka /var/lib/kafka
```

### 4. Download and Install Kafka (All Nodes)
```bash
cd /tmp
curl -LO https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
sudo tar -xzf kafka_2.13-4.0.0.tgz -C /opt/kafka --strip-components=1
sudo chown -R kafka:kafka /opt/kafka
sudo mkdir -p /etc/kafka
sudo chown -R kafka:kafka /etc/kafka
```

## Configuration Files

### Server Properties - kafka-1 (192.168.61.146)
```bash
sudo tee /etc/kafka/server.properties > /dev/null <<'EOF'
node.id=1
process.roles=broker,controller
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.61.146:9092
controller.quorum.voters=1@192.168.61.146:9093,2@192.168.61.147:9093,3@192.168.61.148:9093
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta
num.network.threads=4
num.io.threads=8
num.replica.fetchers=3
queued.max.requests=1000
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
EOF
```

### Server Properties - kafka-2 (192.168.61.147)
```bash
sudo tee /etc/kafka/server.properties > /dev/null <<'EOF'
node.id=2
process.roles=broker,controller
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.61.147:9092
controller.quorum.voters=1@192.168.61.146:9093,2@192.168.61.147:9093,3@192.168.61.148:9093
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta
num.network.threads=4
num.io.threads=8
num.replica.fetchers=3
queued.max.requests=1000
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
EOF
```

### Server Properties - kafka-3 (192.168.61.148)
```bash
sudo tee /etc/kafka/server.properties > /dev/null <<'EOF'
node.id=3
process.roles=broker,controller
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.61.148:9092
controller.quorum.voters=1@192.168.61.146:9093,2@192.168.61.147:9093,3@192.168.61.148:9093
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta
num.network.threads=4
num.io.threads=8
num.replica.fetchers=3
queued.max.requests=1000
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=false
EOF
```

## System Tuning

### Kernel Parameters (All Nodes)
```bash
sudo tee /etc/sysctl.d/99-kafka.conf > /dev/null <<'EOF'
vm.swappiness = 1
vm.max_map_count = 262144
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 60
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 268435456
net.ipv4.tcp_wmem = 4096 65536 268435456
EOF

sudo sysctl -p /etc/sysctl.d/99-kafka.conf
```

### Producer Configuration (All Nodes)
```bash
sudo tee /etc/kafka/producer.properties > /dev/null <<'EOF'
bootstrap.servers=192.168.61.146:9092,192.168.61.147:9092,192.168.61.148:9092
acks=1
batch.size=1048576
linger.ms=5
compression.type=snappy
EOF
```

## Service Setup

### Initialize Storage (All Nodes)
```bash
# Generate UUID (use same on all nodes)
sudo -u kafka /opt/kafka/bin/kafka-storage.sh random-uuid

# Format storage with UUID
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t <YOUR_UUID_HERE> \
  -c /etc/kafka/server.properties
```

### Systemd Service (All Nodes)
```bash
sudo tee /etc/systemd/system/kafka.service > /dev/null <<'EOF'
[Unit]
Description=Apache Kafka (KRaft)
After=network-online.target
Wants=network-online.target

[Service]
User=kafka
Group=kafka
Environment=KAFKA_HEAP_OPTS=-Xms3G -Xmx3G
ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=100000
WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kafka
```

## Testing and Verification

### Create Test Topics
```bash
# Create test topic with RF=3, 3 partitions
/opt/kafka/bin/kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server 192.168.61.146:9092 \
  --partitions 3 \
  --replication-factor 3

# Create large test topic with RF=6, 6 partitions for high volume testing
/opt/kafka/bin/kafka-topics.sh --create \
  --topic large-test-topic \
  --bootstrap-server 192.168.61.146:9092 \
  --partitions 6 \
  --replication-factor 6
```

### Performance Tests

#### Test 1: 512KB Records, 6000 Messages (Inline Properties)
**Description**: Testing large message throughput with inline producer properties using smaller batch size (64KB).

```bash
# 512KB records, 6000 messages
/opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic large-test-topic \
  --num-records 6000 \
  --record-size 524288 \
  --throughput -1 \
  --producer-props bootstrap.servers=192.168.61.146:9092 acks=1 batch.size=65536 linger.ms=5 compression.type=snappy
```

**Detailed Results**:
```
2307 records sent, 461.0 records/sec (230.52 MB/sec), 9.5 ms avg latency, 489.0 ms max latency.
2671 records sent, 534.2 records/sec (267.10 MB/sec), 5.9 ms avg latency, 113.0 ms max latency.
6000 records sent, 501.6 records/sec (250.79 MB/sec), 7.00 ms avg latency, 489.00 ms max latency, 4 ms 50th, 14 ms 95th, 87 ms 99th, 183 ms 99.9th.
```

**Performance Analysis**: Excellent low latency performance with 4ms median latency and consistent throughput at 250MB/sec.

---

#### Test 2: 512KB Records, 10000 Messages (Config File)
**Description**: Same test using producer config file with optimized 1MB batch size instead of inline properties.

```bash
/opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic large-test-topic \
  --num-records 10000 \
  --record-size 524288 \
  --throughput -1 \
  --producer.config /etc/kafka/producer.properties
```

**Detailed Results**:
```
2206 records sent, 441.2 records/sec (220.60 MB/sec), 7.8 ms avg latency, 452.0 ms max latency.
2460 records sent, 491.8 records/sec (245.90 MB/sec), 11.1 ms avg latency, 386.0 ms max latency.
2391 records sent, 478.2 records/sec (239.10 MB/sec), 14.4 ms avg latency, 222.0 ms max latency.
2403 records sent, 480.5 records/sec (240.25 MB/sec), 7.8 ms avg latency, 252.0 ms max latency.
10000 records sent, 474.3 records/sec (237.17 MB/sec), 10.04 ms avg latency, 452.00 ms max latency, 5 ms 50th, 28 ms 95th, 128 ms 99th, 313 ms 99.9th.
```

**Performance Analysis**: Larger batch size (1MB) shows slightly higher but still excellent latency with 5ms median. Throughput remains strong at 237MB/sec.

---

#### Test 3: 100KB Records, 100K Messages, RF=6 (High Replication)
**Description**: High volume test with maximum replication factor testing cluster fault tolerance impact on performance.

```bash
# 100KB size, 100000 records, replication factor 6
/opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic large-test-topic \
  --num-records 100000 \
  --record-size 100000 \
  --throughput -1 \
  --producer.config /etc/kafka/producer.properties
```

**Detailed Results**:
```
12702 records sent, 2537.9 records/sec (242.03 MB/sec), 17.6 ms avg latency, 459.0 ms max latency.
14110 records sent, 2821.4 records/sec (269.07 MB/sec), 13.2 ms avg latency, 147.0 ms max latency.
13603 records sent, 2718.4 records/sec (259.25 MB/sec), 40.5 ms avg latency, 734.0 ms max latency.
5973 records sent, 1193.6 records/sec (113.83 MB/sec), 94.1 ms avg latency, 2760.0 ms max latency.
10117 records sent, 2023.0 records/sec (192.93 MB/sec), 121.3 ms avg latency, 4768.0 ms max latency.
7125 records sent, 1423.9 records/sec (135.79 MB/sec), 96.9 ms avg latency, 3536.0 ms max latency.
5760 records sent, 1141.7 records/sec (108.88 MB/sec), 301.3 ms avg latency, 5676.0 ms max latency.
9852 records sent, 1970.4 records/sec (187.91 MB/sec), 94.2 ms avg latency, 3282.0 ms max latency.
9743 records sent, 1946.3 records/sec (185.61 MB/sec), 161.7 ms avg latency, 3952.0 ms max latency.
9813 records sent, 1962.2 records/sec (187.13 MB/sec), 121.9 ms avg latency, 2286.0 ms max latency.
100000 records sent, 1956.0 records/sec (186.53 MB/sec), 91.63 ms avg latency, 5676.00 ms max latency, 8 ms 50th, 261 ms 95th, 2627 ms 99th, 5084 ms 99.9th.
```

**Performance Analysis**: RF=6 significantly impacts performance due to replication overhead. Higher latency variance visible with 99th percentile at 2.6 seconds. However, median latency remains low at 8ms.

---

#### Test 4: 100KB Records, 100K Messages, RF=3 (Optimal Configuration)
**Description**: High volume test with standard replication factor showing optimal balance of performance and reliability.

```bash
# 100KB size, 100000 records, replication factor 3
/opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic test-topic \
  --num-records 100000 \
  --record-size 100000 \
  --throughput -1 \
  --producer.config /etc/kafka/producer.properties
```

**Detailed Results**:
```
12596 records sent, 2518.7 records/sec (240.20 MB/sec), 12.4 ms avg latency, 416.0 ms max latency.
14638 records sent, 2927.6 records/sec (279.20 MB/sec), 12.2 ms avg latency, 566.0 ms max latency.
15154 records sent, 3030.8 records/sec (289.04 MB/sec), 9.3 ms avg latency, 128.0 ms max latency.
13788 records sent, 2756.5 records/sec (262.88 MB/sec), 16.7 ms avg latency, 884.0 ms max latency.
13450 records sent, 2688.9 records/sec (256.44 MB/sec), 58.4 ms avg latency, 2779.0 ms max latency.
14228 records sent, 2843.9 records/sec (271.21 MB/sec), 10.6 ms avg latency, 346.0 ms max latency.
14284 records sent, 2856.8 records/sec (272.45 MB/sec), 13.5 ms avg latency, 233.0 ms max latency.
100000 records sent, 2808.9 records/sec (267.88 MB/sec), 20.76 ms avg latency, 2779.00 ms max latency, 7 ms 50th, 36 ms 95th, 174 ms 99th, 2125 ms 99.9th.
```

**Performance Analysis**: Best overall performance with RF=3. Peak throughput reached 289MB/sec with excellent 7ms median latency. 99th percentile at 174ms shows much better consistency than RF=6.

## Performance Summary

| Test | Record Size | Records | RF | MB/sec | Records/sec | Avg Latency |
|------|-------------|---------|----|----|-------------|-------------|
| 1 | 512KB | 6K | 3 | 250.79 | 501.6 | 7.00 ms |
| 2 | 512KB | 10K | 3 | 237.17 | 474.3 | 10.04 ms |
| 3 | 100KB | 100K | 6 | 186.53 | 1,956.0 | 91.63 ms |
| 4 | 100KB | 100K | 3 | 267.88 | 2,808.9 | 20.76 ms |

## Monitoring Commands

```bash
# Check cluster health
/opt/kafka/bin/kafka-metadata-quorum.sh \
  --bootstrap-server 192.168.61.146:9092 describe --status

# List topics
/opt/kafka/bin/kafka-topics.sh --list \
  --bootstrap-server 192.168.61.146:9092

# Monitor logs
sudo journalctl -u kafka -f

# Check service status
sudo systemctl status kafka
```