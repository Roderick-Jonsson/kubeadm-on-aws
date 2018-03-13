## AWS Setup

Create a policy role with admin access to all services

Create security group allowing all traffic in subnet

Tag all resources `KubernetesCluster=<cluster name>`

Create S3 bucket for kubeadm discovery file for the k8s nodes to join master

## Kubeadm setup

### Master

> BUG: CNI not getting setup???

Installing cluster on master (user-data - will run as root)

<!--Summary what does this script do-->

```bash
#!/bin/bash

# variable values definied by user
S3BUCKET=5001683761 # S3 Bucket user created to exhcange kubeconfig with nodes and cli users
CLUSTERDOMAIN=aws.powerodit.ch # Domain to use inside Kubernetes cluster



# Install AWS CLI
apt-get update && apt-get install python-pip -y
pip install awscli

# Install docker as container provider
apt-get update
apt-get install -y docker.io

# Add kubernetes repository and install kubelet, kubeadm and kubectl
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

# Configure internal cluster domain via kubelet argument
sed -i "s#--cluster-domain=cluster\.local#--cluster-domain=$CLUSTERDOMAIN#g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Add cloud-provider to kubelet arguments
cat <<EOF >/etc/systemd/system/kubelet.service.d/20-cloud-provider.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--cloud-provider=aws"
EOF

# Restart kubelet to load new configuration
systemctl daemon-reload
systemctl restart kubelet

# install master
# Add custom internal cluster domain
kubeadm init --pod-network-cidr=192.168.0.0/16 --service-dns-domain=$CLUSTERDOMAIN

# Workaround: https://github.com/kubernetes/kubernetes/issues/47695
# Summary: Only during cloud init the hostname is not a FQDN, but it will be afterwards and mismatch as node name, hence disabling NodeRestriction admission plugin
sed -i 's#NodeRestriction\,##g' /etc/kubernetes/manifests/kube-apiserver.yaml

# Install network overlay
# We use calico as it supports network policies
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml

# Create join command for nodes
kubeadm token create --description="Infinite ttl token" --ttl=0 --print-join-command > /root/join.cmd

# Upload kubernetes discovery file to s3 bucket for exchange with nodes
aws s3 cp /root/join.cmd s3://$S3BUCKET
```

### Nodes (Slave)

> IMPORTANT: Do not progress until your first master has fully setup! It will take with a t2.medium instance about 230 seconds to run everything through

Install kubeadm on all hosts (user-data - will run as root)

<!--Summary what does this script do-->

~~~bash
#!/bin/bash

# variable values definied by user
# S3 Bucket user created to exhcange kubeconfig with nodes and cli users
S3BUCKET=5001683761



# Install AWS CLI
apt-get update && apt-get install python-pip -y
pip install awscli

# Install docker as container provider
apt-get update
apt-get install -y docker.io

# Add kubernetes repository and install kubelet, kubeadm and kubectl
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

# Create custom kubelet service arguments
# cluster-domain="internal dns name of cluster"
# cloud-provider="aws integration"
cat <<EOF >/etc/systemd/system/kubelet.service.d/20-cloud-provider.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--cloud-provider=aws"
EOF
systemctl daemon-reload
systemctl restart kubelet

# Join node to master
aws s3 cp s3://$S3BUCKET/join.cmd /root/join.cmd
bash join.cmd
~~~

## Kubectl setup

User configuration of kubectl

~~~
# configure kubectl as normal user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

## Sources

Cloud provider

https://docs.google.com/document/d/17d4qinC_HnIwrK0GHnRlD1FKkTNdN__VO4TH9-EzbIY/edit

https://github.com/kubernetes/kubernetes/issues/57718#issuecomment-354706425



Zu Risiken und Nebenwirkungen lesen Sie die Software Dokumentation oder fragen Sie Ihren System Administrator oder eröffnen ein Issue ;)