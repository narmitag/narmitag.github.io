---
title: "Setting up AWS VPN"
date: 2022-12-01T12:43:03Z
draft: false
---

This example will deploy a VPN into an AWS VPC and show how to connect to it using OpenVPN from either a Mac or Linux host. It will use 
Certificates for authentication.



![Image alt](/images/vpn.png)

Terraform source code can be found at [Github](https://github.com/narmitag/terraform-examples/tree/main/vpn)

If will deploy a simple VPC in one AZ and then create a VPN, the required certificates for the VPN will be stored in AWS Parameter Store.