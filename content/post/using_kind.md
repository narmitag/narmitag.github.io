---
title: "Using kind to Test Kubernetes" # Title of the blog post.
date: 2023-01-13T13:57:45Z # Date of post creation.
 
featured: true # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
thumbnail: "/images/kind.png" # Sets thumbnail image appearing inside card on homepage.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Guides
tags:
  - kind
  - kubernetes
# comment: false # Disable comment if false.
---

**[kind](https://kind.sigs.k8s.io/) is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.**

It requires **Docker** to be installed (it may work with [podman](https://podman.io/) but I've not tried) and the **kind** binary which can be installed in various ways

```bash
brew install kind
or
sudo port selfupdate && sudo port install kind
or
choco install kind
or
go install sigs.k8s.io/kind@v0.17.0
```
 A basic cluster can be spun up with ```kind create cluster```

 ```bash
 ➜  kind (⎈ |N/A:default) kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
➜  kind (⎈ |kind-kind:default) kubectl get nodes
NAME                 STATUS     ROLES           AGE   VERSION
kind-control-plane   NotReady   control-plane   15s   v1.25.3
➜  kind (⎈ |kind-kind:default)
```

Then deleted with ```kind delete cluster```

```bash
➜  kind (⎈ |kind-kind:default) kind delete cluster
Deleting cluster "kind" ...
```


More complex configurations can be defined via a yaml file. Full details on the options can be found [here](https://kind.sigs.k8s.io/docs/user/configuration/)

## A control plane node and 3 workers

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

The yaml file is used via the --config option

```bash
➜  kind (⎈ |N/A:default) kind create cluster --config=basic.yml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
➜  kind (⎈ |kind-kind:default) kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   33s   v1.25.3
kind-worker          Ready    <none>          13s   v1.25.3
kind-worker2         Ready    <none>          13s   v1.25.3
kind-worker3         Ready    <none>          13s   v1.25.3
```

## A specific Kubernetes version

```bash
kind create cluster --image"kindest/node:v1.24.1"
```