# Installing Vagrant/Packer on Ubuntu/Debian
### Add the HashiCorp GPG key.
```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
```
### Add the official HashiCorp Linux repository.
```bash
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
```
### Update and install.
```bash
sudo apt-get update 
sudo apt-get install vagrant
sudo apt-get install packer
```

# Add public box in vagrant
### Using Public Boxes
### Adding a bento box to Vagrant
```bash
vagrant box add bento/ubuntu-18.04
vagrant box add --provider virtualbox bento/ubuntu-16.04
vagrant box add --provider virtualbox bento/ubuntu-18.04
vagrant box add --provider virtualbox bento/ubuntu-20.04
```
### Using a bento box in a Vagrantfile
```bash
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-18.04"
end
```

# Building Boxes
### Requirements: install packer, vagrant and virtualbox

### clone bento project 
```bash
git clone https://github.com/chef/bento.git
``` 
### To build an Ubuntu 18.04 box for only the VirtualBox provider
```bash
cd packer_templates/ubuntu
packer build -only=virtualbox-iso ubuntu-16.04-amd64.json
packer build -only=virtualbox-iso ubuntu-18.04-amd64.json
packer build -only=virtualbox-iso ubuntu-20.04-amd64.json
```
