---
title: "AWS controllers for Kubernetes"
date: 2022-12-15T11:32:37Z
draft: false
tags: ['kubernetes','AWS','ACK']
---

I struggled to find a working (and simple) example on using ACK so I put this together to create an S3 bucket.

AWS Controllers for Kubernetes (also known as ACK) are built around the Kubernetes extension concepts of Custom Resource and Custom Resource Definitions. You can use ACK to define and use AWS services directly from Kubernetes. This helps you take advantage of managed AWS services for your Kubernetes applications without needing to define resources outside of the cluster.

Say you need to use a AWS S3 Bucket in your application that’s deployed to Kubernetes. Instead of using AWS console, AWS CLI, AWS CloudFormation etc., you can define the AWS S3 Bucket in a YAML manifest file and deploy it using familiar tools such as kubectl. The end goal is to allow users (Software Engineers, DevOps engineers, operators etc.) to use the same interface (Kubernetes API in this case) to describe and manage AWS services along with native Kubernetes resources such as Deployment, Service etc.

![Image alt](/images/ack.png)


**Deploy a test cluster with EKS**

```bash

$ cat cluster.yaml

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ack-cluster
  region: us-east-1
  version: "1.23"
iam:
  withOIDC: true

managedNodeGroups:
  - name: managed-ng-1
    minSize: 1
    maxSize: 2
    desiredCapacity: 1
    instanceType: t3.small
    spot: true


$ eksctl create cluster -f cluster.yml

```

***Install the S3 controller***

```bash
export SERVICE=s3
export RELEASE_VERSION=`curl -sL https://api.github.com/repos/aws-controllers-k8s/$SERVICE-controller/releases/latest | grep '"tag_name":' | cut -d'"' -f4`
export ACK_SYSTEM_NAMESPACE=ack-system
export AWS_REGION=us-east-1

aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

helm install --create-namespace -n $ACK_SYSTEM_NAMESPACE ack-$SERVICE-controller \
  oci://public.ecr.aws/aws-controllers-k8s/$SERVICE-chart --version=$RELEASE_VERSION --set=aws.region=$AWS_REGION

$ kubectl get crd

NAME                                         CREATED AT
buckets.s3.services.k8s.aws                  2022-12-15T11:05:35Z
```

***Add the IAM Permissions to the POD***

```bash
export SERVICE=s3
export ACK_K8S_SERVICE_ACCOUNT_NAME=ack-$SERVICE-controller

export ACK_SYSTEM_NAMESPACE=ack-system
export EKS_CLUSTER_NAME=ack-cluster
export POLICY_ARN=arn:aws:iam::aws:policy/AmazonS3FullAccess

# IAM role has a format - do not change it. you can't use any arbitrary name
export IAM_ROLE_NAME=ack-$SERVICE-controller-role

eksctl create iamserviceaccount \
    --name $ACK_K8S_SERVICE_ACCOUNT_NAME \
    --namespace $ACK_SYSTEM_NAMESPACE \
    --cluster $EKS_CLUSTER_NAME \
    --role-name $IAM_ROLE_NAME \
    --attach-policy-arn $POLICY_ARN \
    --approve \
    --override-existing-serviceaccounts
```


and restart the controller pod to get the new IAM permissions

```bash
$ kubectl get deployments -n $ACK_SYSTEM_NAMESPACE

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
ack-s3-controller-s3-chart               1/1     1            1           7m1s

$ kubectl -n $ACK_SYSTEM_NAMESPACE rollout restart deployment ack-s3-controller-s3-chart
deployment.apps/ack-s3-controller-s3-chart restarted

$  kubectl get pods -n $ACK_SYSTEM_NAMESPACE
NAME                                                      READY   STATUS    RESTARTS   AGE
ack-s3-controller-s3-chart-87c8bbfbc-fjjfk                1/1     Running   0          5m33s

$ kubectl describe pod -n $ACK_SYSTEM_NAMESPACE ack-s3-controller-s3-chart-87c8bbfbc-fjjfk | grep "^\s*AWS_"
      AWS_REGION:                      us-east-1
      AWS_ENDPOINT_URL:
      AWS_STS_REGIONAL_ENDPOINTS:      regional
      AWS_ROLE_ARN:                    arn:aws:iam::484235524795:role/ack-s3-controller-role
      AWS_WEB_IDENTITY_TOKEN_FILE:     /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

***Deploy the S3 bucket***

```bash

$ cat bucket.yml
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: my-ack-s3-bucket-123456789
spec:
  name: my-ack-s3-bucket-123456789
  tagging:
    tagSet:
    - key: myTagKey
      value: myTagValue

$ kubectl apply -f bucket.yml

$ kubectl get bucket
NAME                            AGE
my-ack-s3-bucket-123456789      26m

$ kubectl describe bucket my-ack-s3-bucket-123456789 
Name:         my-ack-s3-bucket-123456789
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  s3.services.k8s.aws/v1alpha1
Kind:         Bucket
Metadata:
  Creation Timestamp:  2022-12-15T11:19:20Z
  Finalizers:
    finalizers.s3.services.k8s.aws/Bucket
  Generation:  1
  Managed Fields:
    API Version:  s3.services.k8s.aws/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:name:
        f:tagging:
          .:
          f:tagSet:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2022-12-15T11:19:20Z
    API Version:  s3.services.k8s.aws/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"finalizers.s3.services.k8s.aws/Bucket":
    Manager:      controller
    Operation:    Update
    Time:         2022-12-15T11:19:21Z
    API Version:  s3.services.k8s.aws/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:ackResourceMetadata:
          .:
          f:ownerAccountID:
          f:region:
        f:conditions:
        f:location:
    Manager:         controller
    Operation:       Update
    Subresource:     status
    Time:            2022-12-15T11:19:21Z
  Resource Version:  8095
  UID:               d973c580-d5ba-4a8c-a1cc-fdb3e2128f6d
Spec:
  Name:  my-ack-s3-bucket-123456789
  Tagging:
    Tag Set:
      Key:    myTagKey
      Value:  myTagValue
Status:
  Ack Resource Metadata:
    Owner Account ID:  484235524795
    Region:            us-east-1
  Conditions:
    Last Transition Time:  2022-12-15T11:19:21Z
    Message:               Resource synced successfully
    Reason:
    Status:                True
    Type:                  ACK.ResourceSynced
  Location:                /my-ack-s3-bucket-123456789
Events:                    <none>
```


***Delete the bucket and the cluster***

```bash

kubectl delete -f bucket.yml

exkctl delete cluster -f cluster.yml
```