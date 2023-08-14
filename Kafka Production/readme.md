# Kafka for Production
ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server 3 ຫນ່ວຍ
## Install Docker and Docker Compose
1. Update the system
```
sudo apt-get update
```

2. Install Docker
``` 
sudo apt install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```
3. Install Docker compose
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Installation Kafka, Zookeeper and Prothemeus
ສາມາດຕິດຕັ້ງແບບໃຊ້ທັງ zookeeper ແລະ kafka ພ້ອມກັນໄດ້
1. ສ້າງ Docker Network
On Host 1
```
docker network create --driver bridge streaming_network
```
On Host 2 & 3
```
docker network create --driver bridge --attachable streaming_network
```
2. ສ້າງ `docker-compose.yaml`
```
version: '3'

services:
  zookeeper:
    image: bitnami/zookeeper:latest
    restart: always
    environment:
      ALLOW_ANONYMOUS_LOGIN: 'yes'
    ports:
      - "2181:2181"
      - "8080:8080"
    volumes:
      - /path/on/host:/bitnami/zookeeper
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == manager
    networks:
      - streaming_network

  kafka:
    image: bitnami/kafka:latest
    restart: always
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      ALLOW_PLAINTEXT_LISTENER: 'yes'
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,SSL://:9093,SASL_PLAINTEXT://:9094,SASL_SSL://:9095
      KAFKA_CFG_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    ports:
      - "9092:9092"
      - "9093:9093"
      - "9094:9094"
      - "9095:9095"
      - "8080:8080"
    volumes:
      - /path/on/host:/bitnami/kafka
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == manager
    networks:
      - streaming_network

  prometheus:
    image: prom/prometheus
    volumes:
      - /path/on/host/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /path/on/host/data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - streaming_network
  grafana:
    image: grafana/grafana
    volumes:
      - /path/on/host/grafana:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - streaming_network

networks:
  streaming_network:
    name: streaming_cluster_default
    driver: overlay
```
This `docker-compose.yml` file defines four services:

- `zookeeper`: A Zookeeper node in the cluster. It exposes ports 2181 and 8080 for the Zookeeper service and the Prometheus exporter, respectively.

- `kafka`: A Kafka broker in the cluster. It exposes ports 9092-9095 for the Kafka service and port 8080 for the Prometheus exporter.

- `prometheus`: A Prometheus server that scrapes metrics from the Kafka and Zookeeper nodes. It exposes port 9090 for the Prometheus dashboard.

grafana: A Grafana server that displays the metrics collected by Prometheus in customizable dashboards. It exposes port 3000 for the Grafana dashboard.

The `deploy` section of the `zookeeper` and `kafka` services specifies that three replicas of each service should be run, and the `placement` constraints ensure that the replicas are distributed across the three nodes in the cluster.

The my_network network connects all the services and allows them to communicate with each other using the service names as hostnames.

To start the Kafka cluster and the monitoring services, run the docker-compose up -d command on each of the three nodes in the cluster. You can then access the Prometheus and Grafana dashboards at the following URLs:

Prometheus: `http://<node_ip>:9090`
Grafana: `http://<node_ip>:3000`
3. Create a `prometheus.yml` file in the host directory specified in the `volumes` section of the `prometheus` service. In the `prometheus.yml` file, add the following configuration to specify the Kafka and Zookeeper nodes that Prometheus should scrape metrics from:
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:8080']

  - job_name: 'zookeeper'
    static_configs:
      - targets: ['zookeeper:8080']

```

Other sample
`https://www.baeldung.com/ops/kafka-docker-setup`

## Monitor docker container

```
docker stats [container_name] --no-stream --format "{{ json . }}" | python3 -m json.tool
```

## Create Kafka Topic
This is topic for telbiz system
```
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic walletqueuerequest 
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic walletaccount
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic walletconfirmrequest
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic guessservice
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic topupwithbcel
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic smswithbcel
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic datawithbcel
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic smsconfirmrequest
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic topupconfirmrequest
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic dataconfirmrequest
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic vasconfirmrequest
sudo docker exec broker kafka-topics --bootstrap-server broker:9092 --create --topic vaswithbcel
```
codecamplao/telbiz-consumer-sms
codecamplao/telbiz-consumer-payment
codecamplao/telbiz-consumer-wallet
codecamplao/telbiz-consumer-topup
codecamplao/telbiz-consumer-data
codecamplao/telbiz-consumer-vas