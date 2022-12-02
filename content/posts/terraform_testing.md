---
title: "Terraform Testing"
date: 2022-12-02T14:59:57Z
draft: false
tags: ['AWS', 'terraform']
---

For a long time I've been wanting to look at some way of testing terraform. As part of this I've recently looked using [terratest](https://terratest.gruntwork.io/) and [localstack](https://localstack.cloud/).

While localstack looks promising lots of the things I wanted to test are either in the pro version or not supported so I went back to using live AWS accounts. It would be nice to be able to use localstack in a pipeline, hopefully in the future. I did get a simple pipeline running in [GitHub Actions](https://github.com/narmitag/localstack)

```bash
name: localstack-action-example
on: push
jobs:
  example-job:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Start LocalStack
        env:
          LOCALSTACK_API_KEY: ${{ secrets.LOCALSTACK_API_KEY }}
        run: |
          pip install --upgrade pyopenssl
          pip install localstack awscli-local[ver1] # install LocalStack cli and awslocal
          docker pull localstack/localstack         # Make sure to pull the latest version of the image
          localstack start -d                       # Start LocalStack in the background
          
          echo "Waiting for LocalStack startup..."  # Wait 30 seconds for the LocalStack container
          localstack wait -t 30                     # to become ready before timing out 
          echo "Startup complete"          
      - name: Run some Tests against LocalStack
        run: |
          awslocal s3 mb s3://test
          awslocal s3 ls
          echo "Test Execution complete!"     
      - name: Terraform
        run: |
          pip install terraform-local
          tflocal init
          ls -la
          tflocal apply --auto-approve
```

Terratest does appear to show some promising results, it has allowed me to wrap some tests around some [Terraform examples](https://github.com/narmitag/terraform-examples/tree/main/test) I've recently put together.

At a basic level it can just a basic init->plan-apply->destroy cycle on a Terraform deployment to check it works

```go

package test

import (
	"testing"
	"github.com/gruntwork-io/terratest/modules/terraform"
)

func TestEksCa(t *testing.T) {

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../eks-ca",
	})

	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

}
```
It can also be expanded to query the deployed infrastructure

```go

package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestVpcPeering(t *testing.T) {

	vpcOwnerPrefix := "10.10"
	vpcAcceptorPrefix := "10.20"
	awsRegion := "us-east-1"
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../vpc_peering",
		Vars: map[string]interface{}{
			"cidr_prefix-a": vpcOwnerPrefix,
			"cidr_prefix-b": vpcAcceptorPrefix,
		},
	})

	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

	vpcIdOwner := terraform.Output(t, terraformOptions, "vpc-owner-id")
	vpcIdAccepter := terraform.Output(t, terraformOptions, "vpc-accepter-id")

	vpcOwner := aws.GetVpcById(t, vpcIdOwner, awsRegion)
	vpcAccepter := aws.GetVpcById(t, vpcIdAccepter, awsRegion)

	assert.NotEmpty(t, vpcOwner.Name)
	assert.NotEmpty(t, vpcAccepter.Name)
}
```

A promising start, it would be nice to pair it with localstack for speed in the future.