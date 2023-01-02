# PostgreSQL Production
ການຕິດຕັ້ງ PostgreSQL ສຳລັບງານ Production
- Server 2 ຫນ່ວຍ
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
## Create Docker network
1. On Host 1
```
docker network create --driver bridge postgresql_network
```
2. On Host 2
```
docker network create --driver bridge --attachable postgresql_network
```
## Create docker-compose file
```
version: '3'

services:
  postgres:
    image: postgres:latest
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      PG_EXPORTER_WEB_LISTEN_ADDRESS: ":9187"
      PG_EXPORTER_EXTEND_QUERY_DISABLED: true
      PG_EXPORTER_CONNECTION_TIMEOUT: 5s
    ports:
      - "5432:5432"
      - "9187:9187"
    volumes:
      - /path/on/host:/var/lib/postgresql/data
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == manager
    networks:
      - postgresql_network

  prometheus:
    image: prom/prometheus
    volumes:
      - /path/on/host/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /path/on/host/data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - postgresql_network
  grafana:
    image: grafana/grafana
    volumes:
      - /path/on/host/grafana:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - postgresql_network

networks:
  postgresql_network:
    name: postgresql_cluster_default
    driver: overlay

```
This docker-compose.yml file defines three services:

- `postgres`: A PostgreSQL node in the cluster. It exposes port 5432 for the PostgreSQL service and port 9187 for the Prometheus exporter.

- `prometheus`: A Prometheus server that scrapes metrics from the PostgreSQL nodes. It exposes port 9090 for the Prometheus dashboard.

- `grafana`: A Grafana server that displays the metrics collected by Prometheus in customizable dashboards. It exposes port 3000 for the Grafana dashboard.

The `deploy` section of the `postgres` service specifies that three replicas of the service should be run, and the `placement` constraints ensure that the replicas are distributed across the three nodes in the cluster.

The `postgresql_network` network connects all the services and allows them to communicate with each other using the service names as hostnames.

To start the PostgreSQL cluster and the monitoring services, run the `docker-compose up -d` command on each of the three nodes in the cluster. You can then access the Prometheus and Grafana dashboards at the following URLs:

Prometheus: `http://<node_ip>:9090`
Grafana: `http://<node_ip>:3000`