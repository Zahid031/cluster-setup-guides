# Redis Cluster Setup Guide

This guide shows how to set up a Redis cluster with multiple nodes and a proxy for easy access.

## 1. Install Redis

```bash
# Update system and install dependencies
sudo apt update
sudo apt-get install lsb-release curl gpg

# Add Redis repository
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Install Redis
sudo apt-get update
sudo apt-get install redis
```

## 2. Configure Multiple Redis Instances

### Create Data Directories
```bash
sudo mkdir -p /var/lib/redis/6380 /var/lib/redis/6381
sudo chown -R redis:redis /var/lib/redis
```

### Setup Configuration Files
```bash
# Copy base config for each instance
sudo cp /etc/redis/redis.conf /etc/redis/redis-6380.conf
sudo cp /etc/redis/redis.conf /etc/redis/redis-6381.conf
```

### Configure Instance 1 (Port 6380)
```bash
sudo sed -i 's/^port .*/port 6380/' /etc/redis/redis-6380.conf
sudo sed -i 's|^dir .*|dir /var/lib/redis/6380|' /etc/redis/redis-6380.conf

# Add cluster settings
echo '
daemonize yes
pidfile /var/run/redis_6380.pid
logfile /var/log/redis/redis-server-6380.log
cluster-enabled yes
cluster-config-file nodes-6380.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-6380.aof"
bind 0.0.0.0
protected-mode no
' | sudo tee -a /etc/redis/redis-6380.conf
```

### Configure Instance 2 (Port 6381)
```bash
sudo sed -i 's/^port .*/port 6381/' /etc/redis/redis-6381.conf
sudo sed -i 's|^dir .*|dir /var/lib/redis/6381|' /etc/redis/redis-6381.conf

# Add cluster settings
echo '
daemonize yes
pidfile /var/run/redis_6381.pid
logfile /var/log/redis/redis-server-6381.log
cluster-enabled yes
cluster-config-file nodes-6381.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-6381.aof"
bind 0.0.0.0
protected-mode no
' | sudo tee -a /etc/redis/redis-6381.conf
```

## 3. Start Redis Instances

```bash
sudo redis-server /etc/redis/redis-6380.conf
sudo redis-server /etc/redis/redis-6381.conf
```

## 4. Create Redis Cluster

```bash
# Create cluster with 3 masters and 3 replicas across multiple servers
redis-cli --cluster create \
  192.168.56.21:6380 192.168.56.22:6380 192.168.56.23:6380 \
  192.168.56.22:6381 192.168.56.23:6381 192.168.56.21:6381 \
  --cluster-replicas 1
```

## 5. Verify Cluster

```bash
# Check cluster nodes
redis-cli -c -p 6380 cluster nodes

# Check cluster info
redis-cli -c -p 6380 cluster info

# Check from remote server
redis-cli -c -h 192.168.56.21 -p 6380 cluster nodes
```

## 6. Setup Redis Cluster Proxy

### Create Proxy Configuration
Create `redis-cluster-proxy.conf`:
```
cluster 192.168.56.21:6380
cluster 192.168.56.22:6380
cluster 192.168.56.23:6380
cluster 192.168.56.21:6381
cluster 192.168.56.22:6381
cluster 192.168.56.23:6381

port 6379
bind 0.0.0.0
```

### Docker Compose Setup
Create `docker-compose.yaml`:
```yaml
services:
  redis-cluster-proxy:
    image: kornrunner/redis-cluster-proxy:latest
    container_name: redis-cluster-proxy
    ports:
      - "6379:6379"        
    volumes:
      - ./redis-cluster-proxy.conf:/etc/redis-cluster-proxy.conf:ro
    command: ["-c", "/etc/redis-cluster-proxy.conf"]
    restart: unless-stopped
```

## 7. Test the Cluster

### Python Script to Write Data
```python
from redis import Redis

# Connect through proxy
r = Redis(host='192.168.56.25', port=6379, decode_responses=True)

# Write test data
for i in range(1, 21):
    key = f"key{i}"
    value = f"value{i}"
    try:
        r.set(key, value)
        print(f"Successfully set {key} = {value}")
    except Exception as e:
        print(f"Error setting {key}: {e}")
```

### Python Script to Read Data
```python
from redis import Redis

r = Redis(host='192.168.56.25', port=6379, decode_responses=True)

# Read test data
for i in range(1, 21):
    key = f"key{i}"
    try:
        value = r.get(key)
        if value is not None:
            print(f"Retrieved {key} = {value}")
        else:
            print(f"Key {key} not found")
    except Exception as e:
        print(f"Error retrieving {key}: {e}")

r.close()
```

## 8. Performance Testing

### Install Benchmarking Tools
```bash

sudo apt install lsb-release curl gpg

curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt-get update
sudo apt-get install memtier-benchmark
```

### Redis Benchmark
```bash
# Basic benchmark
redis-benchmark -t set -n 100000 -r 1000000

# Cluster benchmark
redis-benchmark --cluster -h 192.168.56.21 -p 6380 -c 100 -n 10000 -t set -r 1000000

# Test through proxy
redis-benchmark -h 192.168.56.25 -p 6379 -n 100000 -c 100 -t set,get
```

### Memtier Benchmark
```bash
memtier_benchmark \
  -s 192.168.56.25 -p 6379 \
  --ratio=1:1 \
  --data-size=102400 \
  -n 10000 \
  -c 2 -t 2
```

## 9. Management Commands

### Shutdown Node
```bash
redis-cli -h 192.168.56.22 -p 6380 shutdown
```

### Install Python Redis Client
```bash
pip install redis-py-cluster
```

## Architecture Overview

- **3 Master Nodes**: Handle read/write operations across different hash slots
- **3 Replica Nodes**: Provide high availability and read scaling
- **Redis Cluster Proxy**: Single entry point that routes requests to appropriate nodes
- **Hash Slots**: Data is distributed across 16,384 hash slots among master nodes

This setup provides high availability, automatic failover, and horizontal scaling for your Redis deployment.
