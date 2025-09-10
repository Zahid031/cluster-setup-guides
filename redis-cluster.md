#!/bin/bash

# Install Redis
sudo apt update
sudo apt install -y redis-server


#latest redis install


sudo apt-get update
sudo apt-get install lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis



# Create data dirs
sudo mkdir -p /var/lib/redis/6380 /var/lib/redis/6381
sudo chown -R redis:redis /var/lib/redis

# Copy base configs
sudo cp /etc/redis/redis.conf /etc/redis/redis-6380.conf
sudo cp /etc/redis/redis.conf /etc/redis/redis-6381.conf

# Edit configs with sed
sudo sed -i 's/^port .*/port 6380/' /etc/redis/redis-6380.conf
sudo sed -i 's|^dir .*|dir /var/lib/redis/6380|' /etc/redis/redis-6380.conf
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

sudo sed -i 's/^port .*/port 6381/' /etc/redis/redis-6381.conf
sudo sed -i 's|^dir .*|dir /var/lib/redis/6381|' /etc/redis/redis-6381.conf
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

# Start Redis instances
sudo redis-server /etc/redis/redis-6380.conf
sudo redis-server /etc/redis/redis-6381.conf







redis-cli --cluster create \
 192.168.56.21:6380 192.168.56.22:6380 192.168.56.23:6380 \
  192.168.56.22:6381 192.168.56.23:6381 192.168.56.21:6381 \
  --cluster-replicas 1





redis-cli -c -p 6380 cluster nodes
redis-cli -c -p 6380 cluster info


redis-cli -c -h 192.168.56.31 -p 6380 cluster nodes
redis-cli -c -h 192.168.56.32 -p 6380 cluster info


#write performence
#redis-benchmark -h 192.168.56.31 -p 6380 -c 100 -n 10000 -t set

redis-benchmark -t set -n 100000 -r 1000000
redis-benchmark --cluster -h 192.168.56.21 -p 6380 -c 100 -n 10000 -t set -r 1000000


#read performence
redis-benchmark -h 192.168.56.31 -p 6380 -c 100 -n 10000 -t get

#shutdown
redis-cli -h 192.168.56.22 -p 6380 shutdown


pip install redis-py-cluster


#REDIS_HOST=192.168.169.147
#REDIS_PORT=6379
#REDIS_DATABASE=0
#REDIS_PASSWORD='nhR2yAY04Zofd45'




