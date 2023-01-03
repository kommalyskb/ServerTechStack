# ElasticSearch Production
ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server 3 ຫນ່ວຍ (ຂັ້ນຕ່ຳ)
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
4. Create docker network
```
docker network create esnet
```
5. Create a new directory for your Elasticsearch installation and navigate to it:
```
mkdir elasticsearch
cd elasticsearch
```
6. Create a new file called `docker-compose.yml` in your Elasticsearch directory, and paste the following code into it:
```
version: '3'
services:
  elasticsearch1:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
    container_name: elasticsearch1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.zen.ping.unicast.hosts=elasticsearch1,elasticsearch2,elasticsearch3
      - discovery.zen.minimum_master_nodes=2
      - node.master=true
      - node.data=true
      - node.ingest=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
    container_name: elasticsearch2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.zen.ping.unicast.hosts=elasticsearch1,elasticsearch2,elasticsearch3
      - discovery.zen.minimum_master_nodes=2
      - node.master=true
      - node.data=true
      - node.ingest=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet
  elasticsearch3:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
    container_name: elasticsearch3
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.zen.ping.unicast.hosts=elasticsearch1,elasticsearch2,elasticsearch3
      - discovery.zen.minimum_master_nodes=2
      - node.master=true
      - node.data=true
      - node.ingest=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata3:/usr/share/elasticsearch/data
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:7.10.1
    container_name: kibana
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://elasticsearch1:9200
      ELASTICSEARCH_HOSTS: http://elasticsearch1:9200,http://elasticsearch2:9200,http://elasticsearch3:9200
    networks:
      - esnet
  logstash:
    image: docker.elastic.co/logstash/logstash:7.10.1
    container_name: logstash
    volumes:
      - ./config:/config-dir
    command: logstash -f /config-dir/logstash.conf
    networks:
      - esnet
    depends_on:
      - elasticsearch1
      - elasticsearch2
      - elasticsearch3
networks:
  esnet:
volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local
  esdata3:
    driver: local

```
7. Create a new file called `logstash.conf` in your Elasticsearch directory, and paste the following code into it:
```
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
  }
}
```

To start the cluster, copy the `docker-compose.yml` file to one of the nodes (e.g. node1) and run the following command:
```
docker-compose up
```
This will start three Elasticsearch nodes, a Kibana instance, and a Logstash instance and connect them to a shared network. The Elasticsearch nodes will form a cluster with high availability, and the Kibana and Logstash instances will be able to connect to the Elasticsearch cluster.

To start the Elasticsearch containers on the other nodes (`node2` and `node3`), copy the `docker-compose.yml` file to each of the nodes and run the following command on each node:
```
docker-compose up
```

