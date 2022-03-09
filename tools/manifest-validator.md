# Validating Kubernetes YAML for best practice

## Kubeval
The premise of [kubeval](https://www.kubeval.com) is that any interaction with Kubernetes goes via its REST API.
Hence, you can use the API schema to validate whether a given YAML input conforms to the schema.

### Installations
```bash
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
sudo cp kubeval /usr/local/bin
sudo chmode +x /usr/local/bin/kubeval
```

### Check Yaml file
```bash
kubeval additional-properties.yaml
kubeval --strict additional-properties.yaml
cat my-invalid-rc.yaml | kubeval
```
#### You can test a specific API version using the flag `--kubernetes-version`:
```bash
kubeval --kubernetes-version 1.16.1 base-valid.yaml
``` 

## Kube-score
### [Kube-score](https://github.com/zegl/kube-score) analyses YAML manifests and scores them against in-built checks.
### These checks are selected based on security recommendations and best practices, such as:

- Running containers as a non-root user.
- Specifying health checks for pods.
- Defining resource requests and limits.

### Installation:
```bash
kubectl krew install score
```

### Usage:
```bash 
kube-score score base-valid.yaml
kube-score score base-valid.yaml --output-format ci
```