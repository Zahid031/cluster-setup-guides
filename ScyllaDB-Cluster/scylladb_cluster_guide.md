# ScyllaDB 3-Node Cluster SetUP and Configuration Guide

This comprehensive guide will walk you through setting up a production-ready ScyllaDB cluster with 3 nodes, including security configurations and best practices.

## Prerequisites

### System Requirements
- **Operating System**: Ubuntu 20.04 LTS or newer
- **CPU**: x86_64 with SSE4.2 and PCLMUL support
- **Memory**: Minimum 4GB RAM per node (8GB+ recommended for production)
- **Storage**: SSD storage recommended for optimal performance
- **Network**: All nodes should be able to communicate with each other

### Network Planning
Before starting, plan your IP addresses:
- **Node 1**: 192.168.1.10 (example)
- **Node 2**: 192.168.1.11 (example)  
- **Node 3**: 192.168.1.12 (example)

## Step 1: Pre-Installation Checks

### Verify CPU Compatibility
Run this command on all nodes to ensure your CPU supports required features:

```bash
lscpu | grep -E 'sse4_2|pclmul'
```

**Why this matters**: ScyllaDB requires SSE4.2 and PCLMUL CPU instructions for optimal performance and certain cryptographic operations.

### Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 software-properties-common net-tools
```

## Step 2: Install ScyllaDB

Run the following command on **all three nodes**:

```bash
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-version 6.2.3
```


## Step 3: Configure ScyllaDB

### Edit Main Configuration File

On **each node**, edit the ScyllaDB configuration:

```bash
sudo vi /etc/scylla/scylla.yaml
```

**Key configurations to set:**

```yaml
# Cluster identification
cluster_name: 'Scylla-Cluster'

# Storage directories
workdir: /var/lib/scylla
data_file_directories:
    - /var/lib/scylla/data
commitlog_directory: /var/lib/scylla/commitlog

# Seed nodes configuration
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # Use IP addresses of your seed nodes
          - seeds: "192.168.1.10,192.168.1.11,192.168.1.12"

# Node-specific network settings (CHANGE THIS FOR EACH NODE)
listen_address: 192.168.1.10      # Current node's IP
rpc_address: 192.168.1.10         # Current node's IP

# Network topology
endpoint_snitch: GossipingPropertyFileSnitch

# Security settings
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

### Configuration Explanations

- **cluster_name**: Identifies your cluster. All nodes must have the same name.
- **seeds**: Bootstrap nodes that help new nodes discover the cluster topology.
- **listen_address**: IP address for internal cluster communication.
- **rpc_address**: IP address for client connections.
- **endpoint_snitch**: Determines network topology and routing strategy.
- **authenticator**: Enables password-based authentication.
- **authorizer**: Enables role-based access control.

### Configure Data Center Properties

Edit the rack and datacenter properties on **all nodes**:

```bash
sudo vi /etc/scylla/cassandra-rackdc.properties
```

```properties
dc=scylla_data_center
rack=scylla_rack
```

**Why configure this**: Proper datacenter and rack configuration is crucial for replication strategies and network topology awareness.

## Step 4: System Optimization

Run the ScyllaDB setup script on **each node**:

```bash
sudo scylla_setup
```

### Setup Script Responses

When prompted, use these recommended responses:

| Question | Response | Reason |
|----------|----------|---------|
| Check kernel version? | **yes** | Ensures compatibility |
| Verify packages installed? | **yes** | Confirms proper installation |
| Auto-start on boot? | **yes** | Ensures service availability |
| Check for newer versions? | **yes** | Keeps you informed of updates |
| Setup NTP? | **yes** | Critical for cluster synchronization |
| Setup RAID and XFS? | **no** (for basic setup) | Can be configured later for production |
| Enable coredumps? | **yes** | Helpful for debugging |
| System-wide configuration? | **yes** | Optimizes system performance |
| Optimize NIC and disk settings? | **yes** | Significantly improves performance |
| Enforce clocksource? | **no** | Keep current settings |
| Run iotune? | **yes** | Optimizes I/O performance |
| CPU scaling governor? | **yes** | Maximizes CPU performance |
| Enable fstrim service? | **yes** | Maintains SSD performance |
| Scylla only service? | **yes** | Dedicates resources to ScyllaDB |
| Configure remote logging? | **no** (for basic setup) | Can be configured later |
| Tune LimitNOFILES? | **yes** | Handles large number of connections |

## Step 5: Start ScyllaDB Services

On **each node**, start and enable the ScyllaDB service:

```bash
# Start the service
sudo systemctl start scylla-server

# Enable auto-start on boot
sudo systemctl enable scylla-server

# Check service status
sudo systemctl status scylla-server
```

### Verify Cluster Status

Wait a few minutes for nodes to join the cluster, then check:

```bash
nodetool status
```

Expected output:
```
Datacenter: scylla_data_center
==============================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load      Tokens Owns Host ID                              Rack
UN 192.168.1.10   662.13 KB 256    ?    e816b8b6-f8e9-40ab-85ee-69ca5dd85426 scylla_rack
UN 192.168.1.11   624.35 KB 256    ?    0dafccea-c2a5-4656-927c-5419ea05facf scylla_rack
UN 192.168.1.12   674.06 KB 256    ?    9116441e-2e91-4ed1-97b0-8b4ad9b1e9c9 scylla_rack
```

**Status indicators**:
- **UN**: Up and Normal (healthy)
- **DN**: Down and Normal (node is down)
- **UJ**: Up and Joining (node is joining cluster)

## Step 6: Security Configuration

### Connect to the Cluster

```bash
cqlsh -u cassandra -p cassandra 192.168.1.10 9042
```

### Change Default Password

**Critical security step** - change the default cassandra password:

```sql
ALTER ROLE cassandra WITH PASSWORD = 'your-secure-password-here';
```

### Create Administrative User

```sql
CREATE ROLE dbadmin WITH SUPERUSER = true AND LOGIN = true AND PASSWORD = 'admin-secure-password';
```

**Note**: ScyllaDB doesn't support hyphens in role names, use underscores instead.

### Verify Roles

```sql
LIST ROLES;
```

## Step 7: Create and Test Keyspaces

### Create Test Keyspace

```sql
CREATE KEYSPACE test_keyspace 
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
} AND durable_writes = true;
```

**Replication explanation**:
- **SimpleStrategy**: Best for single datacenter deployments
- **replication_factor: 3**: Data is replicated to all 3 nodes
- **durable_writes: true**: Ensures data durability

### Verify Keyspace Creation

```sql
DESCRIBE KEYSPACES;

-- Or alternatively:
SELECT keyspace_name FROM system_schema.keyspaces;
```

## Additional Cluster Management Commands

### Monitor Cluster Health

```bash
# Check cluster status
nodetool status

# View cluster information
nodetool info

# Check gossip information
nodetool gossipinfo

# Monitor compactions
nodetool compactionstats

# Check repairs
nodetool netstats
```

### Performance Monitoring

```bash
# View current operations
nodetool tpstats

# Check cache statistics  
nodetool info | grep -i cache

# Monitor memory usage
free -h
```

### Maintenance Commands

```bash
# Run cleanup after adding/removing nodes
nodetool cleanup

# Repair data consistency
nodetool repair

# Flush memtables to disk
nodetool flush

# View ring topology
nodetool ring
```

### Testing
```bash
cassandra-stress write n=50000 \
    -mode native cql3 user=dbadmin password='#CasndraDb@dm!nL0' \
    -node 192.168.169.44,192.168.169.5,192.168.169.4 \
    -rate threads=200 \
    -log file=/home/$(whoami)/cassandra-stress-logs/stress.log \
    -port native=9042
```


## Production Considerations

### Security Hardening

1. **Change default passwords** immediately
2. **Create dedicated application users** with limited privileges
3. **Enable SSL/TLS** for client connections
4. **Configure firewall rules** to restrict access
5. **Regular security updates**

### Performance Optimization

1. **Use dedicated SSDs** for data directories
2. **Separate commitlog** to different disk if possible
3. **Monitor and tune** memory settings
4. **Regular maintenance** (repairs, cleanups)
5. **Monitor disk space** and plan capacity

### Backup Strategy

```bash
# Create snapshot
nodetool snapshot test_keyspace

# List snapshots
nodetool listsnapshots

# Clear old snapshots
nodetool clearsnapshot
```

## Conclusion

You now have a fully functional 3-node ScyllaDB cluster with:
- ✅ Proper authentication and authorization
- ✅ Optimized system configuration  
- ✅ Network topology awareness
- ✅ Data replication across all nodes
- ✅ Basic monitoring capabilities

Remember to regularly monitor your cluster, perform maintenance tasks, and keep your ScyllaDB installation updated for optimal performance and security.
