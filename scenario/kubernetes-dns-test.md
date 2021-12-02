# Kubernetes DNS 

# Pod DNS Record
## The following DNS resolution:
```bash
<POD-IP-ADDRESS>.<namespace-name>.pod.cluster.local

# Example
# Pod is located in a default namespace
10-244-1-10.default.pod.cluster.local
```

## Test
```bash
# To create a namespace
$ kubectl create ns apps

# To create a Pod
$ kubectl run nginx --image=nginx --namespace apps

# To get the additional information of the Pod in the namespace "apps"
$ kubectl get po -n apps -owide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          99s   10.244.1.3   node01   <none>           <none>

# To get the dns record of the nginx Pod from the default namespace
kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- nslookup 10-244-0-8.apps.pod.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10-244-1-3.apps.pod.cluster.local
Address 1: 10.244.1.3
pod "test" deleted

# Accessing with curl command
kubectl run -it nginx-test --image=nginx --rm --restart=Never -- curl -Is http://10-244-0-8.apps.pod.cluster.local
HTTP/1.1 200 OK
Server: nginx/1.19.2
```

# Service DNS Record
### The following DNS resolution:
```bash
<service-name>.<namespace-name>.svc.cluster.local
# Example
# Service is located in a default namespace
web-service.default.svc.cluster.local
```
### Pod, Service is located in the apps namespace
```bash
# Expose the nginx Pod
$ kubectl expose pod nginx --name=nginx-service --port 80 --namespace apps
service/nginx-service exposed

# Get the nginx-service in the namespace "apps"
$ kubectl get svc -n apps
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.174   <none>        80/TCP    6s

# To get the dns record of the nginx-service from the default namespace
kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- nslookup nginx-service.apps.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-service.apps.svc.cluster.local
Address 1: 10.96.120.174 nginx-service.apps.svc.cluster.local
pod "test" deleted

# Accessing with curl command
$ kubectl run -it nginx-test --image=nginx --rm --restart=Never -- curl -Is http://nginx-service.apps.svc.cluster.local
HTTP/1.1 200 OK
Server: nginx/1.19.2

# With the host command, we will get fully qualified domain name (FQDN).
host web-service
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default.svc
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default.svc.cluster.local
# web-service.default.svc.cluster.local has address 10.106.112.101

kubectl run -it --rm --restart=Never test-pod --image=busybox -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
# pod "test-pod" deleted

kubectl get pods -o wide
# NAME      READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
# test-pod   1/1     Running   0          11m     10.244.1.3   node01   <none>           <none>
# nginx      1/1     Running   0          10m     10.244.1.4   node01   <none>           <none>

kubectl exec -it test-pod -- nslookup 10-244-1-4.default.pod.cluster.local
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      10-244-1-4.default.pod.cluster.local
# Address 1: 10.244.1.4 

$ kubectl get service
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP   85m
web-service   ClusterIP   10.106.112.101   <none>        80/TCP    9m

$ kubectl exec -it test-pod -- nslookup web-service.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-service.default.svc.cluster.local
Address 1: 10.106.112.101 web-service.default.svc.cluster.local
```