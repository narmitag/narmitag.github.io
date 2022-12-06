---
title: "SSM Bastion with no NAT Gateway"
date: 2022-12-05T08:43:21Z
draft: false
tags: ['AWS']
---

Using Systems Manager (SSM) to control access to a Bastion host has several advantages making using a Traditional Bastion host using SSH Keys pretty much obsolete.
* No need for an external IP
* No SSH Keys needed all access is via IAM 
* Access logged including what command are run

A working example can be found on [GitHub](https://github.com/narmitag/terraform-examples/tree/main/ssm_bastion)

One of the biggest issues is that it requires access to AWS services to function so either a NAT gateway is required or VPC endpoints are needed

![Image alt](/images/ssm.png)

The following endpoints need to be created to allow SSM to work without a NAT Gateway
* SSM
* SSMMessages
* EC2
* EC2Messages
* KMS
* Logs

```terraform
module "vpc_endpoints" {
    source = "terraform-aws-modules/vpc/aws//modules/vpc-endpoints"
    version = "3.14.2"

    vpc_id             = module.vpc.vpc_id
    security_group_ids = [data.aws_security_group.default.id]

    endpoints = {
     ssm = {
        service             = "ssm"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
        security_group_ids  = [aws_security_group.vpc_tls.id]
      },
      ssmmessages = {
        service             = "ssmmessages"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
                security_group_ids  = [aws_security_group.vpc_tls.id]
      },
      ec2 = {
        service             = "ec2"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
        security_group_ids  = [data.aws_security_group.default.id]
      },
      ec2messages = {
        service             = "ec2messages"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
                security_group_ids  = [aws_security_group.vpc_tls.id]
      },
      kms = {
        service             = "kms"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
        security_group_ids  = [aws_security_group.vpc_tls.id]
      },
      logs = {
        service             = "logs"
        private_dns_enabled = true
        subnet_ids          = module.vpc.private_subnets
        security_group_ids  = [aws_security_group.vpc_tls.id]
      },
    }

}
```

The instance deployed is just a standard AWS AL2 instance with the IAM manged role ```arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore``` attached to it via an instance profile.

The Terraform example here includes things like logging and KMS encryption which could be removed. All the command run on the bastion are recorded in Cloudwatch logs along with the name of the IAM user who ran them.

 
To connect to the host there is a bash script in ```client\ssh_terminal``` which will locate the bastion in AWS and start a session to it. Basically it is running the command

```bash
aws ssm start-session --target "${INSTANCE_ID}" --region "${REGION}"
```

The instance can also be used for Port forwarding, this is a bit hacky but works. This will allow connection from your local machine to a remote host via the bastion.

![Image alt](/images/port_forward.png)

Step 1: Start an SSH session to the bastion and run

```
socat TCP-LISTEN:4222,reuseaddr,fork TCP4:10.200.36.194:4222
```

Step 2: On your local machine start a port forwarding SSM Session
```
 aws ssm start-session --target "${INSTANCE_ID}" \
         --document-name AWS-StartPortForwardingSession \
         --parameters '{"portNumber":["4222"],"localPortNumber":["9999"]}'
 ```

Step 3: Connect to port 9999 locally which will forward via the Bastion to port 4333 on 10.200.36.194