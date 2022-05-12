---
layout: post
title:  "Basic EKS Cluster with Cluster Autoscaler"
date:   2022-05-11 10:38:08 +0100
categories: AWS EKS
---
This is based on using a sandbox AWS account <link> 

The supporting files can be found on [Github]( https://github.com/narmitag/eks/tree/main/cluster-autoscaler)




Create an EKS deployment file, I tend to create individual nodegroups dedicated to a single AZ 

{% highlight bash %}

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ cat ca-cluster.yaml

---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ca-cluster
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ca-cluster
  region: us-east-1
  version: "1.22"

availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]

managedNodeGroups:
- name: nodegroupA
  desiredCapacity: 1
  availabilityZones: ["us-east-1a"]
  instanceType: t3.small
  spot: true
- name: nodegroupB
  desiredCapacity: 1
  availabilityZones: ["us-east-1b"]
  instanceType: t3.small
  spot: true
- name: nodegroupC
  desiredCapacity: 1
  availabilityZones: ["us-east-1c"]
  instanceType: t3.small
  spot: true
{% endhighlight %}


Create the cluster (will take a long time)

{% highlight bash %}
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ eksctl create cluster -f ca-cluster.yaml
2022-05-12 08:03:06 [ℹ]  eksctl version 0.96.0
2022-05-12 08:03:06 [ℹ]  using region us-east-1
2022-05-12 08:03:06 [ℹ]  subnets for us-east-1a - public:192.168.0.0/19 private:192.168.96.0/19
2022-05-12 08:03:06 [ℹ]  subnets for us-east-1b - public:192.168.32.0/19 private:192.168.128.0/19
2022-05-12 08:03:06 [ℹ]  subnets for us-east-1c - public:192.168.64.0/19 private:192.168.160.0/19
2022-05-12 08:03:06 [ℹ]  nodegroup "nodegroupA" will use "" [AmazonLinux2/1.22]
2022-05-12 08:03:06 [ℹ]  nodegroup "nodegroupB" will use "" [AmazonLinux2/1.22]
2022-05-12 08:03:06 [ℹ]  nodegroup "nodegroupC" will use "" [AmazonLinux2/1.22]
2022-05-12 08:03:06 [ℹ]  using Kubernetes version 1.22
2022-05-12 08:03:06 [ℹ]  creating EKS cluster "ca-cluster" in "us-east-1" region with managed nodes
2022-05-12 08:03:06 [ℹ]  3 nodegroups (nodegroupA, nodegroupB, nodegroupC) were included (based on the include/exclude rules)
2022-05-12 08:03:06 [ℹ]  will create a CloudFormation stack for cluster itself and 0 nodegroup stack(s)
2022-05-12 08:03:06 [ℹ]  will create a CloudFormation stack for cluster itself and 3 managed nodegroup stack(s)
2022-05-12 08:03:06 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-east-1 --cluster=ca-cluster'
2022-05-12 08:03:06 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "ca-cluster" in "us-east-1"
2022-05-12 08:03:06 [ℹ]  CloudWatch logging will not be enabled for cluster "ca-cluster" in "us-east-1"
2022-05-12 08:03:06 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=us-east-1 --cluster=ca-cluster'
2022-05-12 08:03:06 [ℹ]  
2 sequential tasks: { create cluster control plane "ca-cluster", 
    2 sequential sub-tasks: { 
        wait for control plane to become ready,
        3 parallel sub-tasks: { 
            create managed nodegroup "nodegroupA",
            create managed nodegroup "nodegroupB",
            create managed nodegroup "nodegroupC",
        },
    } 
}
2022-05-12 08:03:06 [ℹ]  building cluster stack "eksctl-ca-cluster-cluster"
2022-05-12 08:03:06 [ℹ]  deploying stack "eksctl-ca-cluster-cluster"
2022-05-12 08:03:36 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:04:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:05:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:06:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:07:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:08:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:09:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:10:06 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:11:07 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:12:07 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:13:07 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:14:07 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-cluster"
2022-05-12 08:16:08 [ℹ]  building managed nodegroup stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:16:08 [ℹ]  building managed nodegroup stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:16:08 [ℹ]  building managed nodegroup stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:16:08 [ℹ]  deploying stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:16:08 [ℹ]  deploying stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:16:08 [ℹ]  deploying stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:16:08 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:16:08 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:16:08 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:16:38 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:16:38 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:16:38 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:17:19 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:17:19 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:17:25 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:18:07 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:18:40 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:18:47 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:19:08 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:20:26 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:20:34 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:20:35 [ℹ]  waiting for the control plane availability...
2022-05-12 08:20:35 [✔]  saved kubeconfig as "/home/cloudshell-user/.kube/config"
2022-05-12 08:20:35 [ℹ]  no tasks
2022-05-12 08:20:35 [✔]  all EKS cluster resources for "ca-cluster" have been created
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupA" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-11-66.ec2.internal" is ready
2022-05-12 08:20:35 [ℹ]  waiting for at least 1 node(s) to become ready in "nodegroupA"
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupA" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-11-66.ec2.internal" is ready
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupB" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-51-71.ec2.internal" is ready
2022-05-12 08:20:35 [ℹ]  waiting for at least 1 node(s) to become ready in "nodegroupB"
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupB" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-51-71.ec2.internal" is ready
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupC" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-73-100.ec2.internal" is ready
2022-05-12 08:20:35 [ℹ]  waiting for at least 1 node(s) to become ready in "nodegroupC"
2022-05-12 08:20:35 [ℹ]  nodegroup "nodegroupC" has 1 node(s)
2022-05-12 08:20:35 [ℹ]  node "ip-192-168-73-100.ec2.internal" is ready
2022-05-12 08:20:35 [✖]  parsing kubectl version string  (upstream error: error: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1"
) / "0.0.0": Version string empty
2022-05-12 08:20:35 [ℹ]  cluster should be functional despite missing (or misconfigured) client binaries
2022-05-12 08:20:35 [✔]  EKS cluster "ca-cluster" in "us-east-1" region is ready
{% endhighlight %}

Download the kubeconfig

{% highlight bash %}
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ aws eks update-kubeconfig --name ca-cluster --region=us-east-1
Added new context arn:aws:eks:us-east-1:566428305604:cluster/ca-cluster to /home/cloudshell-user/.kube/config
{% endhighlight %}

Check the cluster is running ok
{% highlight bash %}
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ kubectl get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-11-66.ec2.internal    Ready    <none>   17m   v1.22.6-eks-7d68063
ip-192-168-51-71.ec2.internal    Ready    <none>   17m   v1.22.6-eks-7d68063
ip-192-168-73-100.ec2.internal   Ready    <none>   17m   v1.22.6-eks-7d68063
{% endhighlight %}

Update the Auto scalling groups to add extra nodes and a limit 
{% highlight bash %}
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='$CLUSTER_NAME']].AutoScalingGroupName" --output text > asg.txt
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ for x in $(cat asg.txt); do aws autoscaling update-auto-scaling-group --auto-scaling-group-name  $x  --min-size 1 --desired-capacity 1 --max-size 2; done
{% endhighlight %}

Create the IAM Policy and ServiceAccount for the cluster autoscaler

{% highlight bash %}

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ aws iam create-policy     --policy-name ${CLUSTER_NAME}-k8s-asg-policy   --policy-document file://k8s-asg-policy.json
{
    "Policy": {
        "PolicyName": "ca-cluster-k8s-asg-policy",
        "PolicyId": "ANPAYHYOCBDCEETZ2NA33",
        "Arn": "arn:aws:iam::566428305604:policy/ca-cluster-k8s-asg-policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-05-12T08:26:23+00:00",
        "UpdateDate": "2022-05-12T08:26:23+00:00"
    }
}

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ eksctl create iamserviceaccount   --name cluster-autoscaler   --namespace kube-system   --cluster $CLUSTER_NAME   --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/${CLUSTER_NAME}-k8s-asg-policy"   --approve   --override-existing-serviceaccounts
{% endhighlight %}


Deploy the CA


{% highlight bash %}

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ envsubst < cluster-autoscaler-autodiscover.yaml | kubectl apply -f -
clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
role.rbac.authorization.k8s.io/cluster-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
deployment.apps/cluster-autoscaler created

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ kubectl -n kube-system annotate deployment.apps/cluster-autoscaler  cluster-autoscaler.kubernetes.io/safe-to-evict="false"
deployment.apps/cluster-autoscaler annotated

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ export AUTOSCALER_VERSION=1.22.2
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ kubectl -n kube-system \
>     set image deployment.apps/cluster-autoscaler \
>     cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v${AUTOSCALER_VERSION}
deployment.apps/cluster-autoscaler image updated

{% endhighlight %}





Delete the cluster

{% highlight bash %}

[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ eksctl delete cluster ca-cluster
2022-05-12 08:36:38 [ℹ]  eksctl version 0.96.0
2022-05-12 08:36:38 [ℹ]  using region us-east-1
2022-05-12 08:36:38 [ℹ]  deleting EKS cluster "ca-cluster"
2022-05-12 08:36:39 [ℹ]  will drain 0 unmanaged nodegroup(s) in cluster "ca-cluster"
2022-05-12 08:36:39 [ℹ]  deleted 0 Fargate profile(s)
2022-05-12 08:36:39 [✔]  kubeconfig has been updated
2022-05-12 08:36:39 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2022-05-12 08:36:48 [ℹ]  
3 sequential tasks: { 
    3 parallel sub-tasks: { 
        delete nodegroup "nodegroupC",
        delete nodegroup "nodegroupA",
        delete nodegroup "nodegroupB",
    }, 
    2 sequential sub-tasks: { 
        2 sequential sub-tasks: { 
            delete IAM role for serviceaccount "kube-system/cluster-autoscaler",
            delete serviceaccount "kube-system/cluster-autoscaler",
        },
        delete IAM OIDC provider,
    }, delete cluster control plane "ca-cluster" [async] 
}
2022-05-12 08:36:48 [ℹ]  will delete stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:36:48 [ℹ]  waiting for stack "eksctl-ca-cluster-nodegroup-nodegroupB" to get deleted
2022-05-12 08:36:48 [ℹ]  will delete stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:36:48 [ℹ]  waiting for stack "eksctl-ca-cluster-nodegroup-nodegroupC" to get deleted
2022-05-12 08:36:48 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:36:49 [ℹ]  will delete stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:36:49 [ℹ]  waiting for stack "eksctl-ca-cluster-nodegroup-nodegroupA" to get deleted
2022-05-12 08:36:49 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:36:49 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:37:18 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:37:19 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:37:19 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:37:51 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:37:54 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:38:09 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:38:29 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:39:49 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:39:54 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:40:14 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:40:32 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:40:51 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:41:12 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:41:35 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:42:13 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:42:46 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:42:57 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:43:05 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:43:35 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:44:23 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:44:47 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupC"
2022-05-12 08:45:04 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupB"
2022-05-12 08:46:00 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-nodegroup-nodegroupA"
2022-05-12 08:46:02 [ℹ]  will delete stack "eksctl-ca-cluster-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-05-12 08:46:02 [ℹ]  waiting for stack "eksctl-ca-cluster-addon-iamserviceaccount-kube-system-cluster-autoscaler" to get deleted
2022-05-12 08:46:04 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-05-12 08:46:34 [ℹ]  waiting for CloudFormation stack "eksctl-ca-cluster-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-05-12 08:46:34 [ℹ]  deleted serviceaccount "kube-system/cluster-autoscaler"
2022-05-12 08:46:34 [ℹ]  1 error(s) occurred while deleting cluster with nodegroup(s)
2022-05-12 08:46:34 [✖]  deleting OIDC provider: operation error IAM: DeleteOpenIDConnectProvider, https response error StatusCode: 403, RequestID: 3944137c-c5ff-4263-8156-db113d89d620, api error AccessDenied: User: arn:aws:iam::566428305604:user/cloud_user is not authorized to perform: iam:DeleteOpenIDConnectProvider on resource: arn:aws:iam::566428305604:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/7B0C3F25F3AC9E9E3C2EEDB41AC61FB9 with an explicit deny in a service control policy
Error: failed to delete cluster with nodegroup(s)
[cloudshell-user@ip-10-1-181-252 cluster-autoscaler]$ 
{% endhighlight %}
