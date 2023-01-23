### HPA Requirements

create metric server on kube
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

check metric server
```bash
kubectl get all -n kube-system | grep metrics-server
```

metric server test
```bash
kubectl top pod
kubectl top node
```
fix metric server issue
```bash
# add these configuration on deployment manifest metric server
kubectl edit deployment metrics-server
        - --kubelet-insecure-tls
        - --authorization-always-allow-paths=/livez,/readyz
```
### VPA Requirements:
https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

