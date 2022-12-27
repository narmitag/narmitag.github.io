---
title: "Building a basic kubernetes cluster with kubeadm"
date: 2022-12-09T14:07:31Z
draft: false 
tags: ['kubernetes']
---

![Image alt](/images/k8s-3node.png)

This is just a basic setup installing a 3 node Kubernetes setup on 3 nodes. The nodes can probably be anywhere, AWS, GCP, VMware etc as long as they are running Ubuntu 20.04.

One node needs to be designated as the master and the other 2 as workers.

To start with the pre-requisites need to be installed on all 3 nodes. This can be done via a script or by entering the commands below individually

Download and run the script

```bash
curl https://raw.githubusercontent.com/narmitag/kubernetes/main/basic_setup/setup_machine.sh -o setup_machine.sh
chmod +x setup_machine.sh
sudo ./setup_machine.sh
```

Running the steps:

```bash
#Install containerd
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf 
overlay 
br_netfilter 
EOF
 
sudo modprobe overlay 
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1 
EOF

sudo sysctl --system
sudo apt-get update && sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

#Disable swap 
sudo swapoff -a
sudo sed -e '/swap/ s/^#*/#/' -i /etc/fstab   
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

#Install Kubernetes binaries
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```
The cluster now needs to be bootstrapped from the master

```bash
$ sudo  kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0

$ mkdir -p $HOME/.kube 
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl get no
NAME   STATUS     ROLES           AGE   VERSION
master   NotReady   control-plane   50s   v1.24.0
```

A network plugin needs to be installed to get the node ready (it may take a few minutes for the node to become ready). This will install [calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/)

```bash
$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created

$ kubectl get no
NAME   STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   87s   v1.24.0
```

The workers can now be connected, so get kubeadm to generate the required command on the master.

```bash
$ kubeadm token create --print-join-command
```

The run the command generated above on each worker

```bash
$ sudo kubeadmin join ............
```

The master should now show 3 nodes

```bash

$ kubectl get no
NAME   STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   120s  v1.24.0
worker-1 Ready    <none>          77s   v1.24.0
worker-2 Ready    <none>          97s   v1.24.0
```