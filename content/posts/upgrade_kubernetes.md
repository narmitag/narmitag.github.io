---
title: "Upgrade Kubernetes to 1.26"
date: 2022-12-12T13:05:25Z
draft: false
tags: ['kubernetes']
---

The following will update the 3 node cluster build [here]({{< ref "/posts/basic_kubernetes_cluster" >}}) to 1.26. 

Before installing 1.26 the hosts need to be running containerd > 1.6, the Ubuntu 20.04 hosts can be upgraded using the instructions [here]({{< ref "/posts/upgrade_containerd" >}})

### Upgrading the Master node

```bash
$ export RELEASE=1.26.0 

$ sudo apt-get update &&  sudo apt-get install -y --allow-change-held-packages kubeadm=$RELEASE-00

$ kubectl drain k8s-control --ignore-daemonsets

$ sudo kubeadm upgrade plan v$RELEASE

[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.25.3
[upgrade/versions] kubeadm version: v1.26.0
[upgrade/versions] Target version: v1.26.0
[upgrade/versions] Latest version in the v1.25 series: v1.26.0

W1212 10:19:42.929073    4860 configset.go:177] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeproxy.config.k8s.io", Version:"v1alpha1", Kind:"KubeProxyConfiguration"}: strict decoding error: unknown field "udpIdleTimeout"
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     3 x v1.25.4   v1.26.0

Upgrade to the latest version in the v1.25 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.25.3   v1.26.0
kube-controller-manager   v1.25.3   v1.26.0
kube-scheduler            v1.25.3   v1.26.0
kube-proxy                v1.25.3   v1.26.0
CoreDNS                   v1.9.3    v1.9.3
etcd                      3.5.4-0   3.5.6-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.26.0

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no



$ sudo kubeadm upgrade apply v$RELEASE

[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W1212 10:20:27.629919    5162 configset.go:177] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeproxy.config.k8s.io", Version:"v1alpha1", Kind:"KubeProxyConfiguration"}: strict decoding error: unknown field "udpIdleTimeout"
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.26.0"
[upgrade/versions] Cluster version: v1.25.3
[upgrade/versions] kubeadm version: v1.26.0
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.26.0" (timeout: 5m0s)..
......
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.26.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

$ sudo apt-get update &&  sudo apt-get install -y --allow-change-held-packages kubelet=$RELEASE-00 kubectl=$RELEASE-00

$ sudo systemctl daemon-reload

$ sudo systemctl restart kubelet

$ kubectl uncordon k8s-control
 
$ kubectl get nodes

```

### Upgrading the Worker nodes

Note: these need to be followed on both workers nodes

On the master run

```bash
$ kubectl drain worker-1|2 --ignore-daemonsets --force
```

On the worker node

```bash

$ export RELEASE=1.26.0 
$ sudo apt-get update && 
$ sudo apt-get install -y --allow-change-held-packages  kubeadm=$RELEASE-00

$ sudo  kubeadm upgrade node

[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.

$ sudo apt-get update &&  sudo apt-get install -y --allow-change-held-packages kubelet=$RELEASE-00 kubectl=$RELEASE-00

$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
```

Back on the master run

```bash

$ kubectl uncordon worker-1|2
```