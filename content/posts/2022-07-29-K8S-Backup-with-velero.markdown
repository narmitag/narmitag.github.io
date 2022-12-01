---
categories: AWS K8S velero
date: "2022-07-29T10:14:08Z"
title: Kubernetes  Backup with Velero
tags: ['AWS', 'EKS']
---

[Velero](https://velero.io/) can being used for backup and recovery - 

# Installation via Terraform

```terraform
resource "aws_s3_bucket" "velero" {
  bucket = "eks-velero-backup-${var.environment_name}"
  acl    = "private"
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
  versioning {
    enabled = true
  }
}

resource "aws_s3_bucket_policy" "velero" {
    bucket = aws_s3_bucket.velero.id

    policy = jsonencode({
        Version = "2012-10-17"
        Id      = "velero-${var.environment_name}-bucket-policy"
        Statement = [
            {
                Sid       = "EnforceTls"
                Effect    = "Deny"
                Principal = "*"
                Action    = "s3:*"
                Resource = [
                    "${aws_s3_bucket.velero.arn}/*",
                    "${aws_s3_bucket.velero.arn}",
                ]
                Condition = {
                    Bool = {
                        "aws:SecureTransport" = "false"
                    }
                    NumericLessThan = {
                        "s3:TlsVersion": 1.2
                    }
                }
            },
        ]
    })
}

module "velero" {
  source  = "DNXLabs/eks-velero/aws"
  version = "0.1.2"

  enabled = true

  cluster_name                     = module.eks.cluster_id
  cluster_identity_oidc_issuer     = module.eks.cluster_oidc_issuer_url
  cluster_identity_oidc_issuer_arn = module.eks.oidc_provider_arn
  aws_region                       = var.region
  create_bucket                    = false
  bucket_name                      = "eks-velero-backup-${var.environment_name}"
  helm_chart_version               = "2.30.1"
}
```


When a new namespace is created a daily scheduled backup is included in the namespace

```bash
velero schedule create $NAMESPACE-backup --schedule "0 7 * * *" -n $NAMESPACE

➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero schedule get
NAME                 STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR
29681-sys-backup     Enabled   2022-07-28 15:31:07 +0100 BST   0 3 * * *   0s           6h ago        <none>
backup-demo-backup   Enabled   2022-07-28 08:19:28 +0100 BST   0 3 * * *   0s           6h ago        <none>
dev-backup           Enabled   2022-07-28 09:59:44 +0100 BST   0 3 * * *   0s           6h ago        <none>
dev-update-backup    Enabled   2022-07-28 09:38:01 +0100 BST   0 3 * * *   0s           6h ago        <none>
develop-backup       Enabled   2022-07-28 09:29:47 +0100 BST   0 3 * * *   0s           6h ago        <none>
pfs-retry-backup     Enabled   2022-07-28 17:17:55 +0100 BST   0 3 * * *   0s           6h ago        <none>
ui-backup            Enabled   2022-07-28 11:14:28 +0100 BST   0 3 * * *   0s           6h ago        <none>
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default)
```

# Testing

Backup and recovery of a namespace has been tested with a recovery from S3 and EBS snapshots

A deployment has been created in backup-demo

## A schedule was created 
```bash
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero schedule get

NAME                 STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR
29681-sys-backup     Enabled   2022-07-28 15:31:07 +0100 BST   0 3 * * *   0s           6h ago        <none>
backup-demo-backup   Enabled   2022-07-28 08:19:28 +0100 BST   0 3 * * *   0s           6h ago        <none>
dev-backup           Enabled   2022-07-28 09:59:44 +0100 BST   0 3 * * *   0s           6h ago        <none>
dev-update-backup    Enabled   2022-07-28 09:38:01 +0100 BST   0 3 * * *   0s           6h ago        <none>
develop-backup       Enabled   2022-07-28 09:29:47 +0100 BST   0 3 * * *   0s           6h ago        <none>
pfs-retry-backup     Enabled   2022-07-28 17:17:55 +0100 BST   0 3 * * *   0s           6h ago        <none>
ui-backup            Enabled   2022-07-28 11:14:28 +0100 BST   0 3 * * *   0s           6h ago        <none>



```
## A list of backups available
```bash
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero backup get

NAME                                  STATUS            ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
29681-sys-backup-20220729030042       PartiallyFailed   1        0          2022-07-29 04:00:55 +0100 BST   29d       default            <none>
backup-demo-backup-20220729030042     PartiallyFailed   6        0          2022-07-29 04:01:30 +0100 BST   29d       default            <none>
backup-demo-schedule-20220729030042   Completed         0        0          2022-07-29 04:01:17 +0100 BST   29d       default            <none>
backup-demo-schedule-20220728030041   Completed         0        0          2022-07-28 04:00:41 +0100 BST   28d       default            <none>
dev-backup-20220729030042             Completed         0        0          2022-07-29 04:02:09 +0100 BST   29d       default            <none>
dev-update-backup-20220729030042      Completed         0        0          2022-07-29 04:01:56 +0100 BST   29d       default            <none>
develop-backup-20220729030042         Completed         0        0          2022-07-29 04:01:43 +0100 BST   29d       default            <none>
pfs-retry-backup-20220729030042       Completed         0        0          2022-07-29 04:01:05 +0100 BST   29d       default            <none>
ui-backup-20220729030042              Completed         0        0          2022-07-29 04:00:42 +0100 BST   29d       default            <none>
```

## Test namespace deleted

```bash
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) k get ns | grep backup-demo
backup-demo              Active   2d18h

➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) k delete ns backup-demo
namespace "backup-demo" deleted

➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) k get ns | grep backup-demo

```

## Backup restored
```bash
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero restore create  --from-backup backup-demo-schedule-20220729030042
Restore request "backup-demo-schedule-20220729030042-20220729102625" submitted successfully.
Run `velero restore describe backup-demo-schedule-20220729030042-20220729102625` or `velero restore logs backup-demo-schedule-20220729030042-20220729102625` for more details.

➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero restore describe backup-demo-schedule-20220729030042-20220729102625

Name:         backup-demo-schedule-20220729030042-20220729102625
Namespace:    velero
Labels:       <none>
Annotations:  <none>

Phase:                                 InProgress
Estimated total items to be restored:  201
Items restored so far:                 11

Started:    2022-07-29 10:26:27 +0100 BST
Completed:  <n/a>

Backup:  backup-demo-schedule-20220729030042

Namespaces:
  Included:  all namespaces found in the backup
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Existing Resource Policy:   <none>

Preserve Service NodePorts:  auto
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default)
```

## Backup restored

```bash
➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) velero restore describe backup-demo-schedule-20220729030042-20220729102625

Name:         backup-demo-schedule-20220729030042-20220729102625
Namespace:    velero
Labels:       <none>
Annotations:  <none>

Phase:                       Completed
Total items to be restored:  197
Items restored:              197

Started:    2022-07-29 10:26:27 +0100 BST
Completed:  2022-07-29 10:27:03 +0100 BST

Warnings:
  Velero:     <none>
  Cluster:  could not restore, CustomResourceDefinition "certificaterequests.cert-manager.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "certificates.cert-manager.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "ciliumendpoints.cilium.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "ciliumnetworkpolicies.cilium.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "orders.acme.cert-manager.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "schedules.velero.io" already exists. Warning: the in-cluster version is different than the backed-up version.
            could not restore, CustomResourceDefinition "secretagentconfigurations.secret-agent.secrets.forgerock.io" already exists. Warning: the in-cluster version is different than the backed-up version.
  Namespaces: <none>

Backup:  backup-demo-schedule-20220729030042

Namespaces:
  Included:  all namespaces found in the backup
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Existing Resource Policy:   <none>

Preserve Service NodePorts:  auto

➜  ~ (⎈ |arn:aws:eks:eu-west-2:123456789012:cluster/cluster-1:default) k get po -n backup-demo

NAME                                  READY   STATUS    RESTARTS   AGE
admin-ui-b89f6f748-zf649              2/2     Running   0          58s
am-dd47498c5-pbbzh                    1/2     Running   0          58s
consent-and-auth-ui-f9486bbc4-vc5q2   3/3     Running   0          58s
ds-cts-0                              2/2     Running   0          58s
ds-cts-1                              2/2     Running   0          58s
ds-cts-2                              2/2     Running   0          57s
ds-idrepo-0                           2/2     Running   0          57s
ds-idrepo-1                           1/2     Running   0          57s
ds-umarepo-0                          2/2     Running   0          57s
end-user-ui-759f8bb9d7-pnr72          2/2     Running   0          57s
idm-b478d46cb-vvvcf                   1/2     Running   0          57s
ig-c4fc95547-tdcxh                    1/2     Running   0          56s
login-ui-7c8f994cb6-2rlqm             2/2     Running   0          56s
rcs-agent-6f6657cdbb-2tmdb            2/2     Running   0          56s

```



