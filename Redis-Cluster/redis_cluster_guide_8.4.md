# Redis Cluster Setup Guide

A simple guide to set up a Redis cluster with 6 nodes across 3 servers for high availability.

## 📋 What You'll Build

- **3 Servers**: Each running 2 Redis instances
- **6 Total Nodes**: 3 masters + 3 replicas
- **High Availability**: Automatic failover if a master goes down

**Example Setup:**
```
Server 1 (10.0.0.1): Redis on ports 6380 & 6381
Server 2 (10.0.0.2): Redis on ports 6380 & 6381
Server 3 (10.0.0.3): Redis on ports 6380 & 6381
```

---

## 🚀 Step 1: Install Redis

Run these commands on **all 3 servers**:

```bash
# Update system
sudo apt update
sudo apt install -y lsb-release curl gpg

# Add Redis official repository
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Install Redis
sudo apt update
sudo apt install -y redis

# Verify installation
redis-server --version
```

---

## 📁 Step 2: Create Data Directories

Run on **all 3 servers**:

```bash
# Create directories for both Redis instances
mkdir -p /data/redis/6380 /data/redis/6381

# Set proper ownership
chown -R redis:redis /data/redis
chmod -R 750 /data/redis
```

---

## ⚙️ Step 3: Configure Redis Instances

### Create Configuration Files

On **all 3 servers**:

```bash
# Copy default config for both instances
sudo cp /etc/redis/redis.conf /etc/redis/redis-6380.conf
sudo cp /etc/redis/redis.conf /etc/redis/redis-6381.conf

# Set ownership
sudo chown redis:redis /etc/redis/redis-6380.conf /etc/redis/redis-6381.conf
```

### Configure Port 6380

Edit the file:

```bash
sudo nano /etc/redis/redis-6380.conf
```

Add these settings:

```conf
# Network - allow all connections
bind 0.0.0.0
port 6380

# Cluster Settings
cluster-announce-ip 10.0.0.1           # Change: Use your server's IP
cluster-announce-port 6380
cluster-announce-bus-port 16380
cluster-enabled yes
cluster-config-file nodes-6380.conf
cluster-node-timeout 5000

# Data Storage
dir /data/redis/6380
logfile /data/redis/6380/redis.log
dbfilename dump-6380.rdb
appendfilename appendonly-6380.aof
appendonly yes
appendfsync everysec

# Security
requirepass MySecurePass123              # Change: Use your password
masterauth MySecurePass123               # Change: Use your password

# Memory
maxmemory 4gb
maxmemory-policy volatile-lru

# System
supervised systemd
daemonize no
protected-mode no
cluster-require-full-coverage no

```

### Configure Port 6381

Edit the file:

```bash
sudo nano /etc/redis/redis-6381.conf
```

Add these settings:

```conf
# Network
bind 0.0.0.0
port 6381

# Cluster Settings
cluster-announce-ip 10.0.0.1           # Change: Use your server's IP
cluster-announce-port 6381
cluster-announce-bus-port 16381
cluster-enabled yes
cluster-config-file nodes-6381.conf
cluster-node-timeout 5000

# Data Storage
dir /data/redis/6381
logfile /data/redis/6381/redis.log
dbfilename dump-6381.rdb
appendfilename appendonly-6381.aof
appendonly yes
appendfsync everysec

# Security
requirepass MySecurePass123              # Change: Use your password
masterauth MySecurePass123               # Change: Use your password

# Memory
maxmemory 4gb
maxmemory-policy volatile-lru

# System
supervised systemd
daemonize no
protected-mode no
cluster-require-full-coverage no

```

> **⚠️ Important:** 
> - Change `cluster-announce-ip` to match each server's actual IP
> - Use a strong password instead of `MySecurePass123`
> - Keep the password same across all servers

---

## 🔧 Step 4: Setup Systemd Service

Create a service template on **all 3 servers**:

```bash
sudo nano /etc/systemd/system/redis@.service
```

Add this content:

```ini
[Unit]
Description=Redis Instance %i
After=network.target
Wants=network-online.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/redis-%i.conf
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always
RestartSec=5
LimitNOFILE=100000
RuntimeDirectory=redis
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
```

### Start Redis Services

On **all 3 servers**:

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start both instances
sudo systemctl start redis@6380
sudo systemctl start redis@6381

# Enable auto-start on boot
sudo systemctl enable redis@6380
sudo systemctl enable redis@6381

# Check status
sudo systemctl status redis@6380
sudo systemctl status redis@6381
```

---



## 🔗 Step 5: Create the Cluster

### Test Connectivity First

From **any server**, test if all nodes are reachable:

```bash
redis-cli -a MySecurePass123 -h 10.0.0.1 -p 6380 ping
redis-cli -a MySecurePass123 -h 10.0.0.1 -p 6381 ping
redis-cli -a MySecurePass123 -h 10.0.0.2 -p 6380 ping
redis-cli -a MySecurePass123 -h 10.0.0.2 -p 6381 ping
redis-cli -a MySecurePass123 -h 10.0.0.3 -p 6380 ping
redis-cli -a MySecurePass123 -h 10.0.0.3 -p 6381 ping
```

All should return: `PONG`

### Create the Cluster

Run from **any one server**:

```bash
redis-cli -a MySecurePass123 --cluster create \
10.0.0.1:6380 10.0.0.1:6381 \
10.0.0.2:6380 10.0.0.2:6381 \
10.0.0.3:6380 10.0.0.3:6381 \
--cluster-replicas 1
```

**What this does:**
- Creates 3 master nodes (one per server)
- Creates 3 replica nodes (one per server)
- Distributes data across masters
- Type `yes` when prompted

---

## ✅ Step 6: Verify Cluster

### Check Cluster Status

```bash
redis-cli -a MySecurePass123 -p 6380 cluster info
```

Expected output:
```
cluster_state:ok
cluster_slots_assigned:16384
cluster_known_nodes:6
cluster_size:3
```

### View All Nodes

```bash
redis-cli -a MySecurePass123 -p 6380 cluster nodes
```

### Connect and Test

```bash
# Connect in cluster mode
redis-cli -c -a MySecurePass123 -h 10.0.0.1 -p 6380

# Test setting and getting data
SET mykey "Hello Redis Cluster"
GET mykey
```

---

## 🛠️ Common Commands

### Check Service Status
```bash
sudo systemctl status redis@6380
sudo systemctl status redis@6381
```

### View Logs
```bash
sudo tail -f /data/redis/6380/redis.log
sudo tail -f /data/redis/6381/redis.log
```

### Restart Services
```bash
sudo systemctl restart redis@6380
sudo systemctl restart redis@6381
```

### Check Cluster Health
```bash
redis-cli -a MySecurePass123 -p 6380 --cluster check 10.0.0.1:6380
```

---

## 🔄 Reset Cluster (If Needed)

If you need to start over:

```bash
# Stop services
sudo systemctl stop redis@6380
sudo systemctl stop redis@6381

# Remove cluster data
rm -rf /data/redis/6380/*
rm -rf /data/redis/6381/*

# Start services
sudo systemctl start redis@6380
sudo systemctl start redis@6381

# Or reset without stopping
redis-cli -a MySecurePass123 -p 6380 cluster reset hard
redis-cli -a MySecurePass123 -p 6381 cluster reset hard
```

---

## 📊 Understanding the Setup

### What is a Redis Cluster?
- Splits data across multiple servers (sharding)
- Each master handles a portion of the data (hash slots)
- Each master has a replica for backup
- If a master fails, its replica takes over automatically

### Our Configuration:
```
Server 1: Master A + Replica B
Server 2: Master B + Replica C  
Server 3: Master C + Replica A
```

This ensures if any server goes down, you still have all data available.

---

## 🔒 Security Checklist

- ✅ Change default password (`MySecurePass123`)
- ✅ Use firewall to restrict access
- ✅ Consider using private network IPs
- ✅ Enable SSL/TLS in production
- ✅ Regular backups of `/data/redis/` directories
- ✅ Monitor memory usage and disk space

---

## 🆘 Troubleshooting

**Problem: Can't connect to Redis**
```bash
# Check if Redis is running
sudo systemctl status redis@6380

# Check if port is listening
sudo netstat -tulpn | grep 6380

# Check firewall
sudo ufw status
```

**Problem: Cluster creation fails**
```bash
# Ensure all nodes can communicate
ping 10.0.0.2
ping 10.0.0.3

# Check Redis logs
sudo tail -f /data/redis/6380/redis.log
```

**Problem: Out of memory errors**
```bash
# Check memory usage
redis-cli -a MySecurePass123 -p 6380 info memory

# Adjust maxmemory in config file if needed
```

---

## 📚 Quick Reference

| Command | Description |
|---------|-------------|
| `redis-cli -c -a <pass> -h <ip> -p <port>` | Connect to cluster |
| `cluster info` | View cluster status |
| `cluster nodes` | List all nodes |
| `info replication` | Check master/replica status |
| `ping` | Test connection |
| `shutdown` | Stop Redis instance |

---

## ✨ You're Done!

Your Redis cluster is now ready. You have:
- ✅ 6 Redis nodes across 3 servers
- ✅ Automatic failover protection
- ✅ Data distributed across masters
- ✅ Persistent storage with AOF

**Next Steps:** Connect your application using a Redis cluster client and start storing data!
