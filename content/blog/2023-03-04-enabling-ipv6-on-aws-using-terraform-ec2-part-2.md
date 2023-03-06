---
title: Enabling IPv6 on AWS using Terraform - EC2 "Pet" Instance (Part 2)
author: Colin Barker
date: 2023-03-04T14:26:18.0000Z
description: During my experiments with IPv6, I had an EC2 instance that I use to proxy traffic from AWS into my personal network. It is a "pet" one off instance and I wanted to enable IPv6 using Terraform. Not quite as simple as I expected.
tags:
  - aws
  - networking
  - ipv6
  - terraform
  - ec2
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2023/02/nasa-Q1p7bh3SHj8-unsplash.jpg
---

Header photo by [NASA](https://unsplash.com/@nasa?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/Q1p7bh3SHj8?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

⚠️ **Note**: This is Part 2 of my IPv6 on AWS series, the [first part is available here](/blog/2023-02-11-enabling-ipv6-on-aws-using-terraform/). ⚠️

## What is a "pet" instance, and why?

Before I start, this is an exceptional use case. When using AWS you should always use ephemeral instances, it is always best practice. There are times however, you might need to do this to a singular instance for any number of very valid reasons. In my case, I have a very cheap proxy service between the outside world any my home network. I use this external service as a very cheap method to enable access to some systems that can be accessed through a Site-to-Site VPN to my home. You might have other reasons too, for example - Microsoft Active Directory servers which are hosted on EC2 instances would be considered "pet".

In my case, I have this one static instance using an Elastic IPv4 IP, and I would like to give it an IPv6 address.

### Pets vs Cattle Analogy

{{< quote author="Bill Baker" source="Twitter" url="https://twitter.com/randybias/status/444306871545892864">}}
cattle, not pets
{{< /quote >}}

This phase used during a presentation about Scaling SQL Servers way back in 2006, to show how attitudes towards computing evolved since the early days.

It describes the idea that servers can be two types:

- **Pets**: These are your pride and joys, there is just one [Oz the Cat that sits on your lap while writing blog posts](#oz-the-cat) about IPv6, you look after them, nurture them, and you deal with everything that comes their way as and when it happens.
- **Cattle**: You have a farm, you have a vast amount of animals that help you produce several products that you sell to the market. If one of those animals becomes an issue, you "replace" them. No sentimental attachment.

I will say, this analogy will probably not last the test of time - the latter "Cattle" explanation can upset a few people for different reasons and you probably will see this change in the future. Hoping for something like _"Cooker vs Food"_ or _"House vs Tent"_, but I can't see that happening any time soon!

So with that in mind, you can see the following:

- **Pet Instances**: They have a set hostname, they have had loads of love, care, and attention given to them. When they go wrong, you investigate, identify the issues, remediate, and bring back to health. They also make lots of noises when you need to give them attention. (Yes, [Oz is meowing at me at the moment](#oz-the-cat)!)
- **Cattle Instances**: They have a randomly generated unique identifier, you keep the safe and secure, but once the instance starts to fail, you quickly take them out of the loop, and replace with a healthy instance. The failed instance, you just get rid of.

## Starting Point

Before we begin, we need to set the scene a little. What we will be working with is incredibly simple - an EC2 instance sitting in a Public Subnet.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/03/IPv6EC2StartingPoint.png" caption="Very basic setup of an EC2 instance in a public subnet" gallery="gallery" >}}

You can find the code to deploy the above diagram on my [GitHub - IPv6 on AWS repo](https://github.com/mystcb/ipv6-on-aws/tree/main/03-sample-vpc-with-ec2). Feel free to follow along in your own sandbox account if you wish!

The main block of code we need to start looking at is the `ec2_instance` resource, there are some comments inline to explain what we are doing.

```terraform
# Build the EC2 instance using Amazon Linux 2
resource "aws_instance" "test" {
  ami                     = data.aws_ami.amazon_linux_2.id   # Dynamically chosen Amazon Linux AMI
  ebs_optimized           = true                             # EBS Optimised instance
  instance_type           = "t4g.nano"                       # Using a Graviton based instance here
  disable_api_termination = true                             # Always good practice to stop pet instances being terminated

  # Networking settings, setting the private IP to the 10th IP in the subnet, and attaching to the right SG and Subnets
  source_dest_check      = false
  private_ip             = cidrhost(aws_subnet.public_a.cidr_block, 10)
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.test_sg.id]

  # This requires that the metadata endpoint on the instance uses the new IMDSv2 secure endpoint
  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  # Sets the size of the EBS root volume attached to the instance
  root_block_device {
    volume_size           = "8"      # In GB
    volume_type           = "gp3"    # Volume Type
    encrypted             = true     # Always best practice to encrypt
    delete_on_termination = true     # Make sure that the volume is deleted on termination
  }

  # Name of the instance, for the console
  tags = {
    "Name" = "Sample-EC2-Instance"
  }

  # Ensures the Internet Gateway has been setup before deploying the instance
  depends_on = [
    aws_internet_gateway.sample_igw
  ]
}
```

Nice and simple really, but this is where simple can cause an issue in this case.

## The Issue

Without going to too much detail, as you know [Terraform uses a State](https://developer.hashicorp.com/terraform/language/state) that will store details about the environment it manages, including the settings and configuration of resources. It will use this information to keep track of what it knows, and what has changed - giving it the great ability to see what will change during the next application of your code. This state is generated from the outputs of the providers API's and your code.

However, the API call for creating an EC2 instance does more than just create a single EC2 resource. This also happens with multiple services, and multiple cloud providers as well, so it isn't specific to AWS on this case. The single API call makes the process a lot simpler to build up the compute for you, but it includes one additional resource: The _Elastic Network Interface_.

In typical use, this is great, it saves you building instances with no networking and then having to mess about with it later on. I would also point out, that a cloud service with no networking access, is no different to getting a physical computer, disconnecting everything except the power, wrapping it in concrete, and then seeing what you can do. I mean it will still run (albeit very hot and probably melt), but you can't then do anything with it, or see what is happening!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/03/IPv6EC2Instance.png" caption="EC2 instance showing the Elastic Network Interface (ENI) attached" gallery="gallery" >}}

Terraform does have a resource for the [Elastic Network Interface (ENI)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_interface.html) however, as the API created this resource itself, attached it to the instance using the parameters specified in the `ec2_instance` resource, Terraform doesn't know about this. Herein lies the issue.

### Adding an IPv6 Address using the CLI

Let's say you don't have your Infrastructure defined as code and you needed to assign an IPv6 address to your instance, thankfully the `awscli` has such a method. Under the `ec2` service type, there is a command to `assign-ipv6-addresses` ([Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/assign-ipv6-addresses.html))

To do this, you would run the following command:

> `aws ec2 assign-ipv6-addresses --ipv6-addresses <address> --network-interface-id eni-007ed4d597d6df6b7`

The one item you need to know is, the `network-interface-id`. While the [ec2_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#primary_network_interface_id) resource, does have this as an output, the only way to get this is after the resource has been created.

### What happens in Terraform?

When using the AWS EC2 API to create the instance, behind the scenes it will create that interface using the settings, and return the Interface ID. As the API's are not supposed to keep the State themselves, it will always consider a change to those settings in the creation block, as a new interface/network settings.

As we define the network settings in our resource for the EC2 instance, Terraform knows there is a change to the state, talks to the AWS EC2 API, that then will require the network interface to be recreated. The only way to do that would be create a new interface, detach the old interface, attach the new interface, and you are done, except, that isn't possible. [AWS Documentation on Network Interfaces](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics) state that "Each instance has a default network interface, called the primary network interface. You cannot detach a primary network interface from an instance." This comes into play with my workaround in this blog later on. Therefore, the only option, is to re-create the instance.

If we were to simply use the `ipv6_addresses` in the `ec2_instance` block, to add a new IPv6 address, Terraform will report the following:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

# aws_instance.test must be replaced
-/+ resource "aws_instance" "test" {

      <<SNIP>>

      ~ instance_initiated_shutdown_behavior = "stop" -> (known after apply)
      ~ instance_state                       = "running" -> (known after apply)
      ~ ipv6_address_count                   = 0 -> (known after apply)
      ~ ipv6_addresses                       = [ # forces replacement
          + "2a05:d01c:b90:ee00::10",
        ]
      + key_name                             = (known after apply)
      ~ monitoring                           = false -> (known after apply)
      + outpost_arn                          = (known after apply)

      <<SNIP>>

    }
```

As you can see, adding the address `forces replacement`. Which for our wonderful pet instance here, is a terrifying thought!

## Workaround - Pseudo Code

⚠️ **Note**: This is not going to be the best way, in all honesty this is a hack more than a workaround, but it does make it easier to bypass the issue. Always consider backups before doing this, always consider that if you need to do this, why do you need to do it? ⚠️

Given that the AWS CLI can add an IPv6 address without rebuilding the instance, as well as the console as well, it does mean that a replacement isn't needed to add the address on. For this, we have to play around a bit with Terraform, run it a few times, to get everything up in sync. To this we will need to:

- Create an `aws_network_interface` resource to match the primary ENI with the IPv6 address
- Import the ENI resource into the Terraform State (requires CLI)
- Apply the changes using Terraform, and confirm the IPv6 address has been attached
- Add the `ipv6_addresses` list to the `ec2_instance` resource block
- Delete the `aws_network_interface` from the Terraform State (requires CLI)
- Delete the `aws_network_interface` resource block from the code
- Run a plan to confirm everything is working

As you can see, there are a few steps to do this, but it does mean that if you ever need to re-deploy the code, this will create an identical copy of the instance (minus data) and matches the code.

This also removes the issue later on if you ever wish to destroy the code. If you were to stop after the Apply when you have the IPv6 address assigned, you will now have an `aws_network_interface` resource block managed by Terraform, but also managed by the EC2 Instance itself. As mentioned before, you cant remove the primary network interface from the EC2 instance, and if you try and run a `destroy`, Terraform will attempt to call the API to tell AWS to delete the interface, an come back with a 400 error.

> `Error: detaching EC2 Network Interface (eni-071f55452ccc18997/eni-attach-0a424f74a43a7a0af): OperationNotPermitted: The network interface at device index 0 cannot be detached.`

So the workaround requires us to complete all the steps to make this work.

## Workaround - Terraform

### Create the ENI

First we need to create the `aws_network_interface` resource block. We will need to try and match as much as we can to what we have defined in the `ec2_instance` block. Below is an example of of this block created using the sample code above.

```terraform
resource "aws_network_interface" "test_eni" {
  subnet_id         = aws_subnet.public_a.id
  # Uses the same IP from the ec2_instance resource
  private_ips       = [cidrhost(aws_subnet.public_a.cidr_block, 10)]
  # The new IPv6 to assign - note that 16 in hexadecimal is 10
  ipv6_addresses    = [cidrhost(aws_subnet.public_a.ipv6_cidr_block, 16)]
  # Same security group from the ec2_instance resource
  security_groups   = [aws_security_group.test_sg.id]
  # Continue with additional settings from the ec2_instance resource
  source_dest_check = false

  # Match the attachment details on the EC2 instance
  attachment {
    instance     = aws_instance.test.id
    device_index = 0
  }

}
```

### Import the ENI into State

Next we need to get the ENI information into the state, to do this we need the network interface ID from AWS. This can be done through the console, or by looking at the State file (if you can). In our example, the ENI is `eni-071f55452ccc18997`. Terraform at the bottom of all of its documentation will have the [command you need to import the resource in](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_interface.html#import), in our case we will need to import this ENI into this resource block

> `terraform import aws_network_interface.test_eni eni-071f55452ccc18997`

### Apply the changes

As we have already created the block with the IPv6 address in it, we should be able to run the `plan` and `apply` to add the IPv6 address. Do remember to check your plan! Make sure that it is doing what you expect, in my case the plan showed:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_network_interface.test_eni will be updated in-place
  ~ resource "aws_network_interface" "test_eni" {
        id                        = "eni-071f55452ccc18997"
      ~ ipv6_address_count        = 0 -> (known after apply)
      ~ ipv6_address_list         = [] -> (known after apply)
      + ipv6_address_list_enabled = false
      ~ ipv6_addresses            = [
          + "2a05:d01c:b90:ee00::10",
        ]
      + private_ip_list_enabled   = false
        tags                      = {}
      ~ tags_all                  = {
          + "Environment" = "Sandbox"
          + "Source"      = "Terraform"
        }
        # (15 unchanged attributes hidden)

        # (1 unchanged block hidden)
    }
```

As shown, the only major change will be that there is an additional IPv6 address assigned to it, but also my default tags are added too.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/03/IPv6EC2InstanceWithIPv6.png" caption="The EC2 instance now has an IPv6 address associated with it" gallery="gallery" >}}

Technically at this point, we have done it - but we still have to sort out our Terraform to prevent future issues.

### Add the IPv6 address to the instance block

Thankfully the block we created, also has the same line that we can copy back into the `aws_instance` block, we can sneak this back in to match the state in AWS>

```terraform
# Build the EC2 instance using Amazon Linux 2
resource "aws_instance" "test" {

  <<SNIP>>

  # Networking settings, setting the private IP to the 10th IP in the subnet, and attaching to the right SG and Subnets
  source_dest_check      = false
  private_ip             = cidrhost(aws_subnet.public_a.cidr_block, 10)
  # IPv6 Address for the instance from the aws_network_interface block
  ipv6_addresses         = [cidrhost(aws_subnet.public_a.ipv6_cidr_block, 16)]
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.test_sg.id]

  <<SNIP>>
}
```

### Delete the ENI from the State

Before we remove the block, we need to [remove the information from the State](https://developer.hashicorp.com/terraform/cli/commands/state/rm). This will prevent us from accidentally deleting the block, running an apply, and getting the 400 error mentioned before! To do this we will need to run the following command:

> `terraform state rm aws_network_interface.test_eni`

Now you can delete the whole `aws_network_interface` block from your code.

### Run a plan to check

With all this complete, we just need to run one very last `terraform plan` to confirm, if everything has gone right then your output should be:

```bash
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```

## Summary

Just would like to point out, this is not the ideal way of doing this. I am sure there are other ways, but this is how I got around the issue. Thankfully in my case I had access to the Terraform State, which made the addition of the `aws_network_interface` a lot easier. This workaround doesn't work too well when you do not have access.

This technique can work not just on AWS, but other cloud providers too - its mainly about the logical steps of importing the unmanaged resource, making the changes, and then releasing it back to AWS to manage.

However, we are reminded here why pet instances are just that, sometimes they can be a bit of a pain, but you nurture them for a reason! In my case, I am just a little bit lazy, and didn't want to have to set everything up again!

Hopefully I will continue this IPv6 series soon, where I will go over a number of other services - to see if we can't push forward with the IPv6 transition.

Any comments or queries will be greatly appreciated!

## Oz The Cat

For anyone that is wondering, this is Oz, he is a lovely cat!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/03/IPv6EC2InstanceOzTheCat.jpg" caption="This is Oz!" gallery="gallery" >}}
