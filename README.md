# MongoDB Replica Set Setup Guide

This guide walks you through setting up a 3-node MongoDB replica set cluster with authentication and security features.

## Prerequisites

- 3 Ubuntu/Debian servers or VMs
- Network connectivity between all nodes
- sudo access on all servers

## Server Configuration

| Node | Hostname | IP Address |
|------|----------|------------|
| Primary | mongo-1 | 192.168.56.101 |
| Secondary | mongo-2 | 192.168.56.102 |
| Secondary | mongo-3 | 192.168.56.103 |

## Installation Steps

### Step 1: Initial Setup (On All Nodes)

Update system packages and install required dependencies:

```bash
sudo apt update
sudo apt install -y wget curl gnupg
```

### Step 2: Configure Hostnames and Network

On **mongo-1**:
```bash
sudo hostnamectl set-hostname mongo-1
```

On **mongo-2**:
```bash
sudo hostnamectl set-hostname mongo-2
```

On **mongo-3**:
```bash
sudo hostnamectl set-hostname mongo-3
```

Update `/etc/hosts` on all nodes:
```bash
echo "192.168.56.101 mongo-1" | sudo tee -a /etc/hosts
echo "192.168.56.102 mongo-2" | sudo tee -a /etc/hosts
echo "192.168.56.103 mongo-3" | sudo tee -a /etc/hosts
```

### Step 3: Install MongoDB (On All Nodes)

Add MongoDB GPG key and repository:
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

Update repository and install MongoDB:
```bash
sudo apt update
sudo apt install -y mongodb-org
```

### Step 4: Configure MongoDB (On All Nodes)

Edit the MongoDB configuration file:
```bash
sudo vi /etc/mongod.conf
```

Replace the content with:
```yaml
# Network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0  # Listen on all interfaces

# Storage
storage:
  dbPath: /var/lib/mongodb

# Replication
replication:
  replSetName: "rs0"

# Security (initially disabled for setup)
security:
  keyFile: /etc/mongodb-keyfile
  authorization: disabled  # Will enable after setup

# Logging
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# Process Management
processManagement:
  fork: true
  timeZoneInfo: /usr/share/zoneinfo
```

### Step 5: Create Security Keyfile

On **mongo-1**, generate the keyfile:
```bash
openssl rand -base64 756 | sudo tee /etc/mongodb-keyfile
sudo chmod 600 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
```

Copy keyfile to other nodes:
```bash
# Prepare for copying
sudo cp /etc/mongodb-keyfile /home/vagrant/mongodb-keyfile
sudo chown vagrant:vagrant /home/vagrant/mongodb-keyfile

# Copy to other nodes
scp /home/vagrant/mongodb-keyfile vagrant@mongo-2:/tmp/
scp /home/vagrant/mongodb-keyfile vagrant@mongo-3:/tmp/
```

On **mongo-2** and **mongo-3**:
```bash
sudo mv /tmp/mongodb-keyfile /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
sudo chmod 600 /etc/mongodb-keyfile
```

### Step 6: Start MongoDB Services (On All Nodes)

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
```

## Replica Set Configuration

### Step 7: Initialize Replica Set

On **mongo-1**, connect to MongoDB and initialize the replica set:
```bash
mongosh
```

In the MongoDB shell:
```javascript
// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-1:27017" },
    { _id: 1, host: "mongo-2:27017" },
    { _id: 2, host: "mongo-3:27017" }
  ]
})

// Wait a moment, then check status
rs.status()
```

### Step 8: Create Administrative Users

Create admin user:
```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "admin123",  // Change to a secure password!
  roles: ["root"]
})
```

Test admin connection:
```bash
mongosh -u admin -p admin123 --authenticationDatabase admin
```

Create application user (optional):
```javascript
use admin
db.createUser({
  user: "mongo",
  pwd: "mongo123",    // Change to a secure password!
  roles: [
    { role: "readWrite", db: "mongo_db" }
  ]
})
```

### Step 9: Enable Authentication

Edit MongoDB configuration on all nodes:
```bash
sudo nano /etc/mongod.conf
```

Change the security section:
```yaml
security:
  keyFile: /etc/mongodb-keyfile
  authorization: enabled  # Change from disabled to enabled
```

Restart MongoDB on all nodes:
```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

## Testing the Cluster

### Connection Test

Connect to the replica set cluster:
```bash
mongosh "mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0" --username admin --password admin123 --authenticationDatabase admin
```

### Replication Test

In the MongoDB shell:
```javascript
// Check replica set status
rs.status()
db.isMaster()

// Test write operations
use testdb
db.testcollection.insertOne({message: "Hello from cluster", timestamp: new Date()})
db.testcollection.find()
```

## Security Considerations

- **Change default passwords**: Replace `admin123` and `mongo123` with strong, unique passwords
- **Network security**: Configure firewall rules to restrict access to MongoDB ports
- **SSL/TLS**: Consider enabling SSL/TLS for encrypted connections
- **Backup strategy**: Implement regular backup procedures
- **Monitoring**: Set up monitoring and alerting for the cluster

## Troubleshooting

### Common Issues

1. **Connection refused**: Check if MongoDB service is running and firewall settings
2. **Authentication failed**: Verify username, password, and authentication database
3. **Replica set issues**: Check network connectivity between nodes and keyfile permissions

### Useful Commands

```bash
# Check MongoDB logs
sudo tail -f /var/log/mongodb/mongod.log

# Check service status
sudo systemctl status mongod

# Test connectivity
telnet mongo-2 27017
```

## Maintenance

### Adding a New Node

To add a fourth node to the replica set:
```javascript
rs.add("mongo-4:27017")
```

### Removing a Node

```javascript
rs.remove("mongo-3:27017")
```

### Checking Replica Lag

```javascript
rs.printSlaveReplicationInfo()
```

## Connection Strings

- **Admin connection**: `mongodb://admin:admin123@mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0&authSource=admin`
- **Application connection**: `mongodb://mongo:mongo123@mongo-1:27017,mongo-2:27017,mongo-3:27017/mongo_db?replicaSet=rs0`

---

**Note**: This setup is intended for development/testing environments. For production deployments, implement additional security measures, monitoring, and backup strategies.