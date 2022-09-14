Install Kubernetes

# Install and configure prerequisites
```bash
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

# Install Containerd
## Using Docker repository
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

# Install CNI Plugins For Containerd

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

# Install Nerdctl (Optional)
nerdctl is a Docker-compliant command-line interface for containerd. It is not part of the core package. So, this has to be installed separately.

Download the latest version of `nerdctl` from [GitHub](https://github.com/containerd/nerdctl/releases) and extract it to the `/usr/local/bin` directory.

```bash
export "nerdctl_version=0.22.2"
wget "https://github.com/containerd/nerdctl/releases/download/v${nerdctl_version}/nerdctl-${nerdctl_version}-linux-amd64.tar.gz"

sudo tar Cxzvf /usr/local/bin nerdctl-${nerdctl_version}-linux-amd64.tar.gz
```

# Install Kubernetes

Now that containerd is installed on both our nodes, we can start our Kubernetes installation.

## Add Kubernetes key on both nodes
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```

## Add Kube repository on both nodes
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

## Update both your systems and install all the Kubernetes modules
```bash
sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl
```

## Set hostnames

On the **master** node, run:
```bash
sudo hostnamectl set-hostname "master-node"
exec bash
```

On the **worker** node, run:
```bash
sudo hostnamectl set-hostname "w-node1"
exec bash
```

Set the hostnames in the `/etc/hosts` file of the worker:
```bash
sudo cat <> /etc/hosts
160.119.248.60 master-node
160.119.248.162 node1 W-node1
EOF
```

## Config firewall rules

Set up the following firewall rules on the **master** node
```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 10255/tcp
sudo ufw reload
```

Set up the following firewall rules on the **worker** node
```bash
sudo ufw allow 10251/tcp
sudo ufw allow 10255/tcp
sudo ufw reload
```

## Turn swap off

It’s required for kubelet to work, run on both nodes
```bash
sudo swapoff –a
```

## Enable kubelet service on both nodes:
```bash
sudo systemctl enable kubelet
```

# Deploy the Kubernetes cluster
## Initialise cluster

On the **master** node, execute the following command to initialise the Kubernetes cluster:
```bash
sudo kubeadm init
```

The process can take a few minutes. The last few lines of your output should look similar to this:
```
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Alternatively, if you are the root user, you can run:
export KUBECONFIG=/etc/kubernetes/admin.conf
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.2.105:6443 --token abcdef.abcdefghijklmnop \
--discovery-token-ca-cert-hash sha256:8dfad80a388f4c93a9d5fb6d0b5b3ceda08305bac044ec8417e9f4f3c473893d
```

Copy the `kubeadm join` from the end of the above output. We will be using this command to add worker nodes to our cluster.

If you forgot to copy or misplaced the command, don’t worry; you can get it back by executing this command:
```bash
sudo kubeadm token create --print-join-command
```

## Create and claim directory

As indicated by the output above, we need to create a directory and claim its ownership to start managing our cluster.

Run the following commands:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Deploy pod network to cluster

We will use Flannel to deploy a pod network to our cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

You should see the following output after running the above command:
```
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```
You should be able to verify that your master node is ready now:
```bash
sudo kubectl get nodes
```
Output:
```
NAME          STATUS   ROLES                  AGE   VERSION
master-node   Ready    control-plane,master   90s   v1.23.3
```

…and that all the pods are up and running:
```bash
sudo kubectl get pods --all-namespaces
```
```
Output
NAMESPACE     NAME                                  READY   STATUS              RESTARTS        AGE
kube-system   coredns-957326482-zdgsd               0/1     Running		0               22m
kube-system   coredns-957326482-srfgh               0/1     Running		0               22m
kube-system   etcd-master-node                      1/1     Running             0               22m
kube-system   kube-apiserver-master-node            1/1     Running             0               22m
kube-system   kube-controller-manager-master-node   1/1     Running             0               22m
kube-system   kube-flannel-ds-dnjsd                 0/1     Running		0		22m
kube-system   kube-flannel-ds-dfjyf                 0/1     Running    		0     		22m
kube-system   kube-proxy-jfbur                      1/1     Running             0               22m
kube-system   kube-proxy-sdfeh                      1/1     Running             0               20m
kube-system   kube-scheduler-master-node            1/1     Running             0               22m
```

## Add nodes

At this point, we are ready to add nodes to our cluster.

Copy **your own** `kubeadm join` command from **Step: Initialise cluster** and run it on the worker node:
```bah
kubeadm join 192.168.2.105:6443 --token abcdef.abcdefghijklmnop \
--discovery-token-ca-cert-hash sha256:8dfad80a388f4c93a9d5fb6d0b5b3ceda08305bac044ec8417e9f4f3c473893d
```

Output:
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

Run `kubectl get nodes` on the control-plane to see this node join the cluster.

To verify that the worker node indeed got added to the cluster, execute the following command on the master node:
```bash
kubectl get nodes
```
Output:
```
NAME          STATUS   ROLES                  AGE     VERSION
master-node   Ready    control-plane,master   2m54s   v1.25.0
w-node1       Ready                           27s     v1.25.0
```

You can set the role for your worker node using:

sudo kubectl label node w-node1 node-role.kubernetes.io/worker=worker

Get nodes again to verify:
```bash
kubectl get nodes
```
Output:
```
NAME          STATUS   ROLES                  AGE     VERSION
master-node   Ready    control-plane,master   4m34s   v1.25.0
w-node1       Ready    worker                 1m24s   v1.25.0
```
To add more nodes, repeat this Add nodes step on more machines.

That’s it! Your two-node Kubernetes cluster is up and running!
