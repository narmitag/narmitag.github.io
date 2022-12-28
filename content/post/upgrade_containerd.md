---
title: "Upgrade Containerd on Ubuntu 20.04"
date: 2022-12-12T13:05:14Z
draft: false
tags: ['kubernetes']
categories: ['Guides']
---


Kubernetes 1.26 requires Containerd > 1.6 but the highest version in the Ubuntu 20.04 repos is 1.5.x. The following instructions will get a 20.04 host ready to upgrade to Kubernetes 1.26.

The following instructions assume you are running as ```root```

```bash

mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install containerd.io

systemctl restart containerd

mv /etc/containerd/config.toml /etc/containerd/config.toml.orig
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

systemctl restart containerd
```