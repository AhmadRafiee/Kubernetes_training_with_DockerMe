# Install NGINX Ingress Controller
### If you are running an old version of Kubernetes (1.18 or earlier), please read this paragraph for specific instructions.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/cloud/deploy.yaml
```
### Pre-flight check
A few pods should start in the ingress-nginx namespace:

```bash
# check pod
kubectl get pods --namespace=ingress-nginx

# check pod and service
kubectl get all -n ingress-nginx
```
### After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

# Install cert-manager
### Below you will find details on various scenarios we aim to support and that are compatible with the documentation on this website. Furthermore, the most applicable install methods are listed below for each of the situations.

### Default static install
You don’t require any tweaking of the cert-manager install parameters.
The default static configuration can be installed as follows:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```
### Pre-flight check
A few pods should start in the cert-manager namespace:

```bash
# check pod
kubectl get pods -n cert-manager

# check pod and service
kubectl get all -n cert-manager
```


## Issuer Configuration
The first thing you’ll need to configure after you’ve installed cert-manager is an issuer which you can then use to issue certificates.

This section documents how the different issuer types can be configured. You might want to read more about Issuer and ClusterIssuer resources here.

cert-manager comes with a number of built-in certificate issuers which are denoted by being in the cert-manager.io group. You can also install external issuers in addition to the built-in types. Both built-in and external issuers are treated the same and are configured similarly.

When using ClusterIssuer resource types, ensure you understand the purpose of the Cluster Resource Namespace; this can be a common source of issues for people getting started with cert-manager.

### Create clusterissuer

before creating ingress change `YOUR_EMAIL_ADDRESS` on this manifest.

```bash
cat > clusterissuer.yml <<- EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lets-encrypt
spec:
  acme:
    email: <YOUR_EMAIL_ADDRESS>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: lets-encrypt
    solvers:
    - http01:
        ingress:
          class: nginx
          serviceType: ClusterIP
EOF
kubectl create -f ./clusterissuer.yml
kubectl get clusterissuer
```

# Sample ingress manifest
### Create http ingress for weavescope service

before creating ingress change `SUB.DOMAIN.TLD` on this manifest.
```bash
cat > weave-ingress.yml <<- EOF
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: weave
  namespace: weave
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: <SUB.DOMAIN.TLD>
    http:
      paths:
      - backend:
          serviceName: weave-scope-app
          servicePort: 80
        path: /
EOF

kubectl create -f ./weave-ingress.yml
kubectl -n weave get ingress
curl -I http://SUB.DOMAIN.TLD
```
### Create https ingress and certificate for weavescope service

before creating ingress change `SUB.DOMAIN.TLD` on this manifest.

```bash
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"    
    cert-manager.io/cluster-issuer: "lets-encrypt"
    certmanager.k8s.io/acme-http01-edit-in-place: "false"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
  name: weavescope
  namespace: weave
spec:
  rules:
  - host: <SUB.DOMAIN.TLD>
    http:
      paths:
      - backend:
          serviceName: weave-scope-app
          servicePort: 80
        path: /
  tls:
  - hosts:
    - <SUB.DOMAIN.TLD>
    secretName: scope-abrman-tls-secret
EOF
kubectl create -f ./weave-ingress.yml
kubectl -n weave get ingress
curl -I https://SUB.DOMAIN.TLD
```
## **NOTE:** Be sure to make the DNS record before applying.