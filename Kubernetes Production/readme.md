# Kubernetes for Production
ເຊີເວີທັງຫມົດທີ່ຕ້ອງການ
- Server Master 2 ຫນ່ວຍ
- Server Worker 3 ຫນ່ວຍ
- Server HAProxy 2 ຫນ່ວຍ
- Virtual IP ສຳລັບ Keepalived 1 ອັນ
ການຕິດຕັ້ງ Kubernetes ໃຊ້ສຳລັບງານ Production ແມ່ນຈະຕ້ອງປະຕິບັດດັ່ງນີ້
## ກຽມ Master node ໄວ້ຢ່າງໜ້ອຍ 2 ເຄື່ອງ ແລ້ວ ແຕ່ລະເຄຶ່ອງແມ່ນໃຫ້ຕິດຕັ້ງ ດັ່ງຕໍ່ໄປນີ້

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

3. Disable swap & Update ipv6 setting
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
```
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF 
```
```
sudo sysctl --system
```

4. Install Keepalived
```
sudo apt-get install -y keepalived
```
Create the Keepalived configuration file `/etc/keepalived/keepalived.conf` with the following content:
```
global_defs {
  notification_email {
    admin@example.com
  }
  notification_email_from keepalived@example.com
  smtp_server smtp.example.com
  smtp_connect_timeout 30
}

vrrp_script chk_haproxy {
  script "killall -0 haproxy" # check the haproxy process
  interval 2 # check every 2 seconds
  weight 2 # add 2 points of prio if OK
}

vrrp_instance VI_1 {
  interface ens3
  state MASTER
  virtual_router_id 51
  priority 101 # 101 on master, 100 on backup
  virtual_ipaddress {
    10.0.0.100/24 # virtual IP
  }
  track_script {
    chk_haproxy
  }
}
```
5. Install HAProxy
```
sudo apt-get install haproxy
```
This can be done by creating a file called haproxy.cfg in the `/etc/haproxy` directory on the load balancer node. You can use the following configuration in the `haproxy.cfg` file:
```
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/run/haproxy.stat

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

frontend kubernetes
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    server kubernetes-master-1 <load-balancer-IP>:6443 check
    server kubernetes-master-2 <load-balancer-IP>:6443 check

```
6. Install common libraries
```
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

7. Add the Kubernetes repository
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

8. Install the Kubernetes binaries
```
sudo apt-get update
sudo apt-get install -y kubeadm kubelet kubectl
```
## ສຳລັບ Master ໜ່ວຍທີ່ 1 ແມ່ນໃຫ້ Init ພ້ອມກັບ ຕັ້ງຄ່າ pod network cidr
Init the first master node with the load balancer IP and use network cidr
- `Calico CNI: 192.168.0.0/16`
- `Flannel CNI: 10.244.0.0/16`
```
kubeadm init --apiserver-advertise-address=<LOAD_BALANCER_IP> --pod-network-cidr=10.244.0.0/16
```
## ສຳລັບ Master ໜ່ວຍທີ່ 2 ແມ່ນໃຫ້ Join ໃສ່ Cluster
```
kubeadm join LOAD_BALANCER_IP:6443 --token TOKEN --discovery-token-ca-cert-hash HASH
```
## ສຳລັບ Worker node ແມ່ນໃຫ້ Join ໃສ່ cluster
```
kubeadm join LOAD_BALANCER_IP:6443 --token TOKEN --discovery-token-ca-cert-hash HASH
```
ສຳລັບ command ໃນກໍລະນີທີ່ Token ຫມົດອາຍຸ ແມ່ນສາມາດ print ເອົາ Token ໃຫມ່ໄດ້
```
kubeadm token create --print-join-command
```
## ຢ້າຍ command kubectl ໄປໄວ້ໃນ Folder ຂອງ User ທັງ 2 Master node
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Pod Network
1. ໃນກໍລະນີໃຊ້ Calico ສ້າງ Calico network
```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```
ເບີ່ງ status ຂອງການຕິດຕັ້ງ
```
watch kubectl get pods -n calico-system
```
ຖ້າ ເບີ່ງແລ້ວ ເຫັນ status ເປັນ pending ຢູ່ຕະຫຼອດໃຫ້ແກ້ໄຂດ້ວຍ
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
- ref: https://rubywind.tistory.com/entry/kubectl-Error-calico-kube-controllers-status-Pending-Allow-pod-scheduling-on-control-plane

2. ໃນກໍລະນີໃຊ້ Flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
## ຕິດຕັ້ງ MetalLB ໃຊ້ສຳລັບເຮັດ Load Balancer
To install MetalLB, you need to create the metallb-system namespace first. You can do this by running the following command:

```
kubectl create namespace metallb-system
```
Then, you can install MetalLB by applying the configuration file using `kubectl apply`.
```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.5/manifests/metallb.yaml

```
To use MetalLB, you will need to create a configuration file that specifies the range of IP addresses that MetalLB should use for LoadBalancer services. Here is an example configuration file that uses the IP range 192.168.100.240-192.168.100.250:
```
ໄອພີ Range ດັ່ງກ່າວຕ້ອງແມ່ນເປັນ IP ທີ່ຢູ່ໃນ Network ດຽວກັບ Node ແລ້ວຕ້ອງບໍ່ຖືກໃຊ້
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.100.240-192.168.100.250

```
To apply the configuration, run the following command:
```
kubectl apply -f metallb-config.yaml
```

After installing MetalLB and applying the ConfigMap, you can create a service for your application. To expose that service to the external network, you will need to create an Ingress resource that routes traffic to the service.

Here's an example of how you can do this:
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
```
Next, create an Ingress resource that routes traffic to the service:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-service
          servicePort: 80

```
## ຕິດຕັ້ງ Helm ໄວ້ໃຊ້ບາງອັນ ເຊັ່ນ Cert Manager
1. Download your desired version (https://github.com/helm/helm/releases)
2. Unpack it (tar -zxvf helm-v3.10.3-linux-amd64.tar.gz). Now latest verion https://get.helm.sh/helm-v3.10.3-linux-amd64.tar.gz
3. Find the helm binary in the unpacked directory, and move it to its desired destination (mv linux-amd64/helm /usr/local/bin/helm)
## ຕິດຕັ້ງ Let's Encrypt ກັບ Cert Manager
To use Let's Encrypt with an ingress in a Kubernetes cluster, you can use the Cert-Manager project. Cert-Manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various certificate authorities (CAs).

Here are the steps to install and configure Cert-Manager in your Kubernetes cluster:

1. Add the Cert-Manager Helm chart repository to your local Helm client:
```
helm repo add jetstack https://charts.jetstack.io
```

2. Install the Cert-Manager Helm chart:
```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.3.0
```

3. Create a ClusterIssuer resource for Let's Encrypt:
```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: your@email.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```
4. Update the ingress resource for your application to use the Let's Encrypt ClusterIssuer:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    # Use the Let's Encrypt ClusterIssuer
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-com-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-service
          servicePort: 