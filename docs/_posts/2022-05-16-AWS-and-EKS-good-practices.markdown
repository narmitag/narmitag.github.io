---
layout: post
title:  "AWS and EKS Best Practices"
date:   2022-05-16 08:11:08 +0100
categories: AWS EKS
---

A list of things I think are best practices for AWS

Set a MFA on all accounts
Use SSO when possible
For EKS if you are planning use PVC's on EBS make sure your worker groups only span one AZ
Monitor cost closely - AWS Free tier is not free
User multiple accounts and organisations to control them
CA vs Karpenter


