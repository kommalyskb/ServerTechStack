# Kafka for Production
ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server Zookeeper 2 ຫນ່ວຍ
- Server Kafka 2 ຫນ່ວຍ

ທຸກໜ່ວຍຕ້ອງຕິດຕັ້ງ Docker ແລະ Docker compose
```
sudo apt install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
Here is a Docker Compose file that can be used to set up Apache Kafka in a production environment with high availability:
## Zookeeper
ຕ້ອງຕິດຕັ້ງໃສ່ ທັງ 2 ໜ່ວຍ
```
version: '2'

services:
  zookeeper:
    image: bitnami/zookeeper
    hostname: zookeeper-1
    container_name: zookeeper-1
    volumes:
      - ./data:/bitnami/zookeeper
    ports:
      - "2181:2181"
      - "2888:2888"
      - "3888:3888"
    extra_hosts:
      - "zookeeper-1:10.1.1.1"
      - "zookeeper-2:10.1.1.2"
      - "kafka-1:10.1.1.3"
      - "kafka-2:10.1.1.4"
      - "kafka-3:10.1.1.5"
    environment:
      - ZOO_MY_ID=<zookeeper-id>
      - ALLOW_ANONYMOUS_LOGIN=yes
      - ZOO_PORT=2181
      - ZOO_SERVERS=server.1=<zookeeper-1>:2888:3888 server.2=<zookeeper-2>:2888:3888
```

## Kafka
ຕ້ອງຕິດຕັ້ງໃສ່ທັງ 2 ໜ່ວຍ
```
version: '3'

services:
  kafka:
    image: bitnami/kafka
    hostname: kafka-1
    container_name: kafka-1
    ports:
      - "9092:9092"
      - "9093:9093"
    volumes:
    - ./data:/bitnami/kafka
    extra_hosts:
      - "zookeeper-1:10.1.1.1"
      - "zookeeper-2:10.1.1.2"
      - "kafka-1:10.1.1.3"
      - "kafka-2:10.1.1.4"
      - "kafka-3:10.1.1.5"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
      - KAFKA_CFG_ZOOKEEPER_CONNECT=<zookeeper-1>:2181,<zookeeper-2>:2182
      - KAFKA_CFG_BROKER_ID=<kafka-id>
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://<kafka-hostname>:9092
```
- Replace `<zookeeper-1>`, `<zookeeper-2>`, `<kafka-hostname>` with their IP address, `<zookeeper-id>` with zeekeeper id, `<kafka-id>` with kafka id