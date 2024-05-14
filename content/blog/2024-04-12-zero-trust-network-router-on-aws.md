---
title: Zero Trust Network Router on AWS
author: Colin Barker
date: 2024-04-12T18:41:00.0000Z
description: Zero Trust networks have become a bit of a new feature amongst businesses, and home enthusiasts. In this post I explore the use of Zero Trust networking and how to build it into your AWS Cloud Networking.
tags:
  - aws
  - networking
  - zero-trust
  - terraform
  - routing
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2024/04/sander-weeteling-KABfjuSOx74-unsplash.jpg
---

> Header photo by [Sander Weeteling](https://unsplash.com/@sanderweeteling?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/teal-bookeh-lights-KABfjuSOx74?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)

## What is a "Zero Trust Network"

To put plainly, this is a network that is created that by default has Zero Trust within it, it is based off the idea of a Zero Trust security model which is a specific type of implementation used across different networks.

{{< quote author="Zero trust security model" source="Wikipedia" url="https://en.wikipedia.org/wiki/Zero_trust_security_model">}}
The main concept behind the zero trust security model is "never trust, always verify", which means that users and devices should not be trusted by default
{{< /quote >}}

So, how does this apply to networking? Well the main concept here is, that you create a network that by default disallows traffic that isn't known, verified, or wanted. This can be expanded, to then have all your devices connected into a single network, that by default, only allows traffic to pass between each node if you accept it. This is a concept that is often referred to as a "Zero Trust Network".

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/04/switch-connection.png" caption="Example of a simple network with switches and WiFi connection point" gallery="gallery" >}}

Starting with a basic scenario, above we have a simple office, it has a Data Room with a load of servers, with a couple of Virtual Machines, some Head office computers, a physical server, and a mobile phone connected to a WiFi network. Using this as a base, everything had to be in the same place, but it was all on the same network - devices would talk to each other through the switch, and traffic didn't need to jump over anything other than the switch (excluding the phone!).

As the business expanded, it migrated the on-premise data room to a data centre, the Head Office was moved to a new location, and people had the option of working from home. Additionally, the CTO has asked, "_We need to move to [AWS](https://aws.amazon.com/)_". Networking before this would have been quite complex to set up.

This is where a Zero Trust Network comes into play.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/04/zero-trust-connections.png" caption="Depiction of a Zero Trust Network solution, with AWS hooked in" gallery="gallery" >}}

Let's take a look at the above example. Even though the Head Office move was completed, the data centre had been set up, and there were workloads in AWS, we can see here - the network is still directly connected. With this scenario, each segment is part of the wider network, rather than using a VPN connection to route traffic between them, you can see that it is possible to just talk from one side to another, with the only hop being the Zero Trust network in the middle. Think of the provider as essentially a switch!

In this post, I will concentrate primarily on the Zero Trust Appliance in AWS, and how we can connect to a Zero Trust Network.

## Introduction to Tailscale

[Tailscale](https://tailscale.com/) is one of the many different Software Defined, Zero Trust networking solutions that exist today. Many different providers have different ways of implementing their solutions, but they all are based on the same simple premise, "_never trust, always verify_". Tailscale has [multiple methods](https://tailscale.com/kb/1316/device-add) for adding devices to the network, by default, the Access Control Lists ([ACLs](https://tailscale.com/kb/1018/acls)) will not permit traffic between different devices unless explicitly states in the configuration of the ACL.

For anyone looking at behind the scenes of the technology used at Tailscale, I would recommend "[_How Tailscale Works_](https://tailscale.com/blog/how-tailscale-works)" by Avery Pennarun who wrote how the data plan works, and a couple of examples as to why traditional VPNs might cause latency or even general issues in network connectivity.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/04/tailscale-appliance-demo.png" caption="Our example, Tailscale to connect an Office to AWS" gallery="gallery" >}}

Simply put, Tailscale will act as our "switch" and set up the point-to-point network between an [EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html), and an office where they might have several desktops.

For this post today, I will concentrate specifically on the Appliance that will sit in AWS, and how we would configure this for a customer.

## Building a Zero Trust Network Router on AWS

For this, we will be using a specific set of tooling:

- [Terraform](https://www.terraform.io/) v1.7.5 - An infrastructure as code tool
- [AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) - The translator between Terraform and the AWS API
- [Tailscale Provider](https://registry.terraform.io/providers/tailscale/tailscale/latest/docs) - The translator between Terraform and the Tailscale API

Each "device" that is connected to Tailscale will need to be authenticated and approved before the device will be given access to the network. Even if there is an ACL in place that permits the access, the approval must occur. This can cause an issue when working with Infrastructure as Code (IaC), as the process would need to be automated. This can be overcome by generating a [Tailnet Auth Key](https://tailscale.com/kb/1085/auth-keys) that is used specifically for the launching of the instance, and more specifically allowing your device to be added with "pre-approval". For this, we will create a "[tailnet key](https://registry.terraform.io/providers/tailscale/tailscale/latest/docs/resources/tailnet_key)" resource.

```terraform
# Create an authorization token for the Tailscale router to add itself to the Tailnet
resource "tailscale_tailnet_key" "tailnet_key" {
  preauthorized = true    # Set to true to allow the pre-approval of the device
  expiry        = 7776000 # Time in seconds
  description   = "Tailnet key for the server"
}
```

Simply put, this generates the key as a resource, and then Terraform knows what the key is, and any other properties that will have been shown by the API call to create the key.

With this key, the next step would be to install the Tailscale client onto the EC2 instance and make it into a router for any services within AWS. This would need to be done in a few stages when working with IaC, so we should start right at the beginning.

Tailscale offers a very comprehensive guide on [how to install the tailscale client](https://tailscale.com/kb/1347/installation) on a vast number of devices, including Linux, macOS, Windows, iOS, Android, and even down to Amazon Fire Devices, and Chromebooks too! What we are doing is creating a [Subnet Router](https://tailscale.com/kb/1019/subnets).

The Subnet Router is slightly different on a Zero Trust Network like this, while it is usually recommended to install the client on every single server, client, and virtual machine in the organisation, sometimes - it's not needed. Even more so for organisations that use the Cloud, and have a LOT of ephemeral devices, and where the Software Defined Network of the Cloud Provider already has the security in place that keeps your network [Well-Architected](https://aws.amazon.com/architecture/well-architected/).

Within Terraform, there is a function called [`templatefile`](https://developer.hashicorp.com/terraform/language/functions/templatefile) that can be used to generate strings or blocks of text that can use variables that are generated from within Terraform and then used within the string or block of text. Here, we are using the output of the `tailscale_tailnet_key` resource above, and pushing the generated key value into the [`user_data`](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) for an AWS EC2 Instance, to run a script on the first run. Below is the template used to install Tailscale.

```bash
#!/bin/bash

## Set the hostname of the server
hostnamectl hostname ${hostname}

## Ensure that the server is up to date with all the current packges
DEBIAN_FRONTEND=noninteractive sudo apt update -y
DEBIAN_FRONTEND=noninteractive sudo apt upgrade -y

## Enable IP Forwarding on the router to ensure that packets will flow as required
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

## Install Tailscale from the source
curl -fsSL https://tailscale.com/install.sh | sh

## Start Tailscale with the authorisation key to add it to the network
sudo tailscale up --authkey=${ts_authkey} --advertise-routes=${local_cidrs} --accept-routes
```

There is quite a bit happening in the script, it can be summarised as follows:

- *Set the hostname for the instance* - While this is an EC2 instance, and usually in the cloud you would probably use more ephemeral devices, a Subnet Router will need to act more like a "[pet](https://twitter.com/randybias/status/444306871545892864)" so that Tailscale sees this as an appliance object. Setting the hostname will mean that it is recognisable as to which host this is within your Tailscale network
- *Update the instance using apt* - Pretty simple, make sure the instance is running the latest updates and patches before continuing!
- *Set IP Forwarding on the device* - This is a key step, especially so in Linux, as this is to enable the EC2 instance to take traffic that it receives and forward it on. This [specific setting](https://unix.stackexchange.com/questions/673573/what-exactly-happens-when-i-enable-net-ipv4-ip-forward-1) tells the networking device at a kernel level to route traffic through it, as by default this is switched off.
- *Install Tailscale* - The primary install of Tailscale, taken from the latest version on the Tailscale site.
- *Startup tailscale WITH the keys* - Here we finally get to see where our tailscale key will be used - using the up function, we can set the authkey generated before, as well as which routes to advertise. We will go into this in a second, but for now, this is where the key will go.

Simple enough script, now we have to move on to creating the EC2 instance that will run

```terraform
# Use a data object to get the latest version of Ubuntu
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-arm64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"]
}

# Build the Tailscale router using Ubuntu 22.04 LTS
resource "aws_instance" "this" {

  # General setup of the instance using the Ubuntu AMI
  ami                     = data.aws_ami.ubuntu.id
  ebs_optimized           = true
  instance_type           = "t4g.small"
  disable_api_termination = true

  # Key for SSH Access (if required)
  key_name = try(var.ssh_key_name, null)

  # Run the user_data for this instance to install Tailscale
  user_data = templatefile("${path.module}/templates/tailscale-install.sh.tpl",
    {
      ts_authkey  = tailscale_tailnet_key.this.key
      hostname    = var.hostname
      local_cidrs = var.local_cidrs
    }
  )

  # Networking Settings
  source_dest_check      = false # Disabled to allow IP forwarding from the network
  private_ip             = var.private_ip
  ipv6_addresses         = var.ipv6_addresses
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.this.id]

  .... (snipped)

}
```

A lot happening in this section, but in this example, we are using [AWS Graviton](https://aws.amazon.com/ec2/graviton/) (ARM-based) instances, as they are known to be a lot more efficient than other processor types, they can be cheaper, but also currently on [AWS's Free Tier](https://aws.amazon.com/ec2/instance-types/t4/) for the time being!

- *The data object for the `aws_ami`* - This will look for the latest version of the Ubuntu 22.04 image that exists in the Canonical account. Note the `arm64` element of the filter string to look for the ARM version of the AMI.
- *The [`aws_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) resource to generate the subnet router* - The standard for any EC2 instance, the primary resource with all its configuration
- *The general setup of the EC2 resource* - Here we are using the `data.aws_ami.ubuntu.id` to ensure the right AMI is set, additionally making sure that `api_termination` has been configured, as we don't want someone accidentally deleting the instance!
- *An SSH key* - Some people might want to ensure they have an SSH key so they can log into the instance, in several cases this might not be needed.
- *User data template* - This part is where we are taking the template bash file above, and taking the variables it knows about and entering details of what Terraform knows. Here we can see the `ts_authkey` variable is being set to the previously created `tailscale_tailnet_key` resource. Additional settings are also entered here.
- *Network Settings* - This one contains one key variable that **MUST** be set for any EC2-based router that is created. The `source_dest_check` variable must be set to `false`. The [source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics) is on by default, and within the software-defined VPC network on AWS, it will ensure that the traffic that sees, is the traffic for it - when it acts as a router, it will expect to see traffic pass through it from different devices that it needs to forward on. This check is disabled to allow this to happen. Without it, there is no way for traffic destined for another part of the network to flow to the EC2 instance.

Once this resource is launched, then it will appear in your Tailscale network, be approved, and should also be able to route traffic to and from your AWS network.

Some other elements that make up the design for this router, for example, you will need [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html) set up to ensure traffic is allowed inbound and outbound to the EC2 instance and networks. To make this whole process easier, I created a [Terraform Module](https://developer.hashicorp.com/terraform/tutorials/modules/module) that does all of this for you!

> [https://github.com/mystcb/terraform-aws-tailscale-router](https://github.com/mystcb/terraform-aws-tailscale-router)

In Part 2 of this blog post series, I will go into how to use this module to create a Multi-AZ version of this set up, and also include the changes I will be made to the module to enable Instance Recovery if the Operating System stops responding.

## Examples of other Zero Trust Networking Solutions

- [CloudFlare Zero Trust](https://developers.cloudflare.com/cloudflare-one/) - This has more recently had an update that will allow access through one of its [WARP](https://developers.cloudflare.com/warp-client/) clients. This is still in beta so one to keep an eye on. 
- [Enclave](https://enclave.io/) - This was my first ever experience with Zero Trust networking, so I can't not name-drop this one! While personally, I don't use it anymore, it was here I learnt the basics of the Zero Trust network before moving myself to Tailscale.
