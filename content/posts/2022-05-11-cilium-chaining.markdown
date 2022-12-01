---
categories: AWS EKS
date: "2022-05-11T10:38:08Z"
title: EKS with Cilium in chaining mode
tags: ['AWS', 'EKS']
---
[Cilium](https://cilium.io/) can run in chaining mode which allows it to run alongside the AWS-CNI plugin.

This is based on using a sandbox AWS account <link> 

The supporting files can be found on [Github]( https://github.com/narmitag/eks/tree/main/cilium)

Install some extra tools
```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}


export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```

Deploy an EKS Cluster

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=cilium-cluster2
export ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export CILIUM_NAMESPACE=kube-system

eksctl create cluster -f - << EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.22"

availabilityZones: ["${AWS_DEFAULT_REGION}a","${AWS_DEFAULT_REGION}b"]
managedNodeGroups:
  - instanceType: t3.medium
    name: ${CLUSTER_NAME}-ng
    desiredCapacity: 2
    minSize: 1
    maxSize: 2

EOF

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region=${AWS_DEFAULT_REGION}


```

Install Cilium in chaining mode

```bash

helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.11.3 \
  --namespace kube-system \
  --set cni.chainingMode=aws-cni \
  --set enableIPv4Masquerade=false \
  --set tunnel=disabled \
  --set hubble.listenAddress=":4244" \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

  [cloudshell-user@ip-10-0-84-41 cilium]$ helm install cilium cilium/cilium --version 1.11.3 \
>   --namespace kube-system \
>   --set cni.chainingMode=aws-cni \
>   --set enableIPv4Masquerade=false \
>   --set tunnel=disabled \
>   --set hubble.listenAddress=":4244" \
>   --set hubble.relay.enabled=true \
>   --set hubble.ui.enabled=true
W0512 14:20:59.634900    8693 warnings.go:70] spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[1].matchExpressions[0].key: beta.kubernetes.io/os is deprecated since v1.14; use "kubernetes.io/os" instead
W0512 14:20:59.634940    8693 warnings.go:70] spec.template.metadata.annotations[scheduler.alpha.kubernetes.io/critical-pod]: non-functional in v1.16+; use the "priorityClassName" field instead
NAME: cilium
LAST DEPLOYED: Thu May 12 14:20:58 2022
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble Relay and Hubble UI.

Your release version is 1.11.3.

For any further help, visit https://docs.cilium.io/en/v1.11/gettinghelp
[cloudshell-user@ip-10-0-84-41 cilium]$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS              RESTARTS   AGE
kube-system   aws-node-5lwk4                     1/1     Running             0          9m43s
kube-system   aws-node-8b8zs                     1/1     Running             0          9m36s
kube-system   cilium-hm8mf                       0/1     Init:0/2            0          5s
kube-system   cilium-operator-66bff48676-6kzpb   0/1     ContainerCreating   0          5s
kube-system   cilium-operator-66bff48676-xdmlx   0/1     ContainerCreating   0          5s
kube-system   cilium-pcjmw                       0/1     Init:0/2            0          4s
kube-system   coredns-7f5998f4c-crxct            1/1     Running             0          19m
kube-system   coredns-7f5998f4c-zf6df            1/1     Running             0          19m
kube-system   hubble-relay-8548d8946c-x9jpj      0/1     ContainerCreating   0          5s
kube-system   hubble-ui-5f7cdc86c7-nfbkm         0/3     ContainerCreating   0          5s
kube-system   kube-proxy-89sjz                   1/1     Running             0          9m43s
kube-system   kube-proxy-smp5s                   1/1     Running             0          9m36s
[cloudshell-user@ip-10-0-84-41 cilium]$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS     RESTARTS   AGE
kube-system   aws-node-5lwk4                     1/1     Running    0          9m54s
kube-system   aws-node-8b8zs                     1/1     Running    0          9m47s
kube-system   cilium-hm8mf                       1/1     Running    0          16s
kube-system   cilium-operator-66bff48676-6kzpb   1/1     Running    0          16s
kube-system   cilium-operator-66bff48676-xdmlx   1/1     Running    0          16s
kube-system   cilium-pcjmw                       0/1     Init:1/2   0          15s
kube-system   coredns-7f5998f4c-jg5zx            1/1     Running    0          7s
kube-system   coredns-7f5998f4c-zf6df            1/1     Running    0          19m
kube-system   hubble-relay-8548d8946c-x9jpj      1/1     Running    0          16s
kube-system   hubble-ui-5f7cdc86c7-nfbkm         3/3     Running    0          16s
kube-system   kube-proxy-89sjz                   1/1     Running    0          9m54s
kube-system   kube-proxy-smp5s                   1/1     Running    0          9m47s
[cloudshell-user@ip-10-0-84-41 cilium]$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   aws-node-5lwk4                     1/1     Running   0          9m58s
kube-system   aws-node-8b8zs                     1/1     Running   0          9m51s
kube-system   cilium-hm8mf                       1/1     Running   0          20s
kube-system   cilium-operator-66bff48676-6kzpb   1/1     Running   0          20s
kube-system   cilium-operator-66bff48676-xdmlx   1/1     Running   0          20s
kube-system   cilium-pcjmw                       0/1     Running   0          19s
kube-system   coredns-7f5998f4c-jg5zx            1/1     Running   0          11s
kube-system   coredns-7f5998f4c-zf6df            1/1     Running   0          19m
kube-system   hubble-relay-8548d8946c-x9jpj      1/1     Running   0          20s
kube-system   hubble-ui-5f7cdc86c7-nfbkm         3/3     Running   0          20s
kube-system   kube-proxy-89sjz                   1/1     Running   0          9m58s
kube-system   kube-proxy-smp5s                   1/1     Running   0          9m51s

```

Install the cluster autoscaler

```bash

  eksctl utils associate-iam-oidc-provider \
    --cluster $CLUSTER_NAME \
    --approve

aws iam create-policy   \
  --policy-name ${CLUSTER_NAME}-k8s-asg-policy \
  --policy-document file://k8s-asg-policy.json

eksctl create iamserviceaccount \
  --name cluster-autoscaler \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/${CLUSTER_NAME}-k8s-asg-policy" \
  --approve \
  --override-existing-serviceaccounts

envsubst < cluster-autoscaler-autodiscover.yaml | kubectl apply -f -

kubectl -n kube-system annotate deployment.apps/cluster-autoscaler  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

export AUTOSCALER_VERSION=1.22.2
kubectl -n kube-system \
    set image deployment.apps/cluster-autoscaler \
    cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v${AUTOSCALER_VERSION}
```



Check everything is up and running

```bash
kubectl port-forward -n $CILIUM_NAMESPACE svc/hubble-relay --address 0.0.0.0 --address :: 4245:80 &

hubble --server localhost:4245 status
hubble --server localhost:4245 observe


[cloudshell-user@ip-10-0-84-41 cilium]$ hubble --server localhost:4245 status
Handling connection for 4245
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 1,336/8,190 (16.31%)
Flows/s: 4.46
Connected Nodes: 2/2
[cloudshell-user@ip-10-0-84-41 cilium]$ hubble --server localhost:4245 observe
Handling connection for 4245
May 12 14:25:43.500: 192.168.37.166:39544 <- kube-system/coredns-7f5998f4c-6tln5:8080 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:43.500: 192.168.37.166:39542 -> kube-system/coredns-7f5998f4c-6tln5:8080 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:43.500: 192.168.37.166:39544 -> kube-system/coredns-7f5998f4c-6tln5:8080 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:44.236: 192.168.37.166:42346 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: SYN)
May 12 14:25:44.236: 192.168.37.166:42346 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: SYN, ACK)
May 12 14:25:44.236: 192.168.37.166:42346 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK)
May 12 14:25:44.236: 192.168.37.166:42348 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: SYN)
May 12 14:25:44.236: 192.168.37.166:42348 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: SYN, ACK)
May 12 14:25:44.236: 192.168.37.166:42348 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK)
May 12 14:25:44.237: 192.168.37.166:42348 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:44.238: 192.168.37.166:42346 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:44.239: 192.168.37.166:42348 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:44.239: 192.168.37.166:42348 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:44.239: 192.168.37.166:42346 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:44.239: 192.168.37.166:42346 <- ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:44.239: 192.168.37.166:42348 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:44.239: 192.168.37.166:42346 -> ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:10254 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:45.375: ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:40654 <- 192.168.108.176:443 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:45.376: ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:40654 -> 192.168.108.176:443 to-stack FORWARDED (TCP Flags: ACK)
May 12 14:25:48.079: kube-system/cluster-autoscaler-85d889ffd8-q789m:55780 <- 54.239.31.45:443 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.079: kube-system/cluster-autoscaler-85d889ffd8-q789m:55780 -> 54.239.31.45:443 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.079: kube-system/cluster-autoscaler-85d889ffd8-q789m:55780 -> 54.239.31.45:443 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:48.080: kube-system/cluster-autoscaler-85d889ffd8-q789m:55780 <- 54.239.31.45:443 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:48.080: kube-system/cluster-autoscaler-85d889ffd8-q789m:55780 -> 54.239.31.45:443 to-stack FORWARDED (TCP Flags: ACK)
May 12 14:25:48.168: kube-system/cluster-autoscaler-85d889ffd8-q789m:52022 -> 192.168.94.37:443 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.179: kube-system/cluster-autoscaler-85d889ffd8-q789m:52022 <- 192.168.94.37:443 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.514: 192.168.14.247:56902 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: SYN)
May 12 14:25:48.514: 192.168.14.247:56902 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: SYN, ACK)
May 12 14:25:48.514: 192.168.14.247:56902 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: ACK)
May 12 14:25:48.514: 192.168.14.247:56902 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.514: 192.168.14.247:56904 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: SYN)
May 12 14:25:48.514: 192.168.14.247:56904 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: SYN, ACK)
May 12 14:25:48.514: 192.168.14.247:56904 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: ACK)
May 12 14:25:48.514: 192.168.14.247:56902 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.514: 192.168.14.247:56902 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:48.514: 192.168.14.247:56902 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:48.514: 192.168.14.247:56904 -> kube-system/coredns-7f5998f4c-r84mj:8080 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.514: 192.168.14.247:56904 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: ACK, PSH)
May 12 14:25:48.514: 192.168.14.247:56904 <- kube-system/coredns-7f5998f4c-r84mj:8080 to-stack FORWARDED (TCP Flags: ACK, FIN)
May 12 14:25:52.119: ingress-nginx/ingress-nginx-controller-54d8b558d4-gmz56:40654 <- 192.168.108.176:443 to-endpoint FORWARDED (TCP Flags: ACK, PSH)

```