---
categories: AWS EKS
date: "2022-05-12T08:11:08Z"
title: Installing Karpenter
tags: ['AWS', 'EKS']
---

[Karpenter](https://karpenter.sh/) automatically launches just the right compute resources to handle your cluster's applications. It is designed to let you take full advantage of the cloud with fast and simple compute provisioning for Kubernetes clusters. It is a replacement for the Cluster Autoscaler which has some issues in AWS

This at the moment this example does not work on an acloudguru sandbox account. The supporting files can be found on [Github]( https://github.com/narmitag/eks/tree/main/karpenter)
 
Deploy an EKS Cluster
```bash
export CLUSTER_NAME="karpenter-demo"
export AWS_DEFAULT_REGION="eu-west-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"

eksctl create cluster -f - << EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.21"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}
managedNodeGroups:
  - instanceType: t3.small
    name: ${CLUSTER_NAME}-ng
    desiredCapacity: 1
    minSize: 1
    maxSize: 2
iam:
  withOIDC: true
EOF

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region=${AWS_DEFAULT_REGION}
```

Install and setup Karpenter

```bash

export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"

TEMPOUT=$(mktemp)

curl -fsSL https://karpenter.sh/v0.6.3/getting-started/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${CLUSTER_NAME}" \
  --arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --group system:bootstrappers \
  --group system:nodes

  eksctl create iamserviceaccount \
  --cluster "${CLUSTER_NAME}" --name karpenter --namespace karpenter \
  --role-name "${CLUSTER_NAME}-karpenter" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --role-only \
  --approve

export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

aws iam create-service-linked-role --aws-service-name spot.amazonaws.com

helm repo add karpenter https://charts.karpenter.sh/
helm repo update

helm upgrade --install --namespace karpenter --create-namespace \
  karpenter karpenter/karpenter \
  --version v0.6.3 \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set clusterName=${CLUSTER_NAME} \
  --set clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --wait # for the defaulting webhook to install before creating a Provisioner

cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  provider:
    subnetSelector:
      karpenter.sh/discovery: ${CLUSTER_NAME}
    securityGroupSelector:
      karpenter.sh/discovery: ${CLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF
```


Deploy a test workload and watch the nodes deploy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```

```bash
➜  eks git:(main) ✗ (⎈ |neil.armitage@amido.com@karpenter-demo.eu-west-1.eksctl.io:default) kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2022-05-13T10:26:47.308Z        INFO    controller.controller.nodemetrics       Starting workers        {"commit": "fd19ba2", "reconciler group": "", "reconciler kind": "Node", "worker count": 1}
2022-05-13T10:26:47.308Z        INFO    controller.controller.provisioning      Starting workers        {"commit": "fd19ba2", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "worker count": 10}
2022-05-13T10:26:59.993Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:22.951Z        INFO    controller.provisioning Batched 5 pods in 1.176501626s  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:23.156Z        INFO    controller.provisioning Computed packing of 1 node(s) for 5 pod(s) with instance type option(s) [c1.xlarge c4.2xlarge c3.2xlarge c5d.2xlarge c6i.2xlarge c5a.2xlarge c5.2xlarge c6a.2xlarge c5ad.2xlarge c5n.2xlarge m3.2xlarge m5ad.2xlarge t3.2xlarge m5d.2xlarge m5.2xlarge m5zn.2xlarge m5n.2xlarge t3a.2xlarge m5dn.2xlarge m4.2xlarge] {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.448Z        INFO    controller.provisioning Launched instance: i-0addfe38342f0960a, hostname: ip-192-168-111-106.eu-west-1.compute.internal, type: t3.2xlarge, zone: eu-west-1c, capacityType: spot {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.515Z        INFO    controller.provisioning Bound 5 pod(s) to node ip-192-168-111-106.eu-west-1.compute.internal    {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.515Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:26.517Z        INFO    controller.provisioning Batched 1 pods in 1.00078258s   {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:26.519Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
```


Scale the deployment down and watch the nodes terminate

```bash
kubectl delete deployment inflate
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```

```bash
deployment.apps "inflate" deleted
2022-05-13T10:26:47.308Z        INFO    controller.controller.provisioning      Starting workers        {"commit": "fd19ba2", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "worker count": 10}
2022-05-13T10:26:59.993Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:22.951Z        INFO    controller.provisioning Batched 5 pods in 1.176501626s  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:23.156Z        INFO    controller.provisioning Computed packing of 1 node(s) for 5 pod(s) with instance type option(s) [c1.xlarge c4.2xlarge c3.2xlarge c5d.2xlarge c6i.2xlarge c5a.2xlarge c5.2xlarge c6a.2xlarge c5ad.2xlarge c5n.2xlarge m3.2xlarge m5ad.2xlarge t3.2xlarge m5d.2xlarge m5.2xlarge m5zn.2xlarge m5n.2xlarge t3a.2xlarge m5dn.2xlarge m4.2xlarge] {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.448Z        INFO    controller.provisioning Launched instance: i-0addfe38342f0960a, hostname: ip-192-168-111-106.eu-west-1.compute.internal, type: t3.2xlarge, zone: eu-west-1c, capacityType: spot {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.515Z        INFO    controller.provisioning Bound 5 pod(s) to node ip-192-168-111-106.eu-west-1.compute.internal    {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:25.515Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:26.517Z        INFO    controller.provisioning Batched 1 pods in 1.00078258s   {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:28:26.519Z        INFO    controller.provisioning Waiting for unschedulable pods  {"commit": "fd19ba2", "provisioner": "default"}
2022-05-13T10:31:51.183Z        INFO    controller.node Added TTL to empty node {"commit": "fd19ba2", "node": "ip-192-168-111-106.eu-west-1.compute.internal"}
2022-05-13T10:32:21.001Z        INFO    controller.node Triggering termination after 30s for empty node {"commit": "fd19ba2", "node": "ip-192-168-111-106.eu-west-1.compute.internal"}
2022-05-13T10:32:21.031Z        INFO    controller.termination  Cordoned node   {"commit": "fd19ba2", "node": "ip-192-168-111-106.eu-west-1.compute.internal"}
2022-05-13T10:32:21.239Z        INFO    controller.termination  Deleted node    {"commit": "fd19ba2", "node": "ip-192-168-111-106.eu-west-1.compute.internal"}
```

Tidy up the cluster

```bash
helm uninstall karpenter --namespace karpenter
aws iam delete-role --role-name "${CLUSTER_NAME}-karpenter"
aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"
aws ec2 describe-launch-templates \
    | jq -r ".LaunchTemplates[].LaunchTemplateName" \
    | grep -i "Karpenter-${CLUSTER_NAME}" \
    | xargs -I{} aws ec2 delete-launch-template --launch-template-name {}
eksctl delete cluster --name "${CLUSTER_NAME}"
```