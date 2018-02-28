Cloud provider

https://docs.google.com/document/d/17d4qinC_HnIwrK0GHnRlD1FKkTNdN__VO4TH9-EzbIY/edit

https://github.com/kubernetes/kubernetes/issues/57718#issuecomment-354706425



AWS Prep

- Security Groups (for networking access betwen master/slave)
- Policy roles (for api access to provision EBS/ELB)

**Set a tag on all resources in the form of KubernetesCluster=<cluster name>All instancesOne and only one SG for each instance should be tagged.  This will be modified as necessary to allow ELBs to access the instance**

**Set up IAM Roles for nodesFor the master, you want a policy like this (CloudFormation snippet):         Version: '2012-10-17'          Statement:          - Effect: Allow            Action:            - ec2:\*            - elasticloadbalancing:*            - ecr:GetAuthorizationToken            - ecr:BatchCheckLayerAvailability            - ecr:GetDownloadUrlForLayer            - ecr:GetRepositoryPolicy            - ecr:DescribeRepositories            - ecr:ListImages            - ecr:BatchGetImage            - autoscaling:DescribeAutoScalingGroups            - autoscaling:UpdateAutoScalingGroup            Resource: "*"ec2:* may be overkill here but I haven’t done the work to narrow it down.For nodes:       PolicyDocument:          Version: '2012-10-17'          Statement:          - Effect: Allow            Action:            - ec2:Describe*            - ecr:GetAuthorizationToken            - ecr:BatchCheckLayerAvailability            - ecr:GetDownloadUrlForLayer            - ecr:GetRepositoryPolicy            - ecr:DescribeRepositories            - ecr:ListImages            - ecr:BatchGetImage            Resource: "*"Note the * in ec2:Describe***





Install kubeadm on all hosts (user-data - will run as root)

~~~
apt-get update
apt-get install -y docker.io
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
# Customize kubelet args cluster-domain and cloud-provider
cat <<EOF >/etc/systemd/system/kubelet.service.d/20-cloud-provider.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--cloud-provider=aws"
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
EOF
~~~

Installing cluster on master (user-data - will run as root)

~~~
# install cluster
kubeadm init --pod-network-cidr=192.168.0.0/16 --service-dns-domain=aws.powerodit.ch

# Install network overlay
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
~~~

Join node

```
kubeadm join --token <token> --discovery-token-ca-cert-hash <ca cert hash>
```





User configuraiton of kubectl

~~~
# configure kubectl as normal user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~