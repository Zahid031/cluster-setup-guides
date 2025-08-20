# MongoDB Replica Set Setup Guide

This guide provides two methods for setting up a 3-node MongoDB replica set cluster with authentication and security features.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Method 1: Docker Compose Setup](#method-1-docker-compose-setup)
- [Method 2: Manual VM Setup](#method-2-manual-vm-setup)
- [Testing the Cluster](#testing-the-cluster)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

## Prerequisites

### For Docker Compose
- Docker and Docker Compose installed
- At least 4GB RAM available
- Ports 27017, 27018, 27019 available

### For Manual VM Setup
- 3 Ubuntu/Debian servers or VMs
- Network connectivity between all nodes
- sudo access on all servers
- At least 1GB RAM per server

---

## Method 1: Docker Compose Setup

### Quick Start

1. **Create docker-compose.yml**
```yaml
services:
  mongo1:
    image: zahid03/my-mongo:8.0
    container_name: mongo1
    ports:
      - "27017:27017"
    volumes:
      - ./mongodb-data/mongo1:/data/db
    networks:
      - mongo-cluster
    command: >
      mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb-keyfile --auth
    restart: unless-stopped

  mongo2:
    image: zahid03/my-mongo:8.0
    container_name: mongo2
    ports:
      - "27018:27017"
    volumes:
      - ./mongodb-data/mongo2:/data/db
    networks:
      - mongo-cluster
    command: >
      mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb-keyfile --auth
    restart: unless-stopped

  mongo3:
    image: zahid03/my-mongo:8.0
    container_name: mongo3
    ports:
      - "27019:27017"
    volumes:
      - ./mongodb-data/mongo3:/data/db
    networks:
      - mongo-cluster
    command: >
      mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb-keyfile --auth
    restart: unless-stopped

networks:
  mongo-cluster:
    driver: bridge
```

2. **Setup Directory Structure**
```bash
# Create data directories
mkdir -p mongodb-data/mongo1
mkdir -p mongodb-data/mongo2  
mkdir -p mongodb-data/mongo3

# Note: If you want to use your image then you should build the image first with the your key.
```

3. **Start the Cluster**
```bash
docker-compose up -d
```

4. **Initialize Replica Set**
```bash
# Connect to primary node
docker exec -it mongo1 mongosh

# In MongoDB shell
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})

# Create admin user
use admin
db.createUser({
  user: 'admin',
  pwd: 'admin123',
  roles: ['root']
})

# Exit and test connection
exit
```

5. **Test Connection**
```bash
docker exec -it mongo1 mongosh "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0" -u admin -p admin123 --authenticationDatabase admin
```

### Docker Management Commands
```bash
# View logs
docker-compose logs -f mongo1

# Stop cluster
docker-compose down

# Restart specific service
docker-compose restart mongo1

# View all containers
docker-compose ps
```

---

## Method 2: Manual VM Setup

### Server Configuration
| Node | Hostname | IP Address |
|------|----------|------------|
| Primary | mongo-1 | 192.168.56.101 |
| Secondary | mongo-2 | 192.168.56.102 |
| Secondary | mongo-3 | 192.168.56.103 |

### Step 1: Initial Setup (All Nodes)
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

### Step 3: Install MongoDB (All Nodes)
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

### Step 4: Configure MongoDB (All Nodes)
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

### Step 6: Start MongoDB Services (All Nodes)
```bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
```

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

---

## Testing the Cluster

### Connection Test
```bash
# For Docker setup
docker exec -it mongo1 mongosh "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0" -u admin -p admin123 --authenticationDatabase admin

# For VM setup
mongosh "mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0" -u admin -p admin123 --authenticationDatabase admin
```

### Replication Test
```javascript
// Check cluster status
rs.status()
db.runCommand("isMaster")

// Test write operations
use testdb
db.testcollection.insertOne({
  message: "Hello from replica set", 
  timestamp: new Date()
})

// Verify data
db.testcollection.find().pretty()
```

### Failover Test
```bash
# Stop primary node (Docker)
docker stop mongo1

# Stop primary node (VM)
sudo systemctl stop mongod  # On current primary

# Check if secondary becomes primary
rs.status()
```

---

## Security Considerations

### Production Security Checklist
- [ ] Change default passwords to strong, unique passwords
- [ ] Enable SSL/TLS encryption
- [ ] Configure firewall rules (allow only MongoDB ports)
- [ ] Use specific user roles instead of root access
- [ ] Implement network segmentation
- [ ] Regular security updates
- [ ] Monitor authentication attempts

### Enhanced User Creation
```javascript
// Application-specific user
use admin
db.createUser({
  user: "appuser",
  pwd: "strongpassword123",
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "read", db: "analytics" }
  ]
})

// Read-only user
db.createUser({
  user: "readonly",
  pwd: "readonly123",
  roles: [{ role: "read", db: "myapp" }]
})
```

---

## Troubleshooting

### Common Issues

**Connection Refused**
```bash
# Check service status
docker ps  # For Docker
sudo systemctl status mongod  # For VM

# Check logs
docker logs mongo1  # For Docker
sudo tail -f /var/log/mongodb/mongod.log  # For VM
```

**Authentication Failed**
```bash
# Verify user exists
use admin
db.getUsers()

# Check connection string format
mongosh "mongodb://username:password@host:port/database?authSource=admin"
```

**Replica Set Issues**
```javascript
// Check replica set configuration
rs.conf()
rs.status()

// Force reconfigure if needed
rs.reconfig(config, {force: true})
```

### Health Check Commands
```bash
# Network connectivity
telnet mongo-2 27017

# MongoDB process
pgrep mongod

# Port listening
netstat -tlnp | grep 27017
```

---

## Maintenance

### Adding New Node
```javascript
// Add 4th node
rs.add("mongo-4:27017")
rs.status()
```

### Removing Node
```javascript
// Remove node
rs.remove("mongo-3:27017")
```

### Checking Replica Lag
```javascript
rs.printSlaveReplicationInfo()
```

### Backup Strategy
```bash
# Create backup
mongodump --host="mongo1:27017" --username="admin" --password="admin123" --authenticationDatabase="admin" --out=/backup/

# Restore backup
mongorestore --host="mongo1:27017" --username="admin" --password="admin123" --authenticationDatabase="admin" /backup/
```

### Monitoring Commands
```javascript
// Check replication lag
rs.printSecondaryReplicationInfo()

// Database statistics
db.stats()

// Connection statistics  
db.serverStatus().connections
```

---

## Connection Strings

### Docker Setup
```bash
# Admin connection
mongodb://admin:admin123@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin

# Application connection
mongodb://appuser:password@localhost:27017,localhost:27018,localhost:27019/myapp?replicaSet=rs0
```

### VM Setup
```bash
# Admin connection
mongodb://admin:admin123@mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0&authSource=admin

# Application connection  
mongodb://appuser:password@mongo-1:27017,mongo-2:27017,mongo-3:27017/myapp?replicaSet=rs0
```

---

## Performance Tips

- **Memory**: Allocate sufficient RAM for working set
- **Storage**: Use SSD storage for better I/O performance  
- **Network**: Ensure low latency between replica set members
- **Indexes**: Create appropriate indexes for query patterns
- **Monitoring**: Set up MongoDB Compass or ops manager

---

**Note**: This setup provides a foundation for MongoDB replica sets. For production environments, implement additional monitoring, backup strategies, and security hardening measures.