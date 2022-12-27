---
title: "Setting up AWS Transit Gateway with Terraform"
date: 2022-12-02T12:44:02Z
draft: false
tags: ['AWS', 'Transit Gateway']
---

[Transit Gateway](https://aws.amazon.com/transit-gateway/) allows VPC's to be connected together into a single network as well as connecting to on-prem networks.

This example will deploy a simple setup with 2 VPC's being connected together.

![Image alt](/images/tgw.png)

The Terraform code can be found on [GitHub](https://github.com/narmitag/terraform-examples/tree/main/transit_gateway). The code also includes a RAM (Resource Access Manager) share for linking VPC's in separate accounts but in this examples it's not used.

Just update any the reqruired settings in ```variables.tf``` then deploy

```bash
terraform init
terraform plan
terraform apply
```


To remove the example
```bash
terraform destroy
```