
# Provision Volumes on Kubernetes using Ceph CSI 
## setup ceph cluster function test
### On ceph node
```bash
docker run -d \
--name demo \
-e MON_IP=172.60.70.1 \
-e CEPH_PUBLIC_NETWORK=172.60.70.0/24 \
--net=host \
-v /var/lib/ceph:/var/lib/ceph \
-v /etc/ceph:/etc/ceph \
-e CEPH_DEMO_UID=qqq \
-e CEPH_DEMO_ACCESS_KEY=qqq \
-e CEPH_DEMO_SECRET_KEY=qqq \
-e CEPH_DEMO_BUCKET=qqq  \
ceph/daemon:latest \
demo
```

### check ceph containers
```bash
docker ps
docker logs -f demo 
netstat -ntlp
apt install ceph-common -y
sudo ceph -s 
```

### enable ceph dashboard
```bash
sudo ceph mgr module enable dashboard
```

## On my station
## First, let’s install helm to deploy Ceph CSI chart on Kubernetes.
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Create a namespace for Ceph CSI.
```bash
kubectl create namespace ceph-csi-rbd; 
```

### Cloning Ceph CSI and switching to v3.3.1.
```bash
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi
git checkout v3.3.1
```

### move to rbd chart.
```bash
cd charts/ceph-csi-rbd;

# Optionally, you can change the csi version to v1beta1 for Kubernetes v1.17.x
sed -i 's/storage.k8s.io\/betav1/storage.k8s.io\/v1beta1/g' templates/csidriver-crd.yaml;
```
### get ceph cluster data
```bash
# clusterID can be obtained by the command.
sudo ceph fsid;

# And monitors for ceph monitors can be obtained with this.
sudo ceph mon dump;
```

### Now, we need chart values file to deploy ceph csi. It looks like this.
```bash
cat <<EOF > ceph-csi-rbd-values.yaml
csiConfig:
  - clusterID: "757d3b56-fa4c-475f-a04d-a966ce9519c4"
    monitors:
      - "172.60.70.1:3300"
provisioner:
  name: provisioner
  replicaCount: 2
EOF
```

### Let’s install Ceph CSI chart on Kuberntes.
```bash
helm install --namespace ceph-csi-rbd ceph-csi-rbd --values ceph-csi-rbd-values.yaml ./;
kubectl rollout status deployment ceph-csi-rbd-provisioner -n ceph-csi-rbd;
helm status ceph-csi-rbd -n ceph-csi-rbd;
```

### Create a pool in the ceph.
```bash
sudo ceph osd pool create kubePool 64 64
```

### And initialize the pool as block device.
```bash
sudo rbd pool init kubePool
```

### Let’s create an admin user for the pool kubePool , and encode the generated key as base64.
```bash
sudo ceph auth get-or-create-key client.kubeAdmin mds 'allow *' mgr 'allow *' mon 'allow *' osd 'allow * pool=kubePool' | tr -d '\n' | base64;
```

### Encode the admin user kubeAdmin as base64.
```bash
echo "kubeAdmin" | tr -d '\n' | base64;
```

### Let’s create a secret resource for the admin user.
```bash
cat > ceph-admin-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin
  namespace: default
type: kubernetes.io/rbd
data:
  userID: a3ViZUFkbWlu
  userKey: QVFDNTM4cGdPUnBHTGhBQTJVYU1xWGhvN2U4WEZIeERtek5xMXc9PQ==
EOF
```

### create a secret for admin user.
```bash
kubectl apply -f ceph-admin-secret.yaml;
```

### Finally, we are ready to create Ceph Storage Class. Let’s create it.
```bash
# ceph storage class.
cat > ceph-rbd-sc.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 757d3b56-fa4c-475f-a04d-a966ce9519c4
   pool: kubePool
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: ceph-admin
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: ceph-admin
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/controller-publish-secret-name: ceph-admin
   csi.storage.k8s.io/controller-publish-secret-namespace: default
   csi.storage.k8s.io/node-publish-secret-name: ceph-admin
   csi.storage.k8s.io/node-publish-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: ceph-admin
   csi.storage.k8s.io/node-stage-secret-namespace: default
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF

kubectl apply -f ceph-rbd-sc.yaml;
```

### install ceph-common on all nodes 
```bash
sudo apt update
sudo apt install -y ceph-common
```

### Let’s create a pod with pvc to mount the volume from the external ceph storage using ceph storage class.
```bash
## create pvc and pod.
cat <<EOF > pv-pod.yaml   
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-rbd-sc-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: ceph-rbd-sc
---    
apiVersion: v1
kind: Pod
metadata:
  name: ceph-rbd-pod-pvc-sc
spec:
  containers:
  - name:  ceph-rbd-pod-pvc-sc
    image: busybox
    command: ["sleep", "infinity"]
    volumeMounts:
    - mountPath: /mnt/ceph_rbd
      name: volume
  volumes:
  - name: volume
    persistentVolumeClaim:
      claimName: ceph-rbd-sc-pvc
EOF
 
kubectl apply -f pv-pod.yaml;
```

### Let’s check the volume mount in the container.
```bash
kubectl exec pod/ceph-rbd-pod-pvc-sc -- df -k | grep rbd;
```

### And, check if an image is created in the ceph pool.
```bash
sudo rbd ls -p kubePool;
```

Reference:
- https://itnext.io/provision-volumes-from-external-ceph-storage-on-kubernetes-and-nomad-using-ceph-csi-7ad9b15e9809
- https://artifacthub.io/packages/helm/ceph-csi/ceph-csi-rbd