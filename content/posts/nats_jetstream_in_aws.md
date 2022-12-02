---
title: "Deploying Nats Jetstream in AWS"
date: 2022-12-01T12:43:36Z
draft: false
tags: ['AWS']
---
![Image alt](/images/nats-horizontal-color.png) NATS is a connective technology that powers modern distributed systems. A connective technology is responsible for addressing, discovery and exchanging of messages that drive the common patterns in distributed systems; asking and answering questions, aka services/microservices, and making and processing statements, or stream processing.

An example installation using Terraform can be found on [GitHub](https://github.com/narmitag/terraform-examples/tree/main/nats_jetstream), this deploys a VPC and the supporting infrastructure required along with a AWS instance running Ubuntu with NAT's installed. By default it enables [NKEY](https://docs.nats.io/running-a-nats-service/nats_admin/security/jwt?q=nkey#what-are-nkeys) authentication and stores both the User and Seed Keys in the AWS Parameter Store.