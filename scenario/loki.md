# [Use Loki to Manage k8s Application Logs](https://www.scaleway.com/en/docs/use-loki-to-manage-k8s-application-logs/)
## Loki - Overview
Scaleway Elements Kubernetes Kapsule is not delivered with an embedded logging feature.
In this tutorial, you will learn how to collect your Kubernetes logs using Loki and Grafana. Loki is a log aggregation system inspired by Prometheus.
We believe that it is easy to operate -especially in a Kubernetes environment- as it does not index the content of the logs but set labels for log streams.
As in a cloud-native environment, Prometheus is one of the most common solutions for monitoring. You can re-use the same labels you have already set for Prometheus. For instance, in Kubernetes, the metadata you are using (object labels) can be used in Loki for scraping logs. If you use Grafana for metrics, using Loki will allow you to have a single point of management for both logging and monitoring.

## Installing Loki with Helm
### 1 . Add the Grafana repository to Helm and update it.
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
### 2 . Install the loki-stack with Helm. We install all the stack in a Kubernetes dedicated namespace named loki-stack. We need to deploy it to your cluster and enable persistence (allow Helm to create a Scaleway block device and attach it to the Loki pod to store its data) using a Kubernetes Persistent Volumes to survive a pod re-schedule:
```bash
helm install loki-stack grafana/loki-stack --create-namespace --namespace loki-stack --set promtail.enabled=true,loki.persistence.enabled=true,loki.persistence.size=10Gi
```
### Install Grafana in the loki-stack namespace with Helm. We also want Grafana to survive a re-schedule so we are enabling persistence too :

```bash
helm install loki-grafana grafana/grafana --set persistence.enabled=true,persistence.type=pvc,persistence.size=10Gi --namespace=loki-stack
```
### You can check if the block devices were correctly created by Kubernetes:
```bash
kubectl get pv,pvc -n loki-stack
```
### Now that both Loki and Grafana are installed in the cluster, check if the pods are correctly running:
```bash
kubectl get pods -n loki-stack
```
### To be able to connect to Grafana you first have to get the admin password:
```bash
kubectl get secret --namespace loki-stack loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
Configure a port-forward to reach Grafana from your web browser:
```bash
kubectl port-forward --namespace loki-stack service/loki-grafana 3000:80
```
### Access http://localhost:3000 to reach the Grafana interface. Login using the admin user and the password you got above.
### Add the Loki source to Grafana (http://loki-stack.loki-stack:3100).


