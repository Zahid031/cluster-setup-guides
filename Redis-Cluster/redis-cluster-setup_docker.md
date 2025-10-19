# Redis Cluster Setup with Docker Compose

This guide walks you through setting up a Redis cluster with 6 nodes (3 masters + 3 replicas) using Docker Compose.

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of Redis clustering
- Host IP: `192.168.56.100` (adjust if different)

## Project Structure

```
redis-cluster/
├── docker-compose.yml
├── generate.sh
├── conf/
│   ├── node1/
│   ├── node2/
│   ├── node3/
│   ├── node4/
│   ├── node5/
│   └── node6/
└── data/
    ├── node1/
    ├── node2/
    ├── node3/
    ├── node4/
    ├── node5/
    └── node6/
```

## Step 1: Create Configuration Generator Script

Create a file named `generate.sh`:

```bash
#!/usr/bin/env bash
set -e

BASE_PORT=6380

for N in {1..6}; do
  PORT=$((BASE_PORT + N - 1))
  mkdir -p "conf/node${N}" "data/node${N}"
  
  cat > "conf/node${N}/redis.conf" <<EOF
# Redis config for node${N}
port ${PORT}
cluster-enabled yes
cluster-config-file /conf/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /data
bind 0.0.0.0
protected-mode no
requirepass testpass
masterauth testpass

# Announce IP for external clients
cluster-announce-ip 192.168.56.100
cluster-announce-port ${PORT}
cluster-announce-bus-port $((PORT + 10000))
EOF
  
  echo "Created conf/node${N}/redis.conf (port ${PORT})"
done
```

Make it executable and run it:

```bash
chmod +x generate.sh
./generate.sh
```

## Step 2: Set Directory Permissions

Grant necessary permissions to the configuration and data directories:

```bash
sudo chmod 777 -R data/
sudo chmod 777 -R conf/
```

## Step 3: Create Docker Compose File

Create `docker-compose.yml`:

```yaml
services:
  redis-node-1:
    image: redis:8.0
    container_name: redis-node-1
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node1/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node1:/conf
      - ./data/node1:/data
    ports:
      - "6380:6380"       # client port
      - "16380:16380"     # cluster bus port
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6380", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6

  redis-node-2:
    image: redis:8.0
    container_name: redis-node-2
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node2/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node2:/conf
      - ./data/node2:/data
    ports:
      - "6381:6381"
      - "16381:16381"
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6381", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6

  redis-node-3:
    image: redis:8.0
    container_name: redis-node-3
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node3/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node3:/conf
      - ./data/node3:/data
    ports:
      - "6382:6382"
      - "16382:16382"
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6382", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6

  redis-node-4:
    image: redis:8.0
    container_name: redis-node-4
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node4/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node4:/conf
      - ./data/node4:/data
    ports:
      - "6383:6383"
      - "16383:16383"
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6383", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6

  redis-node-5:
    image: redis:8.0
    container_name: redis-node-5
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node5/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node5:/conf
      - ./data/node5:/data
    ports:
      - "6384:6384"
      - "16384:16384"
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6384", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6

  redis-node-6:
    image: redis:8.0
    container_name: redis-node-6
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./conf/node6/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./conf/node6:/conf
      - ./data/node6:/data
    ports:
      - "6385:6385"
      - "16385:16385"
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "6385", "ping"]
      interval: 10s
      timeout: 5s
      retries: 6
```

## Step 4: Start Redis Nodes

Launch all Redis nodes:

```bash
docker compose up -d
```

Wait for all nodes to be healthy:

```bash
docker compose ps
```

## Step 5: Create the Cluster

Initialize the cluster with 3 masters and 3 replicas (1 replica per master):

```bash
docker compose exec redis-node-1 redis-cli -a testpass --cluster create \
  192.168.56.100:6380 \
  192.168.56.100:6381 \
  192.168.56.100:6382 \
  192.168.56.100:6383 \
  192.168.56.100:6384 \
  192.168.56.100:6385 \
  --cluster-replicas 1
```

Type `yes` when prompted to accept the configuration.

## Step 6: Verify Cluster Status

Check the cluster information:

```bash
docker compose exec redis-node-1 redis-cli -a testpass -p 6380 cluster info
```

View cluster nodes:

```bash
docker compose exec redis-node-1 redis-cli -a testpass -p 6380 cluster nodes
```

## Testing the Cluster

Set a key on the cluster:

```bash
docker compose exec redis-node-1 redis-cli -a testpass -c -p 6380 set mykey "Hello Redis Cluster"
```

Retrieve the key:

```bash
docker compose exec redis-node-1 redis-cli -a testpass -c -p 6380 get mykey
```

**Note:** The `-c` flag enables cluster mode, allowing automatic redirection to the correct node.

## Configuration Details

### Port Mapping

- **Client ports:** 6380-6385
- **Cluster bus ports:** 16380-16385

### Key Settings

- **Password:** `testpass` (change in production!)
- **Cluster timeout:** 5000ms
- **Persistence:** AOF (Append Only File) enabled
- **Bind address:** 0.0.0.0 (accessible from all interfaces)

### Cluster Topology

The `--cluster-replicas 1` option creates:
- 3 master nodes (nodes 1-3)
- 3 replica nodes (nodes 4-6)
- Each master has one replica for high availability

## Useful Commands

Stop the cluster:
```bash
docker compose down
```

View logs for a specific node:
```bash
docker compose logs redis-node-1
```

Access Redis CLI on a specific node:
```bash
docker compose exec redis-node-1 redis-cli -a testpass -c -p 6380
```

Remove all data and start fresh:
```bash
docker compose down -v
sudo rm -rf data/* conf/*/nodes.conf
./generate.sh
docker compose up -d
```

## Troubleshooting

### Cluster creation fails
- Ensure all nodes are healthy: `docker compose ps`
- Check that the host IP (`192.168.56.100`) is correct
- Verify no firewall is blocking cluster bus ports

### Permission denied errors
- Run: `sudo chmod 777 -R data/ conf/`

### Cannot connect from external client
- Verify `cluster-announce-ip` matches your host IP
- Ensure ports are properly exposed in `docker-compose.yml`

## Security Recommendations

For production environments:
1. Change the default password (`testpass`)
2. Use proper network isolation (Docker networks)
3. Enable TLS/SSL encryption
4. Implement firewall rules
5. Use volume mounts with appropriate permissions (not 777)
6. Consider using Redis ACLs for fine-grained access control