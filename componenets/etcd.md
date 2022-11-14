


### Install etcdctl On Ubuntu
```bash
etcd_version=v3.5.5
curl -L https://github.com/coreos/etcd/releases/download/$etcd_version/etcd-$etcd_version-linux-amd64.tar.gz -o etcd-$etcd_version-linux-amd64.tar.gz
tar xzvf etcd-$etcd_version-linux-amd64.tar.gz
rm etcd-$etcd_version-linux-amd64.tar.gz
cp etcd-$etcd_version-linux-amd64/etcdctl /usr/local/bin/
rm -rf etcd-$etcd_version-linux-amd64
```
### Check etcdctl version
```bash
etcdctl version
```

### Set environment variable for using etcd
```bash
export endpoint="https://127.0.0.1:2379"
#export endpoint="https://${master1_ip}:2379,${master2_ip}:2379,${master3_ip}:2379"
export flags="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
            --cert=/etc/kubernetes/pki/etcd/server.crt \
            --key=/etc/kubernetes/pki/etcd/server.key"
endpoints=$(ETCDCTL_API=3 etcdctl member list $flags --endpoints=${endpoint} \
            --write-out=json | jq -r '.members | map(.clientURLs) | add | join(",")')
```

### Check and get information about etcd cluster
```bash
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} member list
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint status
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints} endpoint health
```

### Get cluster information from etcd cluster
```bash
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints}  --prefix --keys-only get /
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints}  --prefix --keys-only get /registry/pods
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints}  --prefix --keys-only get /registry/daemonsets
ETCDCTL_API=3 etcdctl $flags --endpoints=${endpoints}  --prefix --keys-only get /registry/deployments
```