# Velero 
## Backup and migrate Kubernetes resources and persistent volumes

## Prerequisites
  - Access to a Kubernetes cluster, version 1.7 or later. Note: restic support requires Kubernetes version 1.10 or later, or an earlier version with the mount propagation feature enabled. Restic support is not required for this example, but may be of interest later. See Restic Integration.
  - A DNS server on the cluster
  - kubectl installed
  - Sufficient disk space to store backups in Minio. You will need sufficient disk space available to handle any backups plus at least 1GB additional. Minio will not operate if less than 1GB of free disk space is available.


## Download Velero
1. Download the latest official release’s tarball for your client platform.

    We strongly recommend that you use an official release of Velero. The tarballs for each release contain the velero command-line client. The code in the main branch of the Velero repository is under active development and is not guaranteed to be stable!

2. Extract the tarball:
```
tar -xvf <RELEASE-TARBALL-NAME>.tar.gz -C /dir/to/extract/to
```
    The directory you extracted is called the “Velero directory” in subsequent steps.

3. Move the velero binary from the Velero directory to somewhere in your PATH.

**Sample**

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.7.0/velero-v1.7.0-linux-amd64.tar.gz
tar xzf velero-v1.7.0-linux-amd64.tar.gz
mv velero-v1.7.0-linux-amd64/velero /usr/local/bin/
rm -rf velero-v1.7.0-linux-amd64.tar.gz
rm -rf velero-v1.7.0-linux-amd64
```


## Set up server

These instructions start the Velero server and a Minio instance that is accessible from within the cluster only. See [Expose Minio outside your cluster][31] for information about configuring your cluster for outside access to Minio. Outside access is required to access logs and run velero describe commands.

### 1. Create a Velero-specific credentials file (credentials-velero) in your local directory:

````bash
cat > credentials-velero <<- EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
````
### 2. Start the server and the local storage service. In the Velero directory, run:
```bash 
echo "create minio service"
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/velero/main/examples/minio/00-minio-deployment.yaml
```
**Note:** The example Minio yaml provided uses “empty dir”. Your node needs to have enough space available to store the data being backed up plus 1GB of free space. If the node does not have enough space, you can modify the example yaml to use a Persistent Volume instead of “empty dir”

### 3. Install velero in kubernetes cluster
```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.1.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc.cluster.local:9000
```

Cehck velero installation objects
```bash
kubectl logs deployment/velero -n velero
kubectl -n velero get all
```

This example assumes that it is running within a local cluster without a volume provider capable of snapshots, so no VolumeSnapshotLocation is created (--use-volume-snapshots=false). You may need to update AWS plugin version to one that is compatible with the version of Velero you are installing.

Additionally, you can specify --use-restic to enable restic support, and --wait to wait for the deployment to be ready.


### 4. Deploy the example nginx application:
```bash
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/velero/main/examples/nginx-app/base.yaml
```
Check to see that both the Velero and nginx deployments are successfully created:

```bash
kubectl get deployments -l component=velero --namespace=velero
kubectl get deployments --namespace=nginx-example
```

## Backup
### 1. Create a backup for any object that matches the app=nginx label selector"
```
velero backup create nginx-backup --selector app=nginx
```
### 2. (Optional) Create regularly scheduled backups based on a cron expression using the app=nginx label selector:
```
velero schedule create nginx-daily --schedule="0 1 * * *" --selector app=nginx
```
### 3. Check backup
```bash
velero backup describe nginx-backup
velero backup logs nginx-backup
velero get backup
```

### 4.Simulate a disaster:
```bash
kubectl delete namespace nginx-example
```

### 5.To check that the nginx deployment and service are gone, run:
```bash
kubectl get deployments --namespace=nginx-example
kubectl get services --namespace=nginx-example
kubectl get namespace/nginx-example
```
You should get no results.

**NOTE:** You might need to wait for a few minutes for the namespace to be fully cleaned up.


## Restore

### 1.Run
```bash
velero restore create --from-backup nginx-backup
```
### 2.check restore
```bash
velero restore check 
```
After the restore finishes, the output looks like the following:
```bash
NAME                          BACKUP         STATUS      WARNINGS   ERRORS    CREATED                         SELECTOR
nginx-backup-20170727200524   nginx-backup   Completed   0          0         2017-07-27 20:05:24 +0000 UTC   <none>
```

### 3.check all object on ns

```bash
kubectl get all -n nginx-example
velero backup describe nginx-backup
velero backup logs nginx-backup
velero get backup
```
