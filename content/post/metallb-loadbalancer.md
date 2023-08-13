---
title: "Metallb Loadbalancer" # Title of the blog post.
date: 2023-08-13T10:39:46+01:00 # Date of post creation.
description: "Creating a Bare Metal Load Balancer using Metallb." # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
draft: true # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
thumbnail: "/images/metallb.png" # Sets thumbnail image appearing inside card on homepage.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Guides
tags:
  - kubernetes
  - load_balancers
# comment: false # Disable comment if false.
---

**[MetalLB](https://metallb.universe.tf/) hooks into your Kubernetes cluster, and provides a network load-balancer implementation. In short, it allows you to create Kubernetes services of type LoadBalancer in clusters that don’t run on a cloud provider, and thus cannot simply hook into paid products to provide load balancers.**

This example has been tested on a Ubuntu 1.27 Cluster running on a VMWare ESX host. The IP range 192.168.0.230->192.168.0.240 is available on the local network for use.

Edit Kube Proxy to allow ARP

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \ sed -e "s/strictARP: false/strictARP: true/" | \ kubectl apply -f - -n kube-system
```

Install Metallb via helm
```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb

```

This will deploy a contoller and a DaemonSet onto all the workers

```bash
(base) (⎈|kubernetes-admin@kubernetes:default)➜  ~ kubectl get po -n metallb
NAME                                  READY   STATUS    RESTARTS   AGE
metallb-controller-58c5997556-ldlvj   1/1     Running   0          116m
metallb-speaker-7xx4p                 4/4     Running   0          116m
metallb-speaker-g4b8l                 4/4     Running   0          116m
metallb-speaker-pqpng                 4/4     Running   0          116m
metallb-speaker-vgmk2                 4/4     Running   0          116m
(base) (⎈|kubernetes-admin@kubernetes:default)➜  ~
```

Create a file for the Address pool - changing the address range to match your local setup

```bash

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb
spec:
  addresses:
  - 192.168.0.230-192.168.0.240
```
Apply it
```bash
kubectl apply -f address.yml
```

Create a file to advertise the address pool
```bash
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb
```

and apply it 
```bash
kubectl apply -f advertise.yml
```


Now it can be tested

This will just create a local nginx pod and create a Loadbalancer
```bash
kubectl create deploy nginx --image nginx:latest

kubectl expose deploy nginx --port 80 --type LoadBalancer
```

The Load balancer Ip can then be found 

```bash
(base) (⎈|kubernetes-admin@kubernetes:default)➜  kubectl get svc
NAME                                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes                            ClusterIP      10.96.0.1       <none>          443/TCP        80d
nginx                                 LoadBalancer   10.107.115.48   192.168.0.230   80:30282/TCP   4s
```

