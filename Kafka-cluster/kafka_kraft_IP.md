# Apache Kafka KRaft Cluster Setup Guide with IP

This guide walks you through setting up a 3-node Apache Kafka cluster using KRaft mode (without Zookeeper) on Ubuntu/Debian systems.

## Prerequisites

- 3 Virtual Machines or servers
- Ubuntu/Debian operating system
- Network connectivity between all nodes
- Sudo access on all machines

## Cluster Architecture

| Node | IP Address | Role | Node ID |
|------|------------|------|---------|
| kafka-1 | 192.168.56.51 | broker,controller | 1 |
| kafka-2 | 192.168.56.52 | broker,controller | 2 |
| kafka-3 | 192.168.56.53 | broker,controller | 3 |

---

## Step 1: Configure Host Names (On All Nodes)

Add the cluster node entries to the hosts file on each machine:

```bash
echo "192.168.56.51 kafka-1
192.168.56.52 kafka-2
192.168.56.53 kafka-3" | sudo tee -a /etc/hosts
```

## Step 2: Install Java (On All Nodes)

Update the package index and install OpenJDK 21:

```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```

Verify the Java installation:

```bash
java -version
```

## Step 3: Create Kafka User and Directories (On All Nodes)

Create a dedicated kafka user for security:

```bash
sudo useradd -r -m -d /var/lib/kafka -s /usr/sbin/nologin kafka || true
```

Create necessary directories:

```bash
sudo mkdir -p /opt/kafka /var/lib/kafka/{data,meta,logs}
sudo chown -R kafka:kafka /opt/kafka /var/lib/kafka
```

## Step 4: Download and Install Kafka (On All Nodes)

Download Kafka 4.0.0:

```bash
cd /tmp
curl -LO https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
```

Extract and install Kafka:

```bash
sudo tar -xzf kafka_2.13-4.0.0.tgz -C /opt/kafka --strip-components=1
sudo chown -R kafka:kafka /opt/kafka
```

Create configuration directory:

```bash
sudo mkdir -p /etc/kafka
sudo chown -R kafka:kafka /etc/kafka
```

## Step 5: Configure Kafka (On Each Node Individually)

Create the server configuration file. **Important:** The `node.id` and `advertised.listeners` must be different on each node.

### For kafka-1 (192.168.56.51):

```bash
sudo tee /etc/kafka/server.properties > /dev/null <<EOF
# --- Identity & roles ---
node.id=1
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.56.51:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@192.168.56.51:9093,2@192.168.56.52:9093,3@192.168.56.53:9093

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
EOF
```

### For kafka-2 (192.168.56.52):

```bash
sudo tee /etc/kafka/server.properties > /dev/null <<EOF
# --- Identity & roles ---
node.id=2
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.56.52:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@192.168.56.51:9093,2@192.168.56.52:9093,3@192.168.56.53:9093

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
EOF
```

### For kafka-3 (192.168.56.53):

```bash
sudo tee /etc/kafka/server.properties > /dev/null <<EOF
# --- Identity & roles ---
node.id=3
process.roles=broker,controller

# --- Networking ---
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.56.53:9092

# --- Quorum: ALL NODES, with nodeId@host:port ---
controller.quorum.voters=1@192.168.56.51:9093,2@192.168.56.52:9093,3@192.168.56.53:9093

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
EOF
```

## Step 6: Initialize Kafka Storage (On All Nodes)

Generate a cluster UUID (run this on any one node and use the same UUID on all nodes):

```bash
sudo -u kafka /opt/kafka/bin/kafka-storage.sh random-uuid
```

**Note:** Use the same UUID across all nodes. Example output: `kyLP6L_AThKpnaxwecuTsw`

Format the storage directories on each node using the generated UUID:

```bash
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t kyLP6L_AThKpnaxwecuTsw \
  -c /etc/kafka/server.properties
```

## Step 7: Create Systemd Service (On All Nodes)

Create a systemd service file for Kafka:

```bash
sudo tee /etc/systemd/system/kafka.service > /dev/null <<EOF
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
EOF
```

## Step 8: Start Kafka Services (On All Nodes)

Reload systemd and start Kafka:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka
```

Check the service status:

```bash
sudo systemctl status kafka --no-pager
```

## Step 9: Verify Cluster Setup

### Check KRaft Quorum Status

From any node, verify the quorum is healthy:

```bash
/opt/kafka/bin/kafka-metadata-quorum.sh \
  --bootstrap-server 192.168.56.51:9092 \
  describe --status
```

### List Cluster Brokers

Verify all brokers are visible:

```bash
/opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server 192.168.56.51:9092
```

## Step 10: Test the Cluster

Navigate to the Kafka bin directory:

```bash
cd /opt/kafka/bin
```

### Create a Test Topic

```bash
./kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server 192.168.56.51:9092 \
  --partitions 3 \
  --replication-factor 3
```

### Describe the Topic

Verify the topic was created correctly:

```bash
./kafka-topics.sh --describe \
  --topic test-topic \
  --bootstrap-server 192.168.56.51:9092
```

### Produce Messages

Start a producer in one terminal:

```bash
./kafka-console-producer.sh \
  --topic test-topic \
  --bootstrap-server 192.168.56.51:9092
```

Type some messages and press Enter after each one.

### Consume Messages

In another terminal (can be on a different node), start a consumer:

```bash
./kafka-console-consumer.sh \
  --topic test-topic \
  --bootstrap-server 192.168.56.52:9092 \
  --from-beginning
```

You should see the messages you produced appearing in the consumer terminal.

---

## Key Configuration Explanations

### KRaft Mode Benefits
- **No Zookeeper Required:** Simplifies architecture and reduces operational overhead
- **Better Performance:** Reduced latency and improved throughput
- **Simplified Operations:** Easier to deploy, monitor, and maintain

### Important Configuration Parameters

- **`node.id`:** Unique identifier for each node (1, 2, 3)
- **`process.roles`:** Defines if the node acts as broker, controller, or both
- **`controller.quorum.voters`:** Lists all controller nodes in the cluster
- **`advertised.listeners`:** How clients connect to this specific broker
- **`replication.factor`:** Number of copies of each partition (set to 3 for fault tolerance)

### Directory Structure

- **`/opt/kafka`:** Kafka installation directory
- **`/var/lib/kafka/data`:** Topic partition data storage
- **`/var/lib/kafka/meta`:** Metadata logs storage
- **`/etc/kafka`:** Configuration files

---

## Troubleshooting Tips

1. **Check logs:** `sudo journalctl -u kafka -f`
2. **Verify network connectivity:** Test ports 9092 and 9093 between nodes
3. **Ensure consistent UUID:** All nodes must use the same cluster UUID
4. **Check disk space:** Ensure adequate space in `/var/lib/kafka`
5. **Firewall:** Make sure ports 9092 and 9093 are open between cluster nodes

## Next Steps

- Configure monitoring (JMX metrics)
- Set up SSL/SASL for security
- Tune performance parameters based on your workload
- Implement backup and disaster recovery procedures

Your Kafka KRaft cluster is now ready for production workloads!
