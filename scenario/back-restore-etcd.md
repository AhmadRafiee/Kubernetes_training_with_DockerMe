


## Connection etcd
### Install etcdctl On Ubuntu 16.04/18.04
```bash
etcd_version=v3.4.16
curl -L https://github.com/etcd-io/etcd/releases/download/$etcd_version/etcd-$etcd_version-linux-amd64.tar.gz -o etcd-$etcd_version-linux-amd64.tar.gz
tar xzvf etcd-$etcd_version-linux-amd64.tar.gz
rm etcd-$etcd_version-linux-amd64.tar.gz
cp etcd-$etcd_version-linux-amd64/etcdctl /usr/local/bin/
rm -rf etcd-$etcd_version-linux-amd64
etcdctl version
```
### Get etcd information
```bash 
kubectl describe pod etcd-master -n kube-system
```
### check etcd server on master nodes
```bash
master1_ip=172.60.70.150
master1_ip=172.60.70.151
master1_ip=172.60.70.152
export endpoint="https://172.60.70.150:2379,${master2_ip}:2379,${master3_ip}:2379"
export flags="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
              --cert=/etc/kubernetes/pki/etcd/server.crt \
              --key=/etc/kubernetes/pki/etcd/server.key"
endpoints=$(sudo ETCDCTL_API=3 etcdctl member list $flags --endpoints=${endpoint} \
            --write-out=json | jq -r '.members | map(.clientURLs) | add | join(",")')
```

### verify with these commands
```bash
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} member list
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint status
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint health
sudo ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} alarm list

etcdctl member list $flags --endpoints=${endpoint} --write-out=table
etcdctl endpoint status $flags --endpoints=${endpoint} --write-out=table
```

## Backup | Restore etcd
### Commnads pattern:
```bash
ETCDCTL_API=3 etcdctl --endpoints=[ENDPOINT] --cacert=[CA CERT] --cert=[ETCD SERVER CERT] --key=[ETCD SERVER KEY] snapshot save [BACKUP FILE NAME]
```

### Sample command:
```bash
ETCDCTL_API=3 etcdctl --endpoints ${endpoints} $flags snapshot save kubeme-test
ETCDCTL_API=3 etcdctl --endpoints ${endpoints} $flags snapshot status kubeme-test
ETCDCTL_API=3 etcdctl --endpoints ${endpoints} $flags snapshot restore kubeme-test
```

## A Kubernetes CronJob to Back Up the etcd Data
```bash
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: backup
  namespace: kube-system
spec:
  # Run every six hours.
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            # Same image as in /etc/kubernetes/manifests/etcd.yaml
            image: k8s.gcr.io/etcd-amd64:3.1.12
            env:
            - name: ETCDCTL_API
              value: "3"
            command: ["/bin/sh"]
            args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S_%Z).db"]
            volumeMounts:
            - mountPath: /etc/kubernetes/pki/etcd
              name: etcd-certs
              readOnly: true
            - mountPath: /backup
              name: etcd-backup-dir
          restartPolicy: OnFailure
          nodeSelector:
            node-role.kubernetes.io/master: ""
          tolerations:
          - effect: NoSchedule
            operator: Exists
          hostNetwork: true
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
              type: DirectoryOrCreate
          - name: etcd-backup-dir
            hostPath:
              path: /opt/etcd-backup
              type: DirectoryOrCreate
```

### run job from cronjob

```bash
kubectl create job --from=cronjob/<cronjob-name> <job-name>
```
