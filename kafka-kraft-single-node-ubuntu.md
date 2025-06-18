```markdown
## Kafka Single Node Installation and Configuration

### Prerequisites

-   Java Development Kit (JDK) 11 or later

### Installation Steps

1.  **Update Package Index and Install JDK**

    ```bash
    sudo apt update
    sudo apt install openjdk-11-jdk -y
    java -version
    ```

2.  **Create Kafka User**

    ```bash
    sudo adduser --no-create-home --shell=/sbin/nologin --disabled-password --disabled-login --gecos "" kafka
    ```

3.  **Download and Extract Kafka**

    ```bash
    cd /opt
    sudo wget https://downloads.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
    sudo tar -xvzf kafka_2.13-3.9.0.tgz
    sudo mv kafka_2.13-3.9.0 kafka
    sudo chown -R kafka:kafka /opt/kafka
    ```

4.  **Create Log Directories and Set Permissions**

    ```bash
    sudo mkdir -p /tmp/kraft-combined-logs
    sudo chown -R kafka:kafka /tmp/kraft-combined-logs

    sudo mkdir -p /var/log/kafka
    sudo chown -R kafka:kafka /var/log/kafka
    ```

5.  **Generate a UUID**

    ```bash
    /opt/kafka/bin/kafka-storage.sh random-uuid
    ```

6.  **Format Storage**

    ```bash
    /opt/kafka/bin/kafka-storage.sh format -t <UUID> -c /opt/kafka/config/kraft/server.properties
    ```

    *Replace `<UUID>` with the UUID generated in the previous step.*

7.  **Create a Systemd Service File**

    ```bash
    sudo nano /etc/systemd/system/kafka.service
    ```

    Add the following content:

    ```service
    [Unit]
    Requires=network.target
    After=network.target

    [Service]
    Type=simple
    User=kafka
    Environment=KAFKA_HEAP_OPTS="-Xmx512M -Xms256M"
    Environment=KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent"
    Environment="LOG_DIR=/var/log/kafka"
    ExecStart=/bin/bash -c '/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties'
    ExecStop=/opt/kafka/bin/kafka-server-stop.sh
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

8.  **Start Kafka Service**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable kafka
    sudo systemctl start kafka
    sudo systemctl status kafka
    ```

    If the service fails to start, check the logs:

    ```bash
    sudo journalctl -u kafka.service -e
    ```

9.  **Verify Kafka is Running**

    ```bash
    netstat -tulnp | grep 9092
    ```

### Kafka Topic Operations

1.  **Create a Topic**

    ```bash
    /opt/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
    ```

2.  **Produce Messages**

    ```bash
    /opt/kafka/bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
    ```

3.  **Consume Messages**

    ```bash
    /opt/kafka/bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
    ```
```
