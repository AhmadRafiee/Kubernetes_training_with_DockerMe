# [Installing Helm](https://helm.sh/docs/intro/install/)
## This guide shows how to install the Helm CLI. 

## From Script
### Helm now has an installer script that will automatically grab the latest version of Helm and install it locally.
### You can fetch that script, and then execute it locally. It's well documented so that you can read through it and understand what it is doing before you run it.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## From Apt (Debian/Ubuntu)
### Members of the Helm community have contributed a Helm package for Apt. This package is generally up to date.
```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## From Snap
### The Snapcrafters community maintains the Snap version of the Helm package:
```bash
sudo snap install helm --classic
```