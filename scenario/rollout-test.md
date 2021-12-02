# [Deployment and rollout](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
### Create the Deployment by running the following command:
```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```
- check deployment
```bash
kubectl get deployments
```
- To see the Deployment rollout status
```bash
kubectl rollout status deployment/nginx-deployment
```
- To see the ReplicaSet
```bash
kubectl get rs
```
-  see the labels automatically generated for each Pod
```bash
kubectl get pods --show-labels
```
- Let's update the nginx Pods to use the nginx:1.16.1 image instead of the nginx:1.14.2 image.

```bash
kubectl --record deployment.apps/nginx-deployment set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
# OR
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
```
- check deployment
```bash
kubectl get deployments
```
- To see the ReplicaSet
```bash
kubectl get rs
```
- Get details of your Deployment:
```bash
kubectl describe deployments
```
- check the revisions of this Deployment
```bash
kubectl rollout history deployment.v1.apps/nginx-deployment
```
- Scaling a Deployment 
```bash
kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
```
- You can make as many updates as you wish, for example, update the resources that will be used:
```bash
kubectl set resources deployment.v1.apps/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```
‍- Get details of your Deployment:

```bash
kubectl describe deployment nginx-deployment
```