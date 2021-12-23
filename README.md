# Terraform/Terragrunt Setup for Multi-Cloud

This repo contains all of the necessary information to get multi cloud K3s up and going for my [Homelab-K3s-Cluster](https://github.com/danmanners/homelab-k3s-cluster).

## Setup (Fedora/RHEL/CentOS)

You'll need several tools and utilities to get everything up and going. This list _may not_ be exhaustive.

```bash
# Install the Azure CLI
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=dnf
### Import the Signing Key
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
### Add the repository
echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/azure-cli.repo
### Install the CLI
sudo dnf install azure-cli -y

# Install the Google Cloud CLI
# https://cloud.google.com/sdk/docs/install#rpm
### Install the Repo
sudo tee -a /etc/yum.repos.d/google-cloud-sdk.repo << EOM
[google-cloud-sdk]
name=Google Cloud SDK
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el8-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOM
### Install the Google Cloud CLI
sudo dnf install google-cloud-sdk -y

# Install the AWS CLI
sudo dnf install awscli -y

## Log into and/or configure each of the cloud providers
### Google Cloud
gcloud init
gcloud auth application-default login
### Microsoft Azure
az login
### Amazon Cloud
aws configure

# Install Terraform
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf -y install terraform

# Install Terragrunt
sudo wget https://github.com/gruntwork-io/terragrunt/releases/download/v0.35.16/terragrunt_linux_amd64 -O /usr/local/bin/terragrunt
sudo chmod a+x /usr/local/bin/terragrunt
```

# Quick Setup

```bash
terraform init
terraform apply -var-file=environment/cloud.tfvars

# Join the hosts to the ZeroTier network
for resource in $(tf output | grep "tpi" | awk '{gsub(/"/, ""); print $1","$3}'| xargs echo -n); do
  # Set your SSH Username below
  SSH_USER="danmanners"

  # Set your Zerotier Network ID below
  ZT_NETID="NOT_REAL_ZEROTIER_ID"

  # Break apart the variables from their comma-separated values
  CLOUD_HOST="$(echo "$resource" | awk -F, '{print $1}')"
  CLOUD_IP="$(echo "$resource" | awk -F, '{print $2}')"

  # Set the hostname for the host
  ssh $SSH_USER@$CLOUD_IP -t "sudo hostnamectl set-hostname --static $CLOUD_HOST"

  # Install Zerotier and join the network
  ssh $SSH_USER@$CLOUD_IP -t "curl -s https://install.zerotier.com | sudo bash && \
  sudo zerotier-cli join $ZT_NETID"

  # Disable the snapd service; this can take up a **TON** of system resources on smaller VMs.
  ssh $SSH_USER@$CLOUD_IP -t "sudo systemctl stop snapd && \
  sudo systemctl disable snapd"

  # Add DNS to the ZeroTier hosts on their ZT interfaces
  ssh $SSH_USER@$CLOUD_IP -t 'sudo systemd-resolve -i $(ip l show | grep zt | awk '\''{gsub(/:/,""); print $2}'\'') --set-dns=10.45.0.1'

  # Install K3s
  ssh $SSH_USER@$CLOUD_IP -t "sudo wget https://github.com/k3s-io/k3s/releases/download/v1.21.0%2Bk3s1/k3s \
    -O /usr/local/bin/k3s && \
    sudo chmod a+x /usr/local/bin/k3s"

  # Finally, copy over the systemd file and set it up
  scp files/k3s-node.service $SSH_USER@$CLOUD_IP:/tmp/k3s-node.service
  ssh $SSH_USER@$CLOUD_IP -t "sudo mv /tmp/k3s-node.service /etc/systemd/system/k3s-node.service && \
  sudo chown root:root /etc/systemd/system/k3s-node.service && \
  sudo systemctl daemon-reload && \
  sudo mkdir -p /etc/rancher/k3s"
done
```

Then, manually SSH to each host and create the `/etc/rancher/k3s/config.yaml` file. Get the Zerotier interface with:

```bash
ip l show | grep zt | awk '{gsub(/:/,""); print "flannel-iface: "$2}'
```

Finally, we can start everything up:

```bash
for resource in $(tf output | grep "tpi" | awk '{gsub(/"/, ""); print $1","$3}'| xargs echo -n); do
  # Set your SSH Username below
  SSH_USER="danmanners"

  # Break apart the variables from their comma-separated values
  CLOUD_HOST="$(echo "$resource" | awk -F, '{print $1}')"
  CLOUD_IP="$(echo "$resource" | awk -F, '{print $2}')"

  # Enable and start the k3s service
  ssh $SSH_USER@$CLOUD_IP -t "sudo systemctl enable --now k3s-node"
done
```

After about 60 seconds, you should see your new hosts from `kubectl get nodes`:

```bash
k get nodes
NAME                STATUS   ROLES                  AGE     VERSION
...
tpi-k3s-do-edge-1   Ready    <none>                 10m     v1.21.0+k3s1
tpi-k3s-do-edge-2   Ready    <none>                 3m49s   v1.21.0+k3s1
...
```

Once Traefik is up and going, get the HTTP/HTTPS ports, install NGINX, copy over the config with the appropriate variables, and run `sudo systemctl enable --now nginx`. This will ensure that the hosts are actually listening on ports 80 and 443 as regular web traffic expects.
