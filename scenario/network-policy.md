# Network policy sample

Create an nginx deployment and expose it via a service
To see how Kubernetes network policy works, start off by creating an nginx Deployment.
```bash
kubectl create deployment nginx --image=nginx
```

Expose the Deployment through a Service called nginx.
```
kubectl expose deployment nginx --port=80
```

The above commands create a Deployment with an nginx Pod and expose the Deployment through a Service named nginx. The nginx Pod and Deployment are found in the default namespace.
```bash
kubectl get svc,pod

# sample output
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubernetes          10.100.0.1    <none>        443/TCP    46m
service/nginx               10.100.0.16   <none>        80/TCP     33s

NAME                        READY         STATUS        RESTARTS   AGE
pod/nginx-701339712-e0qfq   1/1           Running       0          35s
```

Test the service by accessing it from another Pod
You should be able to access the new nginx service from other Pods. To access the nginx Service from another Pod in the default namespace, start a busybox container:
```bash
kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
```
In your shell, run the following command:
```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
remote file exists
```

Limit access to the nginx service
To limit the access to the nginx service so that only Pods with the label access: true can query it, create a NetworkPolicy object as follows:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
EOF
```

Test access to the service when access label is not defined
When you attempt to access the nginx Service from a Pod without the correct labels, the request times out:
```bash
kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
```

In your shell, run the command:
```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
wget: download timed out
```

Define access label and test again
You can create a Pod with the correct labels to see that the request is allowed:
```bash
kubectl run busybox --rm -ti --labels="access=true" --image=busybox:1.28 -- /bin/sh
```
In your shell, run the command:
```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
remote file exists
```
[References](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)