# Kafka Kraft Setup on AWS

This document outlines the steps to set up a Kafka cluster using Kraft mode on AWS.

## Prerequisites

*   3 EC2 instances (t2.medium, Ubuntu 22.04 recommended)
*   Kafka 3.9.0 (Scala 2.13)

## Infrastructure Setup

1.  **Security Group:** Create a security group named `kafka`. Allow inbound traffic on ports 9091, 9092, and 9093 from within the security group.
2.  **EC2 Instances:** Launch 3 Ubuntu 22.04 EC2 instances.
3.  **Attach Security Group:** Assign the `kafka` security group to each instance.

## Node Preparation

Perform these steps on each EC2 instance:

```bash
# Update and install dependencies
sudo apt-get update
sudo apt-get install -y unzip default-jdk

# Create kafka user
sudo adduser --no-create-home --shell=/sbin/nologin --disabled-password --disabled-login --gecos "" kafka

# Create directories
sudo mkdir -p /data /var/log/kafka /etc/kafka
sudo chown -R kafka:kafka /data /var/log/kafka

# Download and extract Kafka
wget https://downloads.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
sudo mv kafka_2.13-3.9.0.tgz /etc/kafka
sudo tar -xzvf /etc/kafka/kafka_2.13-3.9.0.tgz -C /etc/kafka --strip 1
sudo chown -R kafka:kafka /etc/kafka
```

## Kafka Kraft Configuration

### 1. Generate Cluster UUID

Generate a UUID for the Kafka cluster:

```bash
/etc/kafka/bin/kafka-storage.sh random-uuid
```

Save this UUID.

### 2. Configure `server.properties`


Edit `/opt/kafka/config/kraft/server.properties` on each node.  Pay close attention to the following properties, adjusting them for each node:

*   `node.id`:  A unique integer for each node in the cluster (e.g., 1, 2, 3).
*   `controller.quorum.voters`: A comma-separated list of node IDs and their corresponding host:port addresses for the controller quorum.  This should be the same on all nodes.  Example: `1@kafka1:9093,2@kafka2:9093,3@kafka3:9093`  (Replace `kafka1`, `kafka2`, `kafka3` with the actual hostnames or IP addresses of your servers).
*   `listeners`:  The address that Kafka brokers bind to.  Use the hostname or IP address of the current node.  Example: `PLAINTEXT://kafka1:9092`
*   `advertised.listeners`: The address that clients use to connect to the Kafka brokers.  Use the hostname or IP address of the current node. Example: `PLAINTEXT://kafka1:9092`
*   `controller.listener.names`: `controller`
*   `inter.broker.listener.name`: `PLAINTEXT`


**Note:**

    node-1-private-ip = 172.31.43.210
    node-2-private-ip = 172.31.47.170
    node-3-private-ip = 172.31.36.217


Edit `/etc/kafka/config/kraft/server.properties` on each node.  Replace the example IPs with your instance's private IPs.

*   **Node 1:**

    ```properties
    node.id=1
    controller.quorum.voters=1@172.31.43.210:9093,2@172.31.47.170:9093,3@172.31.36.217:9093
    listeners=PLAINTEXT://:9092,CONTROLLER://:9093
    inter.broker.listener.name=PLAINTEXT
    advertised.listeners=PLAINTEXT://172.31.43.210:9092
    controller.listener.names=CONTROLLER
    log.dirs=/data/kraft-combined-logs
    listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
    log.retention.hours=-1
    log.retention.bytes=-1
    ```

*   **Node 2:**

    ```properties
    node.id=2
    controller.quorum.voters=1@172.31.43.210:9093,2@172.31.47.170:9093,3@172.31.36.217:9093
    listeners=PLAINTEXT://:9092,CONTROLLER://:9093
    inter.broker.listener.name=PLAINTEXT
    advertised.listeners=PLAINTEXT://172.31.47.170:9092
    controller.listener.names=CONTROLLER
    log.dirs=/data/kraft-combined-logs
    listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
    log.retention.hours=-1
    log.retention.bytes=-1
    ```

*   **Node 3:**

    ```properties
    node.id=3
    controller.quorum.voters=1@172.31.43.210:9093,2@172.31.47.170:9093,3@172.31.36.217:9093
    listeners=PLAINTEXT://:9092,CONTROLLER://:9093
    inter.broker.listener.name=PLAINTEXT
    advertised.listeners=PLAINTEXT://172.31.36.217:9092
    controller.listener.names=CONTROLLER
    log.dirs=/data/kraft-combined-logs
    listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
    log.retention.hours=-1
    log.retention.bytes=-1
    ```

### 3. Configure Systemd Service

Create `/etc/systemd/system/kafka.service` on each node, replacing `ftpUeuPCQOWiqpcnG8Mqag` with your generated UUID:

```service
[Unit]
Requires=network.target
After=network.target

[Service]
Type=simple
User=kafka
Environment=KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
Environment=KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent"
ExecStartPre=/bin/bash -c '/etc/kafka/bin/kafka-storage.sh format -t ftpUeuPCQOWiqpcnG8Mqag -c /etc/kafka/config/kraft/server.properties --ignore-formatted'
ExecStart=/bin/bash -c '/etc/kafka/bin/kafka-server-start.sh /etc/kafka/config/kraft/server.properties'
ExecStop=/etc/kafka/bin/kafka-server-stop.sh
Environment="LOG_DIR=/var/log/kafka"
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 4. Start Kafka

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka.service
sudo systemctl start kafka.service
sudo systemctl status kafka.service
```

## Testing

### 1. Create a Topic 

```bash
/etc/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic quick-events-1 --create --partitions 2 --replication-factor 2
```

### 2. Produce Messages (insert the messages)

```bash
/etc/kafka/bin/kafka-console-producer.sh --topic quick-events-1 --bootstrap-server 172.31.43.210:9092,172.31.47.170:9092,172.31.36.217:9092
```

### 3. Consume Messages (List the messages from beginning itself)

```bash
/etc/kafka/bin/kafka-console-consumer.sh --topic quick-events-1 --bootstrap-server  172.31.43.210:9092,172.31.47.170:9092,172.31.36.217:9092 --from-beginning
```
### 4. List the topics
```bash
/etc/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
/etc/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic quick-events-1 --list
```
## References

*   [https://medium.com/@vonschnappi/setting-up-kafka-with-kraft-in-aws-837310466591](https://medium.com/@vonschnappi/setting-up-kafka-with-kraft-in-aws-837310466591)
*   [https://blog.codefarm.me/2024/01/12/install-kafka-with-kraft/](https://blog.codefarm.me/2024/01/12/install-kafka-with-kraft/)

