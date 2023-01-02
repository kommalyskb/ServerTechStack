# Minio for Production
Minio is an open source object storage server that is compatible with the Amazon S3 API. It can be used as a standalone application or in distributed mode with multiple server nodes.

ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server: Minio 3 ຫນ່ວຍ
- Server: HAProxy 2 ຫນ່ວຍ
- Virtual IP ສຳລັບ Keepalived 1 ອັນ

## Install Docker and Docker compose on all the nodes that will be part of the Minio cluster.
```
sudo apt install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Create a shared storage location 
that will be used by all the Minio nodes. This could be an NFS mount, a shared block storage device, or any other storage solution that is supported by Docker.

For example, in the instructions I provided, the shared directory is /mnt/minio-storage. This means that you need to create a directory with this name on each node, and make sure that it is accessible by all nodes. You can do this by using a shared filesystem such as NFS, or by setting up a distributed file system such as Ceph.
```
sudo mkdir -p /mnt/minio-storage
```
## Docker Compose file
On each node, create a Docker Compose file that defines the Minio service. The Compose file should specify the shared storage location as the `MINIO_VOLUMES` environment variable, and `MINIO_PEERS` should be IP address of other nodes
```
version: '3'
services:
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
    environment:
      - MINIO_ACCESS_KEY=minio
      - MINIO_SECRET_KEY=minio123
      - MINIO_DISTRIBUTION_POLICY=all
      - MINIO_QUORUM_CONSENSUS=on
      - MINIO_PEERS="http://node-2:9000,http://node-3:9000"
    volumes:
      - /mnt/minio-storage:/export
```
- ແຕ່ລະເຄື່ອງຕ້ອງໄດ້ແກ້ MINIO_PEERS ຕາມຄວາມເໝາະສົມ ເຊັ່ນ ຖ້າຢູ່ node-1 ກໍ່ໃຫ້ໃສ່ແຕ່ node-2 ກັບ node-3, ແຕ່ຖ້າຢູ່ node-2 ແມ່ນໃຫ້ໃສ່ node-1 ກັບ node-3

Start the Minio cluster by running `docker-compose up` on each node.

## Set HAProxy for Load Balancer
1. Install HAProxy and Keepalived on each load balancer node:
```
sudo apt-get update
sudo apt-get install haproxy keepalived
```
2. Configure HAProxy by editing the `/etc/haproxy/haproxy.cfg` file. Add the following configuration to the file:
```
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    server node1 <node1_ip>:9000 check
    server node2 <node2_ip>:9000 check
    server node3 <node3_ip>:9000 check
```
Replace `<node1_ip>`, `<node2_ip>`, and `<node3_ip>` with the IP addresses of your Minio nodes.
3. Configure Keepalived by editing the `/etc/keepalived/keepalived.conf` file. Add the following configuration to the file:
```
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface <interface_name>
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass password
    }
    virtual_ipaddress {
        <virtual_ip>
    }
    track_script {
        chk_haproxy
    }
}
```
Replace `<interface_name>` with the name of the network interface that you want to use for the virtual IP, and replace `<virtual_ip>` with the virtual IP that you want to use for the load balancer.
4. Start the HAProxy and Keepalived services:
```
sudo systemctl start haproxy
sudo systemctl start keepalived
```
5. Enable the HAProxy and Keepalived services to start on boot:
```
sudo systemctl enable haproxy
sudo systemctl enable keepalived
```