# Apache CouchDB for Production
Apache CouchDB ™ lets you access your data where you need it. The Couch Replication Protocol is implemented in a variety of projects and products that span every imaginable computing environment from globally distributed server-clusters, over mobile phones to web browsers

ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server: Apache CouchDB 3 ຫນ່ວຍ
- Server: HAProxy 2 ຫນ່ວຍ
- Virtual IP ສຳລັບ Keepalived 1 ອັນ

## Install Docker and Docker compose on all the nodes that will be part of the CouchDB cluster.
```
sudo apt install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
## How to create a Docker network
At the moment, the CouchDB nodes have no awareness of one another. To fix that we need to create a new Docker network. Do this with:
```
docker network create -d bridge --subnet 172.25.0.0/16 isolated_nw
```
## How to deploy the CouchDB containers
We’re going to deploy three CouchDB containers, each using a unique external port. The first will use port 5984 and is deployed with:
```
version: '3'       
services:
  couchserver:     
    image: couchdb 
    restart: always
    ports:
      - "5984:5984"
      - "5986:5986"
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=1qaz2wsx
      - TZ=Asia/Vientiane
      - NODENAME=couchdb-{id}.telbiz.la
    volumes:
        - ./dbdata:/opt/couchdb/data
    networks:
      - isolated_nw
networks:
  isolated_nw:
```

`{id}` is index of server
Start docker-compose all of 3 nodes
```
docker-compose up -d
```
