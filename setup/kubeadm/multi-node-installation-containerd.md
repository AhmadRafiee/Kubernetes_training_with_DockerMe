### Install tools and dependency
### On all Servers
```bash
echo "Change hostname"
hostnamectl set-hostname

echo "update and upgrade"
apt update ; apt upgrade -y

echo "install tools"
apt install -y wget git vim bash-completion curl htop net-tools dnsutils \
               atop sudo software-properties-common telnet axel jq iotop

echo "nodes ip address"
master1_ip=192.168.1.141
master2_ip=192.168.1.146
master3_ip=192.168.1.145
worker1_ip=192.168.1.150
worker2_ip=192.168.1.148
worker3_ip=192.168.1.147
lb_api1_ip=192.168.1.154
lb_api2_ip=192.168.1.155
lb_ingress1_ip=192.168.1.157
lb_ingress2_ip=192.168.1.163
vip_api=192.168.1.44
vip_ingress=192.168.1.45
storage_ip=192.168.1.158

echo "nodes name"
master1_name=master1
master2_name=master2
master3_name=master3
worker1_name=worker1
worker2_name=worker2
worker3_name=worker3
vip_api_name=api

echo "domain name"
domain_name=monlog.ir
```

### Install and configuration containerd
### On all Servers
```bash
# Install repo dependency
sudo apt-get install ca-certificates curl gnupg lsb-release

# Add Docker’s official GPG key:
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Use the following command to set up the repository:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# install containerd
sudo apt-get install containerd.io

# Start containerd automatically at boot
systemctl enable containerd
systemctl restart containerd
systemctl status containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup

# Restart containerd
sudo systemctl restart containerd
```


### Pre-configuration kubernetes
### On master and worker nodes
```bash
echo "Add sysctl settings"
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system >/dev/null 2>&1

echo "Disable and turn off SWAP"
sed -i '/swap/d' /etc/fstab
swapoff -a

echo "Installing apt-transport-https pkg"
apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Download the Google Cloud public signing key:
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# Add the Kubernetes apt repository:
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
apt-get update -y

# Install Kubernetes kubeadm, kubelet and kubectl
apt-get install -y kubelet=1.25.5-00
apt-get install -y kubectl=1.25.5-00
apt-get install -y kubeadm=1.25.5-00

echo "check versions tools"
kubelet --version
kubeadm version
kubectl version

# hold kubernetes packages
sudo apt-mark hold kubelet kubeadm kubectl

echo "Enable and start kubelet service"
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

### haproxy and keepalived install and configuration
### On API loadbalancer nodes
```bash
echo "install haproxy and keepalived service"
apt install -y haproxy keepalived

echo "copy and move haproxy config"
cat /etc/haproxy/haproxy.cfg
cat <<EOT >> /etc/haproxy/haproxy.cfg
listen Stats-Page
  bind *:8000
  mode http
  stats enable
  stats hide-version
  stats refresh 10s
  stats uri /
  stats show-legends
  stats show-node

frontend fe-apiserver
   bind 0.0.0.0:6443
   mode tcp
   option tcplog
   default_backend be-apiserver

backend be-apiserver
   mode tcp
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
   server control-plane-1 ${master1_ip}:6443 check
   server control-plane-2 ${master2_ip}:6443 check
   server control-plane-3 ${master3_ip}:6443 check
EOT
cat /etc/haproxy/haproxy.cfg

echo "check haproxy config file"
haproxy -c -f /etc/haproxy/haproxy.cfg

echo "Enable and start haproxy service"
{
systemctl enable haproxy
systemctl restart haproxy
systemctl is-active --quiet haproxy && echo -e "\e[1m \e[96m haproxy service: \e[30;48;5;82m \e[5mRunning \e[0m" || echo -e "\e[1m \e[96m docker service: \e[30;48;5;196m \e[5mNot Running \e[0m"
}

echo "check haproxy status page"
netstat -ntlp | grep 8000
```
### On Master API loadbalancer node
```bash
cat <<EOT > /etc/keepalived/keepalived.conf
global_defs {
   enable_script_security
   script_user root
}

vrrp_script check_haproxy {
   script "killall -0 haproxy"
   interval 2
   weight 2
   }

vrrp_instance KUBE_API_LB {
   state MASTER
   interface ens160
   virtual_router_id 51
   priority 101
   # The virtual ip address shared between the two loadbalancers
   virtual_ipaddress {
      ${vip_api}/32
   }
   track_script {
      check_haproxy
   }
}
EOT
cat /etc/keepalived/keepalived.conf

echo "check keepalived config file"
keepalived -t -l -f /etc/keepalived/keepalived.conf

echo "Enable and start keepalived service"
{
systemctl enable keepalived
systemctl restart keepalived
systemctl is-active --quiet keepalived && echo -e "\e[1m \e[96m keepalived service: \e[30;48;5;82m \e[5mRunning \e[0m" || echo -e "\e[1m \e[96m docker service: \e[30;48;5;196m \e[5mNot Running \e[0m"
}

echo "check vip"
ip a | grep 192.168.1.44/32
```
### On Slave API loadbalancer node
```bash
cat <<EOT > /etc/keepalived/keepalived.conf
global_defs {
   enable_script_security
   script_user root
}
# Script used to check if HAProxy is running
vrrp_script check_haproxy {
   script "killall -0 haproxy"
   interval 2
   weight 2
}

vrrp_instance KUBE_API_LB {
   state BACKUP
   interface ens160
   virtual_router_id 51
   priority 100
   virtual_ipaddress {
      ${vip_api}/32
   }
   track_script {
      check_haproxy
   }
}
EOT

cat /etc/keepalived/keepalived.conf
echo "check keepalived config file"
keepalived -t -l -f /etc/keepalived/keepalived.conf

echo "Enable and start keepalived service"
{
systemctl enable keepalived
systemctl restart keepalived
systemctl is-active --quiet keepalived && echo -e "\e[1m \e[96m keepalived service: \e[30;48;5;82m \e[5mRunning \e[0m" || echo -e "\e[1m \e[96m docker service: \e[30;48;5;196m \e[5mNot Running \e[0m"
}

echo "check vip"
ip a | grep 192.168.1.44/32

```
### kubeadm config
### On master1
```bash
echo "create kubeadm config file"
cat <<EOT >/opt/kubeadm_config.yml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${master1_ip}
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
  extraArgs:
    authorization-mode: "Node,RBAC"
  certSANs:
    - "${vip_api}"
    - "${master1_ip}"
    - "${master2_ip}"
    - "${master3_ip}"
    - "${vip_api_name}"
    - "vip"
    - "${master1_name}"
    - "${master2_name}"
    - "${master3_name}"
    - "vip.${domain_name}"
    - "${vip_api_name}.${domain_name}"
    - "${master1_name}.${domain_name}"
    - "${master2_name}.${domain_name}"
    - "${master3_name}.${domain_name}"
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
dns:
  type: CoreDNS
etcd:
  local:
    imageRepository: "quay.io/coreos"
    imageTag: "v3.5.4"
    dataDir: "/var/lib/etcd"
    serverCertSANs:
      - "${vip_api}"
      - "${master1_ip}"
      - "${master2_ip}"
      - "${master3_ip}"
      - "${vip_api_name}"
      - "vip"
      - "${master1_name}"
      - "${master2_name}"
      - "${master3_name}"
      - "${vip_api_name}.${domain_name}"
      - "${master1_name}.${domain_name}"
      - "${master2_name}.${domain_name}"
      - "${master3_name}.${domain_name}"
    peerCertSANs:
      - "${vip_api}"
      - "${master1_ip}"
      - "${master2_ip}"
      - "${master3_ip}"
      - "${vip_api_name}"
      - "vip"
      - "${master1_name}"
      - "${master2_name}"
      - "${master3_name}"
      - "${vip_api_name}.${domain_name}"
      - "${master1_name}.${domain_name}"
      - "${master2_name}.${domain_name}"
      - "${master3_name}.${domain_name}"
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.25.5
controlPlaneEndpoint: "vip.${domain_name}:6443"
networking:
  dnsDomain: cluster.local
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "172.124.0.0/17"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOT
cat /opt/kubeadm_config.yml

echo "Get images list"
kubeadm config images list --config /opt/kubeadm_config.yml
kubeadm config images pull --config /opt/kubeadm_config.yml
```
### before `kubeadm init` run these steps:
  - pull all docker images
  - dns recorde add and check
```bash
echo "initalize master1"
kubeadm init --config /opt/kubeadm_config.yml

echo "kubectl config"
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "check certificate"
kubeadm certs check-expiration
openssl x509 -text -noout -in /etc/kubernetes/pki/apiserver.crt 

echo "Deploy Calico network"
kubectl create -f https://docs.projectcalico.org/latest/manifests/calico.yaml

echo "Install etcdctl On Ubuntu 16.04/18.04 "
etcd_version=v3.5.4
curl -L https://github.com/coreos/etcd/releases/download/$etcd_version/etcd-$etcd_version-linux-amd64.tar.gz -o etcd-$etcd_version-linux-amd64.tar.gz
tar xzvf etcd-$etcd_version-linux-amd64.tar.gz
rm etcd-$etcd_version-linux-amd64.tar.gz
cp etcd-$etcd_version-linux-amd64/etcdctl /usr/local/bin/
rm -rf etcd-$etcd_version-linux-amd64
etcdctl version

echo "check etcd server on master nodes"
export endpoint="https://${master1_ip}:2379,${master2_ip}:2379,${master3_ip}:2379"
export flags="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
              --cert=/etc/kubernetes/pki/etcd/server.crt \
              --key=/etc/kubernetes/pki/etcd/server.key"
endpoints=$(sudo ETCDCTL_API=3 etcdctl member list $flags --endpoints=${endpoint} \
            --write-out=json | jq -r '.members | map(.clientURLs) | add | join(",")')

echo "verify with these commands"
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} member list
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint status
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint health
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} alarm list

etcdctl member list $flags --endpoints=${endpoint} --write-out=table
etcdctl endpoint status $flags --endpoints=${endpoint} --write-out=table
```
### master and worker join cluster
### after initializing master 1 run these steps
- join master2:
  - check pod
  - check nodes
  - check etcd cluster
‍
- join master3:
  - check pod
  - check nodes
  - check etcd cluster

- join worker1,2,3:
  - check pod
  - check nodes