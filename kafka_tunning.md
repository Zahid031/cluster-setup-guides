# General Broker Settings

# Allow more concurrent connections
num.network.threads=8
num.io.threads=16
queued.max.requests=1000

# Disk I/O parallelism (depends on SSD, you can go high)
num.replica.fetchers=4

# Larger log segments reduce metadata churn
log.segment.bytes=1073741824   # 1GB
log.segment.delete.delay.ms=60000


# Replication & Reliability
# Safe but fast enough
min.insync.replicas=2
unclean.leader.election.enable=false ***
replica.lag.time.max.ms=30000


# Reduce rebalance delays for faster consumer startup
group.initial.rebalance.delay.ms=0 *** immidiate rebalance


# Topic-level tuning

partitions: For high throughput, start with 12–24 partitions per topic.

Location updates: high throughput → more partitions. ***

Payment events: critical, lower partitions (e.g., 3–6). ***

replication.factor=3 (you already set 👍).

min.insync.replicas=2 → ensures availability + durability. ***

# JVM
-Xms2G -Xmx2G -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M


KAFKA_HEAP_OPTS="-Xmx4G -Xms4G"

swapoff -a


# Linux sysctl tuning (/etc/sysctl.conf):
# sudo nano /etc/sysctl.d/99-kafka.conf

vm.swappiness=1  #Tells Linux to avoid swapping as much as possible.
vm.dirty_ratio=40
vm.dirty_background_ratio=10
net.core.wmem_max=26214400
net.core.rmem_max=26214400
net.ipv4.tcp_fin_timeout=30
net.ipv4.tcp_keepalive_time=60


# Kafka tuning
net.core.somaxconn = 1024  #65535
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_tw_reuse = 1
net.core.netdev_max_backlog = 3000
vm.swappiness = 1
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.max_map_count = 262144



# Practical Setup for Ride-sharing (like Uber/Pathao)

High-volume topics (e.g., location-updates):

24 partitions, replication=3, min.insync=2.

Medium-volume topics (e.g., trip-events):

12 partitions, replication=3.

Critical topics (e.g., payments):

6 partitions, replication=3, acks=all








############################
# Node Identity & Roles
############################
node.id=1                           # Change per broker (1,2,3)
process.roles=broker,controller

############################
# Networking
############################
controller.listener.names=CONTROLLER
listeners=BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
listener.security.protocol.map=BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT
inter.broker.listener.name=BROKER
advertised.listeners=BROKER://192.168.56.51:9092   # Change per broker

############################
# Quorum (all controllers)
############################
controller.quorum.voters=1@192.168.56.51:9093,2@192.168.56.52:9093,3@192.168.56.53:9093

############################
# Storage (data & metadata)
############################
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/meta
log.segment.bytes=1073741824             # 1GB log segments
log.segment.delete.delay.ms=60000        # 60s delay before deletion

############################
# Broker Performance
############################
num.network.threads=8
num.io.threads=16
num.replica.fetchers=4
queued.max.requests=1000

############################
# Topic Defaults
############################
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

############################
# Internal Topics
############################
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2  *****

############################
# Consumer Group
############################
group.initial.rebalance.delay.ms=0

############################
# Safety & Convenience
############################
auto.create.topics.enable=false






# Kafka cluster tuning — files and explanations

This README explains the files provided above and why key lines are set. Use this as a guide when you copy files onto each of the three nodes.

---

## server.properties (KRaft combined broker+controller)
- `process.roles=broker,controller`
  Runs both controller and broker on the same process. For small clusters this is convenient; for larger production, run dedicated controllers.

- `node.id=1`
  Unique integer id for each node; set to 1, 2, 3 on the three machines.

- `listeners=...` and `advertised.listeners=...`
  Address and ports Kafka listens on, and the addresses clients use to reach this broker. Replace the IP to the node’s reachable IP.

- `controller.quorum.voters=1@192.168.56.51:9093,2@192.168.56.52:9093,3@192.168.56.53:9093`
  KRaft controller quorum — list all controllers with node IDs and controller port. All three should be identical across nodes.

- `log.dirs` / `metadata.log.dir`
  Physical locations for topic data and KRaft metadata. Put on the fastest available disk; separate from OS if possible.

- `log.segment.bytes=1073741824` (1GB)
  Larger segments reduce metadata churn (fewer segment files) while slightly increasing compaction/segment rewrite cost. 1 GB is a reasonable starting point for moderate throughput.

- `log.segment.delete.delay.ms=60000`
  Delay before segment deletion; prevents immediate deletion race conditions.

- `num.network.threads=8`, `num.io.threads=16`
  Threads for networking and I/O. You have 4 vCPUs — these values are intentionally above CPU count to allow concurrency; monitor CPU and adjust. If CPU becomes saturated, lower these.

- `num.replica.fetchers=4`
  Parallel fetcher threads for replication; helps replication throughput.

- `queued.max.requests=1000`
  Max number of requests queued; if you see request rejections or high queue, tune.

- `socket.*` entries
  Increase socket buffers for throughput.

- `num.partitions=3` / `default.replication.factor=3` / `min.insync.replicas=2`
  Defaults for topics: replication factor 3 for availability, min ISR 2 to ensure safe durability (can accept one broker failure and still produce if acks=all).

- `unclean.leader.election.enable=false`
  Prevents unclean leader elections which can cause data loss. Keeps durability high at the cost of potential short unavailability during many failures.

- `group.initial.rebalance.delay.ms=0`
  Consumer groups will rebalance immediately — useful for low-latency consumer startup. Be cautious if you have many consumer joins at once.

- `offsets.topic.replication.factor=3` and transaction-related settings
  Ensure internal topics are replicated for durability.

- `auto.create.topics.enable=false`
  Prevents accidental topic creation; recommended for production. Create topics explicitly with tuned partitions/replication.

---

## kafka-env.sh (JVM / heap)
- `KAFKA_HEAP_OPTS="-Xms2G -Xmx2G ..."`
  Sets Java heap. On an 8 GB host we set 2G to leave RAM for OS page cache and other processes. Kafka benefits greatly from OS page cache; do not give Java all memory. If you expect heavier broker workload, you can increase to 3G but avoid >50% of host RAM.

- `UseG1GC` and related flags
  G1GC is sensible on modern Java versions; `MaxGCPauseMillis=20` aims for small pauses. Monitor GC logs and adjust.

---

## /etc/systemd/system/kafka.service.d/limits.conf and /etc/security/limits.d/kafka.conf
- `LimitNOFILE=100000` and `nofile 100000`
  Kafka opens many files (segment files, sockets). Raise file descriptor limits to avoid EMFILE errors.

- `nproc` increases allowed processes/threads if Kafka or its helper processes need them.

Note: systemd limits are used when starting via systemd; `/etc/security/limits.d/` affects interactive sessions and non-systemd startup. Set both to be safe.

---

## /etc/sysctl.d/99-kafka.conf (kernel parameters)
### Networking
- `net.core.somaxconn = 1024`
  Increase listen backlog to allow more TCP connections to queue.

- `net.ipv4.tcp_max_syn_backlog = 4096`
  Help with many incoming connection attempts.

- `net.core.netdev_max_backlog = 3000`
  Queue length for packets in the driver before kernel hands them to the network stack.

- `net.ipv4.tcp_tw_reuse = 1` and `tcp_fin_timeout = 30`
  Speed up socket reuse in high churn environments.

- `net.core.rmem_max` / `wmem_max`
  Increase socket buffer upper bounds for high throughput.

### Memory
- `vm.swappiness = 1`
  Prefer using RAM and page cache; avoid swapping. If you completely disable swap (`swapoff -a` and remove from /etc/fstab), this parameter is less relevant. Many teams leave a small swap enabled and set swappiness low as a safety valve.

- `vm.max_map_count = 262144`
  Large number of memory-mapped files can occur with many segment files; increase if you see mmap-related errors.

- `vm.dirty_ratio = 10` and `vm.dirty_background_ratio = 5`
  Lower values reduce how much dirty data the kernel can hold in RAM before writeback; Kafka writes data to disk and OS handles flushing — keeping these moderate helps keep latency bounded.

- `fs.file-max = 2097152`
  Maximum number of file handles system-wide; raises the system cap.

---

## Filesystem & mount options
- `XFS` recommended for Kafka log volumes; mount with `noatime,nodiratime` to cut metadata writes and improve throughput.

---

## Swap: `swapoff -a` vs `vm.swappiness=1`
- **If you permanently disable swap** (swapoff + remove swap from `/etc/fstab`), the kernel will have no swap device — `vm.swappiness` then has no effect. Disabling swap entirely eliminates the chance of the OS swapping your JVM, but also removes the small safety net against OOM if memory spikes.
- **If you keep a small swap file**, set `vm.swappiness=1` to make the kernel avoid swapping except as a last resort. This is often recommended for latency-sensitive services like Kafka.

---

## Tuning strategy & monitoring
- Start with these conservative settings, then load-test with traffic representative of your arrival patterns (small many writes vs large batches).
- Monitor CPU, GC pause times, disk IOPS/latency, network saturation, replication under-replicated partitions, and OS-level metrics (iostat, vmstat, sar).
- Adjust `num.io.threads`, `num.network.threads`, and heap based on observed CPU and GC behaviour. If CPU is saturated, reduce threads or increase CPU resources. If GC pauses are high, tune heap/GC flags.

---

## Per-node changes
- Change `node.id` and `advertised.listeners` on each node (IDs 1,2,3 and their IPs). The `controller.quorum.voters` line must list all three controllers with the same entries on each node.

---

## Quick checklist to apply
1. Copy `server.properties` to Kafka config dir on each node; update node.id and advertised.listeners accordingly.
2. Create `kafka-env.sh` (or set the same env vars in your packaging); ensure startup scripts source it.
3. Add `99-kafka.conf` to `/etc/sysctl.d/` and run `sudo sysctl --system`.
4. Add file limits and systemd override, reload systemd, restart Kafka.
5. Ensure data mount uses XFS (or ext4 with noatime) and has enough space.
6. Load-test and monitor; iterate.

---

If you want, I can:
- produce per-node `server.properties` with node.id and IPs already filled (give me your three node IPs if you want), **or**
- create an `ansible` playbook that applies the sysctl/limits/mount changes and deploys the configs to all three nodes.

Which would you like next?






