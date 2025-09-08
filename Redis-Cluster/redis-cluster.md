sudo apt update
sudo apt install redis-server -y

sudo cp /etc/redis/redis.conf /etc/redis/redis-6380.conf


port 6380
daemonize yes
pidfile /var/run/redis_6380.pid
logfile /var/log/redis/redis-server-6380.log
dir /var/lib/redis/6380
cluster-enabled yes
cluster-config-file nodes-6380.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-6380.aof"
bind 0.0.0.0
protected-mode no


sudo cp /etc/redis/redis.conf /etc/redis/redis-6381.conf

port 6381
daemonize yes
pidfile /var/run/redis_6381.pid
logfile /var/log/redis/redis-server-6381.log
dir /var/lib/redis/6381
cluster-enabled yes
cluster-config-file nodes-6381.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-6381.aof"
bind 0.0.0.0
protected-mode no

sudo mkdir -p /var/lib/redis/6380
sudo mkdir -p /var/lib/redis/6381
sudo chown -R redis:redis /var/lib/redis


sudo redis-server /etc/redis/redis-6380.conf
sudo redis-server /etc/redis/redis-6381.conf

create the cluster:
redis-cli --cluster create \
  IP_A:6380 IP_B:6380 IP_C:6380 \
  IP_B:6381 IP_C:6381 IP_A:6381 \
  --cluster-replicas 1
  
  
  
verify:
# Connect to the first primary on Node A
redis-cli -c -h IP_A -p 6380

# Once connected, run:
CLUSTER NODES
