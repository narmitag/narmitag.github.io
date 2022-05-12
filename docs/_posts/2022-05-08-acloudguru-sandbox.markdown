---
layout: post
title:  "Using ACloudGuru AWS Sandboxes"
date:   2022-05-06 14:38:08 +0100
categories: AWS
---
Running your own AWS account for testing can lead to unexpected costs. Unless care is taken around securing the account, the account can be hijacked and used for other purposes. Personally I’ve stopped running my own accounts and moved to using the [sandbox accounts](https://acloudguru.com/platform/cloud-sandbox-playgrounds) provided by acloudguru as part of their Personal Plus subscription.


Most of the examples provided in the blog should run on these playgrounds with some exceptions. Generally the setup is using the [AWS CloudShell](https://aws.amazon.com/cloudshell/) within the sandbox account to remove any issue with local machine setups.

To save repeating the setup on CloudShell - the following commands need to be run to setup tools needed for the examples

{% highlight bash %}
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
sudo yum -y install jq gettext bash-completion moreutils envsubst
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
{% endhighlight %}

[cloudshell.sh](https://github.com/narmitag/eks/blob/8eb92ca5f1883f41be065d0db5b1c1adde2bbeb2/setup/cloudshell.sh)