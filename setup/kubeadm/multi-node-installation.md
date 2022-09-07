# Install tools and dependency

## Install and configure prerequisites (On all Servers)
```bash
echo "Disable and turn off SWAP"
sed -i '/swap/d' /etc/fstab
swapoff -a
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

## Install Containerd (On all Servers)
### Using Docker repository
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y containerd.io runc
cat <<EOF | sudo tee -a /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
EOF
sudo sed -i 's/^disabled_plugins \=/\#disabled_plugins \=/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl status containerd
```
## Using Containerd binaries (offical)
### Install Containerd core
First, download the latest version of `containerd` from [GitHub](https://github.com/containerd/containerd/releases) and extract the files to the `/usr/local/` directory.

```bash
# Set Containerd version
export "containerd_version=1.6.8"

# Download Containerd
wget "https://github.com/containerd/containerd/releases/download/v${containerd_version}/containerd-${containerd_version}-linux-amd64.tar.gz"
# Extract Containerd
sudo tar Czxvf /usr/local "containerd-${containerd_version}-linux-amd64.tar.gz"
```

Download Containerd service
```bash
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

Configuration
```bash
sudo mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

Install Containerd service
```bash
sudo mv containerd.service /usr/lib/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```
View Containerd service status
```bash
sudo systemctl status containerd
```
### Install runC

runC is an open-source container runtime for spawning and running containers on Linux according to the OCI specification.

Download the latest version of `runC` from [GitHub](https://github.com/opencontainers/runc/releases) and install it as `/usr/local/sbin/runc`.
```bash
export "runc_version=1.1.4"

wget "https://github.com/opencontainers/runc/releases/download/v${runc_version}/runc.amd64"

sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Install CNI Plugins For Containerd

For the container to run, you need to install CNI plugins. So, download the latest version of CNI plugins from [GitHub](https://github.com/containernetworking/plugins/releases) and place them in the `/opt/cni/bin` directory.
```bash
export "cni_version=1.1.1"

sudo mkdir -p /opt/cni/bin/

sudo wget "https://github.com/containernetworking/plugins/releases/download/v${cni_version}/cni-plugins-linux-amd64-v${cni_version}.tgz"

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v${cni_version}.tgz
```
Restart the containerd service.
```bash
sudo systemctl restart containerd
```

### Install Nerdctl (Optional)
nerdctl is a Docker-compliant command-line interface for containerd. It is not part of the core package. So, this has to be installed separately.

Download the latest version of `nerdctl` from [GitHub](https://github.com/containerd/nerdctl/releases) and extract it to the `/usr/local/bin` directory.

```bash
export "nerdctl_version=0.22.2"
wget "https://github.com/containerd/nerdctl/releases/download/v${nerdctl_version}/nerdctl-${nerdctl_version}-linux-amd64.tar.gz"

sudo tar Cxzvf /usr/local/bin nerdctl-${nerdctl_version}-linux-amd64.tar.gz
```
## Set ips and domains on all servers
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



## Pre-configuration kubernetes (On master and worker nodes)
```bash
echo "Add sysctl settings"
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system >/dev/null 2>&1

echo "Installing apt-transport-https pkg"
apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
ls -ltr /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
apt-get update -y

echo "Install Kubernetes kubeadm, kubelet and kubectl"
apt-get install -y kubelet=1.21.0-00
apt-get install -y kubectl=1.21.0-00
apt-get install -y kubeadm=1.21.0-00

echo "check versions tools"
kubelet --version
kubeadm version
kubectl version

echo "Enable and start kubelet service"
systemctl enable kubelet 
systemctl start kubelet  
systemctl status kubelet 
```

## haproxy and keepalived install and configuration
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
## kubeadm config
### On master1
```bash
echo "create kubeadm config file"
cat <<EOT >/opt/kubeadm_config.yml
apiVersion: kubeadm.k8s.io/v1beta2
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
  criSocket: /var/run/dockershim.sock
  name: master1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
  timeoutForControlPlane: 4m0s
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
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    imageRepository: "quay.io/coreos"
    imageTag: "v3.4.16"
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
#kubernetesVersion: "v1.21.1"
imageRepository: "k8s.gcr.io"
useHyperKubeImage: false
kind: ClusterConfiguration
controlPlaneEndpoint: "vip.${domain_name}:6443"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "172.124.0.0/17"
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
kubectl create -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml

echo "Install etcdctl On Ubuntu 16.04/18.04 "
etcd_version=v3.4.16
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
## master and worker join cluster
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
