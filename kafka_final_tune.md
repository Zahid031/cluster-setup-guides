sudo nano /etc/sysctl.d/99-kafka.conf

vm.swappiness = 1                 # Avoid swapping; keep JVM in RAM
vm.max_map_count = 262144         # Needed for mmap of log segments

# Disk writeback tuning
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10

# Networking tuning
net.core.somaxconn = 65535         # Max listen backlog (default 128)
net.core.netdev_max_backlog = 16384  # Max packets in NIC backlog

net.ipv4.tcp_max_syn_backlog = 8192  # Pending TCP connections
net.ipv4.tcp_tw_reuse = 1            # Reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout = 15        # Faster socket close
net.ipv4.tcp_keepalive_time = 60    # Keepalive to detect dead clients



# Socket buffer sizes
net.core.rmem_max = 134217728    # 128 MB
net.core.wmem_max = 134217728    # 128 MB
net.ipv4.tcp_rmem = 4096 87380 268435456
net.ipv4.tcp_wmem = 4096 65536 268435456



./kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server 192.168.61.146:9092 \
  --partitions 3 \
  --replication-factor 3



/opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic test-topic \
  --num-records 1000000 \
  --record-size 100 \
  --throughput 1000 \
  --producer-props bootstrap.servers=192.168.61.146:9092



  /opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic large-test-topic \
  --num-records 100000 \
  --record-size 1048576 \
  --throughput 100 \
  --producer-props bootstrap.servers=192.168.61.146:9092



  /opt/kafka/bin/kafka-consumer-perf-test.sh \
  --topic test-topic \
  --messages 1000000 \
  --bootstrap-server 192.168.61.146:9092 \
  --group test-group


  /opt/kafka/bin/kafka-consumer-perf-test.sh \
  --topic large-test-topic \
  --messages 100000 \
  --bootstrap-server 192.168.61.146:9092 \
  --group large-test-group



#512kb 6000
  /opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic large-test-topic \
  --num-records 6000 \
  --record-size 524288 \
  --throughput -1 \
  --producer-props bootstrap.servers=192.168.61.146:9092 acks=1 batch.size=65536 linger.ms=5 compression.type=snappy
2307 records sent, 461.0 records/sec (230.52 MB/sec), 9.5 ms avg latency, 489.0 ms max latency.
2671 records sent, 534.2 records/sec (267.10 MB/sec), 5.9 ms avg latency, 113.0 ms max latency.
6000 records sent, 501.6 records/sec (250.79 MB/sec), 7.00 ms avg latency, 489.00 ms max latency, 4 ms 50th, 14 ms 95th, 87 ms 99th, 183 ms 99.9th.




with config file:
/opt/kafka/bin/kafka-producer-perf-test.sh   --topic large-test-topic   --num-records 10000   --record-size 524288   --throughput -1   --producer.config /etc/kafka/producer.properties
2206 records sent, 441.2 records/sec (220.60 MB/sec), 7.8 ms avg latency, 452.0 ms max latency.
2460 records sent, 491.8 records/sec (245.90 MB/sec), 11.1 ms avg latency, 386.0 ms max latency.
2391 records sent, 478.2 records/sec (239.10 MB/sec), 14.4 ms avg latency, 222.0 ms max latency.
2403 records sent, 480.5 records/sec (240.25 MB/sec), 7.8 ms avg latency, 252.0 ms max latency.
10000 records sent, 474.3 records/sec (237.17 MB/sec), 10.04 ms avg latency, 452.00 ms max latency, 5 ms 50th, 28 ms 95th, 128 ms 99th, 313 ms 99.9th.



100 kb size 100000 files replication 6:

/opt/kafka/bin/kafka-producer-perf-test.sh   --topic large-test-topic   --num-records 100000   --record-size 100000   --throughput -1   --producer.config /etc/kafka/producer.properties
12702 records sent, 2537.9 records/sec (242.03 MB/sec), 17.6 ms avg latency, 459.0 ms max latency.
14110 records sent, 2821.4 records/sec (269.07 MB/sec), 13.2 ms avg latency, 147.0 ms max latency.
13603 records sent, 2718.4 records/sec (259.25 MB/sec), 40.5 ms avg latency, 734.0 ms max latency.
5973 records sent, 1193.6 records/sec (113.83 MB/sec), 94.1 ms avg latency, 2760.0 ms max latency.
10117 records sent, 2023.0 records/sec (192.93 MB/sec), 121.3 ms avg latency, 4768.0 ms max latency.
7125 records sent, 1423.9 records/sec (135.79 MB/sec), 96.9 ms avg latency, 3536.0 ms max latency.
5760 records sent, 1141.7 records/sec (108.88 MB/sec), 301.3 ms avg latency, 5676.0 ms max latency.
9852 records sent, 1970.4 records/sec (187.91 MB/sec), 94.2 ms avg latency, 3282.0 ms max latency.
9743 records sent, 1946.3 records/sec (185.61 MB/sec), 161.7 ms avg latency, 3952.0 ms max latency.
9813 records sent, 1962.2 records/sec (187.13 MB/sec), 121.9 ms avg latency, 2286.0 ms max latency.
100000 records sent, 1956.0 records/sec (186.53 MB/sec), 91.63 ms avg latency, 5676.00 ms max latency, 8 ms 50th, 261 ms 95th, 2627 ms 99th, 5084 ms 99.9th.


100 kb size 100000 files replication 3:

/opt/kafka/bin/kafka-producer-perf-test.sh   --topic test-topic   --num-records 100000   --record-size 100000   --throughput -1   --producer.config /etc/kafka/producer.properties
12596 records sent, 2518.7 records/sec (240.20 MB/sec), 12.4 ms avg latency, 416.0 ms max latency.
14638 records sent, 2927.6 records/sec (279.20 MB/sec), 12.2 ms avg latency, 566.0 ms max latency.
15154 records sent, 3030.8 records/sec (289.04 MB/sec), 9.3 ms avg latency, 128.0 ms max latency.
13788 records sent, 2756.5 records/sec (262.88 MB/sec), 16.7 ms avg latency, 884.0 ms max latency.
13450 records sent, 2688.9 records/sec (256.44 MB/sec), 58.4 ms avg latency, 2779.0 ms max latency.
14228 records sent, 2843.9 records/sec (271.21 MB/sec), 10.6 ms avg latency, 346.0 ms max latency.
14284 records sent, 2856.8 records/sec (272.45 MB/sec), 13.5 ms avg latency, 233.0 ms max latency.
100000 records sent, 2808.9 records/sec (267.88 MB/sec), 20.76 ms avg latency, 2779.00 ms max latency, 7 ms 50th, 36 ms 95th, 174 ms 99th, 2125 ms 99.9th.





  du -sh /var/lib/kafka/data/*
