---
title: "Setting up AWS VPN"
date: 2022-12-01T12:43:03Z
draft: false
---

This example will deploy a [VPN](https://aws.amazon.com/vpn/) into an AWS VPC and show how to connect to it using OpenVPN from either a Mac or Linux host. It will use 
Certificates for authentication, many other authentication options are available.



![Image alt](/images/vpn.png)

Terraform source code can be found at [Github](https://github.com/narmitag/terraform-examples/tree/main/vpn)

If will deploy a simple VPC in one AZ and then create a VPN, the required certificates for the VPN will be stored in AWS Parameter Store.

### Deployment


* Check out the code and update the region in ```variables.tf``` to the desired region.
* Make sure you have valid AWS credentials in the environment or update ```provider.tf``` with the required values.
* Run ```terraform init``` and ```terraform apply```

```bash
➜  vpn git:(main) ✗ (⎈ |N/A:default) terraform init
Initializing modules...

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Reusing previous version of hashicorp/tls from the dependency lock file
- Using previously-installed hashicorp/aws v4.42.0
- Using previously-installed hashicorp/tls v4.0.4

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

➜  vpn git:(main) ✗ (⎈ |N/A:default) terraform apply
data.aws_availability_zones.available: Reading...
data.aws_availability_zones.available: Read complete after 0s [id=us-east-1]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_acm_certificate.ca will be created
  + resource "aws_acm_certificate" "ca" {

....

aws_ec2_client_vpn_network_association.vpn[0]: Still creating... [8m50s elapsed]
aws_ec2_client_vpn_network_association.vpn[0]: Still creating... [9m0s elapsed]
aws_ec2_client_vpn_network_association.vpn[0]: Creation complete after 9m1s [id=cvpn-assoc-0ae03994bfbf5dea9]
╷
│ Warning: Argument is deprecated
│ 
│   with aws_ec2_client_vpn_network_association.vpn[0],
│   on vpn.tf line 155, in resource "aws_ec2_client_vpn_network_association" "vpn":
│  155:   security_groups        = [aws_security_group.vpn.id]
│ 
│ Use the `security_group_ids` attribute of the `aws_ec2_client_vpn_endpoint` resource instead.
╵

Apply complete! Resources: 30 added, 0 changed, 0 destroyed.

Outputs:

VPN-endpoint = "cvpn-endpoint-0b1df2b162a5679e1"
```

### Connecting to the VPN

Install OpenVPN ```brew install openvpn```

Get the parameters
```bash
aws ssm get-parameter --name "dev_vpn_key" --with-decryption \
        --output text --query Parameter.Value > key.txt

aws ssm get-parameter --name "dev_vpn_certificate" --with-decryption \
        --output text --query Parameter.Value > cert.txt

aws ec2 export-client-vpn-client-configuration \
        --client-vpn-endpoint-id cvpn-endpoint-0b1df2b162a5679e1 \
        --output text > vpn.txt
```

Connect to the VPN
```bash
➜  vpn git:(main) ✗ (⎈ |N/A:default) sudo openvpn --config vpn.txt \
        --key key.txt --cert cert.txt

Password:
2022-12-02 09:35:06 WARNING: file 'key.txt' is group or others accessible
2022-12-02 09:35:06 OpenVPN 2.5.7 arm-apple-darwin21.5.0 [SSL (OpenSSL)] [LZO] [LZ4] [PKCS11] [MH/RECVDA] [AEAD] built on Jun  8 2022
2022-12-02 09:35:06 library versions: OpenSSL 1.1.1s  1 Nov 2022, LZO 2.10
2022-12-02 09:35:06 TCP/UDP: Preserving recently used remote address: [AF_INET]3.234.149.235:443
2022-12-02 09:35:06 Socket Buffers: R=[131072->131072] S=[131072->131072]
2022-12-02 09:35:06 Attempting to establish TCP connection with [AF_INET]3.234.149.235:443 [nonblock]
2022-12-02 09:35:06 TCP connection established with [AF_INET]3.234.149.235:443
2022-12-02 09:35:06 TCP_CLIENT link local: (not bound)
2022-12-02 09:35:06 TCP_CLIENT link remote: [AF_INET]3.234.149.235:443
2022-12-02 09:35:06 TLS: Initial packet from [AF_INET]3.234.149.235:443, sid=ee28dcaa 0ddbf149
2022-12-02 09:35:07 VERIFY OK: depth=1, O=demo, CN=dev.vpn.ca
2022-12-02 09:35:07 VERIFY KU OK
2022-12-02 09:35:07 Validating certificate extended key usage
2022-12-02 09:35:07 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
2022-12-02 09:35:07 VERIFY EKU OK
2022-12-02 09:35:07 VERIFY OK: depth=0, O=demo, CN=dev.vpn.server
2022-12-02 09:35:07 Control Channel: TLSv1.2, cipher TLSv1.2 ECDHE-RSA-AES256-GCM-SHA384, peer certificate: 2048 bit RSA, signature: RSA-SHA256
2022-12-02 09:35:07 [dev.vpn.server] Peer Connection Initiated with [AF_INET]3.234.149.235:443
2022-12-02 09:35:08 SENT CONTROL [dev.vpn.server]: 'PUSH_REQUEST' (status=1)
2022-12-02 09:35:08 PUSH: Received control message: 'PUSH_REPLY,route 10.0.0.0 255.255.0.0,route-gateway 10.0.240.1,topology subnet,ping 1,ping-restart 20,ifconfig 10.0.240.2 255.255.255.224,peer-id 0,cipher AES-256-GCM'
2022-12-02 09:35:08 OPTIONS IMPORT: timers and/or timeouts modified
2022-12-02 09:35:08 OPTIONS IMPORT: --ifconfig/up options modified
2022-12-02 09:35:08 OPTIONS IMPORT: route options modified
2022-12-02 09:35:08 OPTIONS IMPORT: route-related options modified
2022-12-02 09:35:08 OPTIONS IMPORT: peer-id set
2022-12-02 09:35:08 OPTIONS IMPORT: adjusting link_mtu to 1626
2022-12-02 09:35:08 OPTIONS IMPORT: data channel crypto options modified
2022-12-02 09:35:08 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2022-12-02 09:35:08 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2022-12-02 09:35:08 Opened utun device utun7
2022-12-02 09:35:08 /sbin/ifconfig utun7 delete
ifconfig: ioctl (SIOCDIFADDR): Can't assign requested address
2022-12-02 09:35:08 NOTE: Tried to delete pre-existing tun/tap instance -- No Problem if failure
2022-12-02 09:35:08 /sbin/ifconfig utun7 10.0.240.2 10.0.240.2 netmask 255.255.255.224 mtu 1500 up
2022-12-02 09:35:08 /sbin/route add -net 10.0.240.0 10.0.240.2 255.255.255.224
add net 10.0.240.0: gateway 10.0.240.2
2022-12-02 09:35:08 /sbin/route add -net 10.0.0.0 10.0.240.1 255.255.0.0
add net 10.0.0.0: gateway 10.0.240.1
2022-12-02 09:35:08 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
2022-12-02 09:35:08 Initialization Sequence Completed
```

You can now connect directly to the endpoints in the private subnet


### Tidy Up

```bash
terraform destroy 
```