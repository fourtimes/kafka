# Setting up Kafka with Kraft in AWS

## Tech spec

    - 3 EC2 instances, t2.medium, Ubuntu 22.04
    - Kafka 3.9.0 for scala 2.13

Download the kafka 3.9.0 package - https://kafka.apache.org/downloads

## Setting up infra

    - Create a security group called kafka. The required ports are 9091, 9092, 9093. The rules in the security group should allow any member of the security group to access these ports.
    - Create 3 Ubuntu 22.04 EC2 instances. You can just select Ubuntu when creating the EC2 instances.
    - Attach the security group from step 1 to each instance. You can do that when creating the instances.

### Preparing the nodes

Below are a few commands to prepare the environment for Kafka installation and config. This should be done on each node:
```md
# update and install zip and jdk
sudo apt-get update
sudo apt-get install unzip
sudo apt-get install default-jdk -y

# create kafka user
sudo adduser --no-create-home  --shell=/sbin/nologin --disabled-password --disabled-login --gecos "" kafka

# create necessary dirs
sudo mkdir /data
sudo mkdir -p /var/log/kafka
sudo mkdir -p /etc/kafka
sudo chown -R kafka:kafka /data
sudo chown -R kafka:kafka /var/log/kafka

# download and extract kafka
wget https://downloads.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
sudo mv kafka_2.13-3.9.0.tgz /etc/kafka
sudo tar -xzvf /etc/kafka/kafka_2.13-3.9.0.tgz -C /etc/kafka --strip 1
sudo chown -R kafka:kafka /etc/kafka
```
## Installing and configuring Kafka Kraft
### 1. Generate random cluster uuid
Kraft mode requires a cluster uuid. You can create one by running the command:
```sh
/etc/kafka/bin/kafka-storage.sh random-uuid # output - ftpUeuPCQOWiqpcnG8Mqag
```
Save the uuid for later.

### 2. Configure Kraft server.properties
First, note the private IP of each node. Then, with your preferred editor edit **`/etc/kafka/config/kraft/server.properties`**.

The properety **`node_id`** should be unique and incremental for each node. make sure to associate the IP of each node with its ID. It’s required for the **`controller.quorum.voters`** property.

**Note:**
1) node-1-private-ip = 172.31.43.210
2) node-2-private-ip = 172.31.47.170
3) node-3-private-ip = 172.31.36.217

```properties
# pass the values kafka-node-1 configuration file - /etc/kafka/config/kraft/server.properties
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
```properties
# pass the values kafka-node-2 configuration file - /etc/kafka/config/kraft/server.properties
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
```properties
# pass the values kafka-node-3 configuration file - /etc/kafka/config/kraft/server.properties
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
### 3. Configure Kafka systemd service

Create a systemd service file at **`/etc/systemd/system/kafka.service`** for each node. This step requires the cluster UUID - **`ftpUeuPCQOWiqpcnG8Mqag`** that was created earlier.
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

### 4. Start Kafka and check its status
```
sudo systemctl daemon-reload
sudo systemctl enable kafka.service
sudo systemctl start kafka.service
sudo systemctl status kafka.service
```
## Testing and implementation
### 1. Topic
```md
# Create the topic
/etc/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic quick-events-1 --create --partitions 2 --replication-factor 2
Created topic messages.

# List the topics
/etc/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic quick-events-1 --list
```
### 2. Insert the messages
```
/etc/kafka/bin/kafka-console-producer.sh --topic quick-events-1 --bootstrap-server 172.31.43.210:9092,172.31.47.170:9092,172.31.36.217:9092
```
### 3. List the messages from beginning itself
```
/etc/kafka/bin/kafka-console-consumer.sh --topic quick-events-1 --bootstrap-server  172.31.43.210:9092,172.31.47.170:9092,172.31.36.217:9092 --from-beginning
```

---

**REFERENCE** 
- https://medium.com/@vonschnappi/setting-up-kafka-with-kraft-in-aws-837310466591
- https://blog.codefarm.me/2024/01/12/install-kafka-with-kraft/

