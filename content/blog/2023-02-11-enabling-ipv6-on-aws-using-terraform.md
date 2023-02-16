---
title: Enabling IPv6 on AWS using Terraform (Part 1)
author: Colin Barker
date: 2023-02-11T15:08:20.0000Z
description: IPv6 is still taking its time to be adopted around the world. AWS has been working since 2011 on enabling IPv6 capabilities, so is it about time you enabled this too?
tags:
  - aws
  - networking
  - ipv6
  - terraform
  - infracost
categories:
  - aws
images:
  - src: https://static.colinbarker.me.uk/img/blog/2023/02/nasa-Q1p7bh3SHj8-unsplash.jpg
    alt: Logical Layout of Blog Setup
---

Header photo by [NASA](https://unsplash.com/@nasa?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/Q1p7bh3SHj8?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## What is IPv6?

According to [Wikipedia](https://en.wikipedia.org/wiki/IPv6), Internet Protocol version 6 (IPv6) was introduced in December 1995 (just over 25 years ago!), based on the original IPv4 protocol that we all know and love today. The [Internet Engineering Task Force (IETF)](https://en.wikipedia.org/wiki/Internet_Engineering_Task_Force) developed this new protocol to help deal with the (at the time) anticipated exhaustion of the IPv4 address range. This process seemed like it could be simple enough, create a new system to replace and older system and enable the expansion of the Internet to meet modern day standards.

This is where the problem lie. Adopting IPv6 wasn't as simple as just replacing IPv4. The two protocols are different enough that on top of physical hardware changes (older devices unable to support IPv6), it also meant a different way of thinking when it came to working with networking both on the Internet, and within (Intranets, local networks, home networks). One of the biggest issues that faced a lot of the world, is that ISP adoption rate was surprisingly low.

However, as the [IPv4 Exhaustion](https://inetcore.com/project/ipv4ec/en-us/index.html) happened as early as 2011, providers have started very quickly adopting the new standard.

### So, what is IPv6?

I would recommend reading the [Wikipedia](https://en.wikipedia.org/wiki/IPv6) for more details, as much of what I would write here, would essentially be copied just from that page! In summary though"

- IPv6 uses 128-bit addresses over the IPv4 32-bit addresses, allowing approximately 3.4 x 10^38 addresses, over IPv4's 4,294,967,296 (2 x 10^32)
- Addresses are in 8 groups of hexadecimal digits, separated by colons. For example: `fd42:cb::123` in short hand. (which would expand to be `fd42:00cb:0000:0000:0000:0000:0000:0123`)
- Route Aggregation is built into the addressing scheme allowing for the expansion of global route tables with minimal space used.
- Steve Leibson (in a now lost article) one said "[If] we could assign an IPv6 address to every atom on the surface of the earth, [we would] still have enough addresses left to do another 100+ Earths!"

⚠️ **Note**: I mentioned short hand above, and this comes with a lot of caveats but the primary rule is, you can drop any zero in an IPv6 address, and it is assumed, as long as it is before the hexadecimal character. For example `00cb:0000:0001` can be shortened to `cb::1`. However, you can only have ONE `::` in each address (otherwise how does it know where to shorten it!), so for example `00cb:0000:0100:0000:0001` must be shortened to `cb:0:100::1`. I'll go into this in more detail in a future post!⚠️

This is why it is important to start embracing IPv6, we have a lot more space to make lives easier for the world, and the only way we can ensure the continued adoption of the protocol is to enable this everywhere.

In this post, I will go through how you can enable, using Terraform, IPv6 on your existing AWS Cloud Networks. When planning IPv6, there is a lot to consider, and there are some architectural changes will need to be considered.

## Setting the scene - an existing AWS Cloud Network

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6BasicVPC.png" caption="Basic AWS VPC with Networking" gallery="gallery" >}}

This should be a very familiar layout to most people, an VPC in AWS with some very basic networking setup. In the diagram above, we have a VPC, with Public and Private Subnets in two availability zones. We have an Internet Gateway for the public subnets, and a NAT Gateway in each Availability Zone for the Private Subnets to talk to the internet. I have placed some sample IP addressing in, just for reference as part of this post.

If you wish to deploy this yourself, I have place some sample code on my [GitHub](https://github.com/mystcb) that you can use. (https://github.com/mystcb/ipv6-on-aws/tree/main/01-sample-vpc)

Throughout this post, you will see me mention the cost of running this using an estimate. I have been using for a while, a tool called `infracost` which is an open source (with subscription based additions) cost estimator tool - https://www.infracost.io/. For this demonstration, using the sample code listed above, it would cost an estimated $76.65/month - so if you don't want rack up a bill, only deploy when you want to test, and use Terraform to destroy the services when you are done.

As an example, here it the cost estimate for this deployment:

```bash
# infracost breakdown --path=.

Evaluating Terraform directory at .
  ✔ Downloading Terraform modules
  ✔ Evaluating Terraform directory
  ✔ Retrieving cloud prices to calculate costs

Project: mystcb/ipv6-on-aws/01-sample-vpc

 Name                                     Monthly Qty  Unit            Monthly Cost

 aws_eip.natgwip[1]
 └─ IP address (if unused)                        730  hours                  $3.65

 aws_nat_gateway.sample_natgw_subnet_a
 ├─ NAT gateway                                   730  hours                 $36.50
 └─ Data processed                      Monthly cost depends on usage: $0.05 per GB

 aws_nat_gateway.sample_natgw_subnet_b
 ├─ NAT gateway                                   730  hours                 $36.50
 └─ Data processed                      Monthly cost depends on usage: $0.05 per GB

 OVERALL TOTAL                                                               $76.65
──────────────────────────────────
```

Remember, all costs are estimates! With the cloud, its pay as you use, utility based so the costs will be what you use. The above is just an estimate, so keep that in mind!

## IPv6 and AWS

As mentioned before, there are some concepts that have to be considered when designing for IPv6. One specific concept for networking can seem a little confusing at first, but with the right security in place, will ensure that no accidental access to the service can happen.

Within AWS, IPv6 addresses are global unicast addresses. A unicast address is an address that identifies a unique object or node on a network. While the premise of a unicast address is not new (as it was the same in IPv6), with the exhaustion of IPv4 addresses, new methods to give unique addresses to multiple nodes was devised. [Network Address Translation (NAT)](https://en.wikipedia.org/wiki/Network_address_translation) was one such method for doing this. This allowed the mapping of a single unicast IP address to be used by multiple nodes, by routing traffic through the NAT node and allowing it to re-write the headers to make it seem like it had come from the unique address.

NAT is a wide subject, and I am sure I will write more about it some day, but a real world example that you see in most places is your home network. You have multiple private nodes with access to the internet, typically using a single public unicast address.

So how does this relate to IPv6 and AWS? Remember earlier, I mentioned that there are so many addresses available in the IPv6 ecosystem that you could give a unique address to every atom on the planet? Well, we can do just that, but to the nodes that require addresses. This is what AWS does, each IPv6 address you assign to nodes in AWS, are global unicast addresses, unique to each node.

This means a "private" IPv6 Subnet does sound like it might be complicated to set up, but actually it isn't as bad as you think! However, to start the process off, we will start with adding a IPv6 to the Public Subnets, to set the ground work, and go from there.

## Enabling IPv6 on your VPC

Regardless of what you need IPv6 for, you need to enable this on your VPC. Just like your IPv4 CIDR that you assign to the VPC, you will need to assign a IPv6 CIDR to your VPC.

⚠️ **Difference**: Private IPv6 addresses do exist, but you can't assign them to AWS VPCs. You must use public CIDR ranges ⚠️

In a way, the above does make sense - every device is globally unique, so why would you need to make a private address. Personally, I use internal private addresses to make it easier to remember when connecting to servers, but I am very much an old-school person here, and you should be using name resolution to connect to instances!

Therefore, when you go to assign an IPv6 CIDR range to your VPC, you have one of three options:

- [IPAM-allocated](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html)
- Amazon-provided
- [An IPv6 CIDR owned by you (BYOIP)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-byoip.html)

For simplicity, I will be using the Amazon-provided CIDR ranges. In the future, I will go over the IPAM-allocated and BYOIP options, but for now these are just additional ways to get an IPv6 CIDR on your VPC.

AWS have access to a fairly large range of IPv6 addresses, this means they can allocate you a unique set of addresses just for you.

⚠️ **Terminology**: A Network Boarder Group is the name (chosen by AWS) that defines a unique set of Availability Zones or Local Zones where AWS advertises IP addresses ⚠️

When requesting an IPv6 CIDR from AWS, they will need you to select a Network Border Group. This is to ensure that the IPv6 addresses you are receiving can be routed successfully to your VPC. Back to the IPv6 description above, to make routing simpler in the IPv6 world, AWS will route specific ranges to specific groups, and therefore you have to select the right group. Thankfully, as the groups are quite large, for the majority of regions - there is only a single Network Border Group, and Terraform will select this automatically!

Lets start with the `vpv.tf` that exists in the sample code (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc.tf).

```HCL
# Creation of the sample VPC for the region
#tfsec:ignore:aws-ec2-require-vpc-flow-logs-for-all-vpcs
resource "aws_vpc" "sample_vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    "Name" = "Sample-VPC"
  }
}
```

Pretty simple, probably a little too simple! But keep in mind that this is just a sample for now!

Terraform has two resource parameters that will be used to set and assign the IPv6 CIDR to the VPC. `assign_generated_ipv6_cidr_block` and `ipv6_cidr_block_network_border_group`. The border group is used a lot more with Local Zones, so we don't need to worry about this at the moment, but do keep this in mind for more complex deployments.

Just setting the `assign_generated_ipv6_cidr_block` to `true` is enough for AWS to assign the VPC a CIDR range. [_Terraform documentation_](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#assign_generated_ipv6_cidr_block).

Your file should now look like this:

```HCL
# Creation of the sample VPC for the region
#tfsec:ignore:aws-ec2-require-vpc-flow-logs-for-all-vpcs
resource "aws_vpc" "sample_vpc" {
  cidr_block                       = var.vpc_cidr
  assign_generated_ipv6_cidr_block = true

  tags = {
    "Name" = "Sample-VPC"
  }
}
```

With this, your `terraform plan` should now look like this:

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_vpc.sample_vpc will be updated in-place
  ~ resource "aws_vpc" "sample_vpc" {
      ~ assign_generated_ipv6_cidr_block     = false -> true
        id                                   = "vpc-0123456789abcdef0"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
        tags                                 = {
            "Name" = "Sample-VPC"
        }
        # (16 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

As you can see with the plan, this will grab some additional details to add to the resource object as attributes to reference later on. This will be key to make your terraform portable, and not hard coded!

Once applied, this will then allocate the IPv6 CIDR to the VPC, and you should then see the following!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6CIDRonVPC.png" caption="Console view of the CIDR allocations on a VPC showing the IPv6 allocation" gallery="gallery" >}}

There we go, we have hit the first step! IPv6 is now enabled on the VPC! As you can see, we have received a `/56` block of IP's. That gives you the room to have a total of `4,722,366,482,869,645,500,000 hosts` in your network. I don't think we will be running out any time soon!

Next step, lets assign to each of the subnets their own CIDR, so that resources in the subnets can assign their own IPv6 address.

## Adding IPv6 CIDRs to the public subnets

Just like IPv4 CIDR's, you can break down the IPv6 range you have into smaller ranges that are all routed internally using the AWS's VPC networking. With a manual set up you would normally do something like this:

- **IPv4 VPC CIDR**: 192.168.0.0/20
- **Public Subnet in AZ1**: 192.168.10.0/24
- **Public Subnet in AZ2**: 192.168.11.0/24

This is shown in the sample VPC we are using in this post. Simple enough right? Breaking down the CIDR into smaller subnets, and then assigning them to the correct location. It is very much the same with IPv6, but the ranges are just that much larger, that sometimes its best to use an automated method for doing this. What everyone should be doing, is the automatic generation of the ranges for IPv4 as well, which is available in the example!

To do this, we can use a terraform function called `cidrsubnet` (https://developer.hashicorp.com/terraform/language/functions/cidrsubnet). This function can calculate the subnet addresses within a given IP network block or CIDR, and then be used as a value for a variable in the `aws_subnet` resource block.

This function takes a bit of getting used to, but here is my trick for understanding how it works!

The function looks like this:

```
cidrsubnet(prefix, newbits, netnum)
```

For further details, feel free to read the documentation above, but for our sample we will use the following:

- `prefix` the CIDR range. Available as an attribute as this is generated by AWS. `aws_vpc.sample_vpc.ipv6_cidr_block`
- `newbits` this is the number of additional bits you need to add to the CIDR prefix, to break the network down. In our case we will use `8` as it is a round number, but you will need to size this to your specifications.
- `netnum` this is the network number assigned to the broken down blocks that you will select for this subnet. It can't be more than the `newbits` and you can't use the same `netnum` on multiple subnets.

The trick I have used is as follows. `newbits` is calculated as the difference between the CIDR's prefix `/56` and the network size you need. So in our case I would like to make each subnet a `/64` in size. The difference between the two (`64 - 56 = 8`) means the bit difference is `8`. For the moment, I will leave this here, but I will create an article in the future that explains how this works, and why its the number of bits!

For the `netnum` I tend to visualise it in the diagram - I wanted 4 subnets in my sample, (2 public, 2 private), so I am able to reference the networks created above by their counter, with the first network being `0`, with the last network being `3` (as your counter started at 0).

So for our networks, we can enter the values and get the following:

```HCL
cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 0) # Subnet 1 - x:x::0:0/64
cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 1) # Subnet 2 - x:x::1:0/64
cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 2) # Subnet 3 - x:x::2:0/64
cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 3) # Subnet 4 - x:x::3:0/64
```

The next step is the parameter for the `aws_subnet` resource block, which is `ipv6_cidr_block`. Essentially the same as the `cidr_block` parameter, but for IPv6!

So for each of our public subnets, lets add this in. In our sample file `vpc_subnets.tf` (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc_subnets.tf), we have two public subnets, `a` and `b`. So lets make the change. Below is the example with the new parameters added:

```HCL
# Public Subnet A
resource "aws_subnet" "public_a" {
  vpc_id               = aws_vpc.sample_vpc.id
  ipv6_cidr_block      = cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 0)
  cidr_block           = cidrsubnet(aws_vpc.sample_vpc.cidr_block, 4, 10)
  availability_zone_id = data.aws_availability_zones.available.zone_ids[0]
  tags = {
    "Name" = "Public-Subnet-A"
  }
}

# Public Subnet B
resource "aws_subnet" "public_b" {
  vpc_id               = aws_vpc.sample_vpc.id
  ipv6_cidr_block      = cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 1)
  cidr_block           = cidrsubnet(aws_vpc.sample_vpc.cidr_block, 4, 11)
  availability_zone_id = data.aws_availability_zones.available.zone_ids[1]
  tags = {
    "Name" = "Public-Subnet-B"
  }
}
```

Once again, we run our `terraform plan` and we should get an output similar to this:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_subnet.public_a will be updated in-place
  ~ resource "aws_subnet" "public_a" {
        id                                             = "subnet-0123456789abcdefg"
      + ipv6_cidr_block                                = "xxxx:yyyy:zzzz:nnn1::/64"
        tags                                           = {
            "Name" = "Public-Subnet-A"
        }
        # (15 unchanged attributes hidden)
    }

  # aws_subnet.public_b will be updated in-place
  ~ resource "aws_subnet" "public_b" {
        id                                             = "subnet-gfedcba9876543210"
      + ipv6_cidr_block                                = "xxxx:yyyy:zzzz:nnn2::/64"
        tags                                           = {
            "Name" = "Public-Subnet-B"
        }
        # (15 unchanged attributes hidden)
    }

Plan: 0 to add, 2 to change, 0 to destroy.
```

The plan shows the addition of the new CIDR block to each subnet, noting the network number changing slightly to accommodate the size we requested. Once applied, we should see these details in the console:

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6onSubnet.png" caption="IPv6 CIDR on one of the two sample subnets" gallery="gallery" >}}

Perfect, now let's launch an EC2 instance and then try and connect to something using the IPv6 address!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6EC2NotConnected.png" caption="The EC2 instance can't connect to an IPv6 address" gallery="gallery" >}}

Well, almost there - we re missing the routing.

## Adding IPv6 Routing to the public subnets

Having a look at our existing route tables, we can see that AWS added in the route for the VPC IPv6 CIDR to target the local VPC in the route table, which means it can find resources in the VPC that have the IPv6 address that is within the CIDR. Great for local traffic, but we need to be able to access the outside world using IPv6!

⚠️ **Note:** While traffic can still route with IPv4, we need to enable routing for IPv6, otherwise any traffic inbound to the server won't be able to send traffic back. ⚠️

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6RouteTableMissing.png" caption="The sample Public-Route-Table doesn't have any IPv6 route to the internet" gallery="gallery" >}}

We can also see this in the sample code too, in the `vpc_public_routing.tf` file (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc_public_routing.tf)

```HCL
# Primary Sample Route Table (Public)
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.sample_vpc.id

  tags = {
    "Name" = "Public-Route-Table"
  }
}

# Route entry specifically to allow access to the Internet Gateway
resource "aws_route" "public2igw" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.sample_igw.id
}
```

Here we can see the `Public-Route-Table` resources, but it only shows traffic to the Internet Gateway (IGW) for IPv4 traffic. We will need to add a route to allow IPv6 traffic to hit the IGW.

We can do this by creating a new `aws_route` resource, but we need to use the `destination_ipv6_cidr_block` parameter instead.

For the destination though, with IPv4 you could use the block `0.0.0.0/0` to mean "all traffic". If we were to type the IPv6 version out in full, it would look like `0000:0000:0000:0000:0000:0000:0000:0000/0`. Bit of a pain! Thankfully we can apply the short hand rule mentioned at the start, remove all the 0's, shrink down, and you get quite simply `::/0`, which suddenly seems much shorter than the IPv4 version! With this we can add this as the destination IPv6 block.

```HCL
resource "aws_route" "public2igw_ipv6" {
  route_table_id              = aws_route_table.public_rt.id
  destination_ipv6_cidr_block = "::/0"
  gateway_id                  = aws_internet_gateway.sample_igw.id
}
```

Once again, lets run the `terraform plan` and we should get something like this:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_route.public2igw_ipv6 will be created
  + resource "aws_route" "public2igw_ipv6" {
      + destination_ipv6_cidr_block = "::/0"
      + gateway_id                  = "igw-0123456789abcdefg"
      + id                          = (known after apply)
      + instance_id                 = (known after apply)
      + instance_owner_id           = (known after apply)
      + network_interface_id        = (known after apply)
      + origin                      = (known after apply)
      + route_table_id              = "rtb-0123456789abcdefg"
      + state                       = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

And once applied, we should be able to see the new route in the routing table!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6RouteTableAdded.png" caption="::/0 has been added to the route table, pointing to the Internet Gateway (IGW)" gallery="gallery" >}}

Jumping back on our EC2 instance, and we get a connection!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6EC2Connected.png" caption="We are now able to connect using IPv6 to the outside world!" gallery="gallery" >}}

⚠️ **Attention:** Inbound traffic to an EC2 instance, for example, will still be protected by its Security Group. The rules in a security group are specific to the IP protocol, so an allow for an IPv4 inbound rule, will only allow that. Make sure that you add any additional IPv6 rules to the security group to permit inbound access to resources in the Public Subnet ⚠️

## Adding IPv6 outbound routing to the private subnets

We are getting to the last part of this post about enabling IPv6 on AWS, and we do need to cover the private subnets to complete the sample network configuration. This part will come with a few warnings, but if you are keeping in line with the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/), this will not be an issue at all!

Thinking back to what we mentioned before, AWS will use global unicast addresses for resources and services in AWS. Meaning, that you do not have a "private" CIDR for your private subnets. As you know, in IPv4 there is the [RFP1918](https://www.rfc-editor.org/rfc/rfc1918) that defines a number of IP blocks, specifically for "internal connectivity". These IP ranges are not routable on the public internet. This allowed the expansion of devices within a private network, without using up the public internet space. As IPv6 can assign every device on the planet, it makes it easier for us to ensure that the address assigned to a node is unique.

Next we have to look at the definition of what AWS considers a "public" subnet and a "private" subnet. A "public" subnet, is considered such, whereby the route table for the subnet points its outbound internet traffic directly to an Internet Gateway (IGW), and the IGW allows external traffic to route to the subnet. For a "private" subnet, there is no IGW for it to connect to, and you would use a service such as a NAT Gateway (NGW) to connect through to the internet, and as the NAT Gateway doesn't allow traffic inbound to the network, and the IP address of the node will be a non-internet rotatable IP address, traffic can't reach the service in the VPC.

This definition sill applies to Private IPv6 subnets, and this is why AWS created the [_IPv6 Egress-Only Internet Gateway_](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html). Much like the IPv4 Internet Gateway counterpart, the IPv6 Egress-Only Internet Gateway is a horizontally scaled, redundant, and highly available VPC component that allows outbound communication for IPv6 traffic to the internet. As this new IPv6 egress-only internet gateway only permits traffic outbound, and not inbound, this will make a subnet private, as traffic can not reach it.

As this is a VPC component, and not a service like the NAT Gateway, you technically only need to deploy one, like the Internet Gateway, and point outbound traffic to the egress-only internet gateway, and it will scale as needed.

With this in mind, lets start by adding in the Egress-only Internet Gateway. This is created by the terraform resource `aws_egress_only_internet_gateway` (https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/egress_only_internet_gateway). This is very similar to the normal Internet Gateway, and even the set up is the same.

Within the `vpc_private_routing.tf` file (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc_private_routing.tf) you will need to add the following resource:

```HCL
# IPv6 Egress-only Internet Gateway
resource "aws_egress_only_internet_gateway" "sample_ipv6_egress_igw" {
  vpc_id = aws_vpc.sample_vpc.id

  tags = {
    "Name" = "Sample-VPC-IPv6-Egress-Only-IGW"
  }
}
```

Once again, running `terraform plan` will output something similar to this:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_egress_only_internet_gateway.sample_ipv6_egress_igw will be created
  + resource "aws_egress_only_internet_gateway" "sample_ipv6_egress_igw" {
      + id       = (known after apply)
      + tags     = {
          + "Name" = "Sample-VPC-IPv6-Egress-Only-IGW"
        }
      + tags_all = {
          + "Environment" = "Sandbox"
          + "Name"        = "Sample-VPC-IPv6-Egress-Only-IGW"
          + "Source"      = "Terrform"
        }
      + vpc_id   = "vpc-0123456789abcdefg"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

At this point, I would like to point out - that all the changes we have made so far, have not increased the cost of running! Much like the Internet Gateway, the only cost you have is the outbound traffic. As the Egress-Only Internet Gateway is the same, there is no cost for running this in a VPC other than the outbound traffic you intend to push through it. Unlike the NAT Gateways, which will cost you about $36.50/month, and then for best practice, you would need one in each availability zone, which then adds up. It does make IPv6 only networking in AWS far cheaper than IPv4!

Apply the changes, and the new Egress-only Internet Gateway will be created.

## Adding IPv6 CIDRs to the private subnets

The next step, is that we need to add the CIDR blocks to the private subnets. This is pretty much identical to the public subnet addition, in fact, it is identical. This is due to there being no difference between public blocks and private blocks within the VPC IPv6 range. So for speed, we just repeat the same.

Head back to our sample file `vpc_subnets.tf` (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc_subnets.tf), and look for the two "private" subnets. Add in the `ipv6_cidr_block` for each of them, based off the next two blocks calculated earlier:

```HCL
# Private Subnet A
resource "aws_subnet" "private_a" {
  vpc_id               = aws_vpc.sample_vpc.id
  ipv6_cidr_block      = cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 2)
  cidr_block           = cidrsubnet(aws_vpc.sample_vpc.cidr_block, 4, 12)
  availability_zone_id = data.aws_availability_zones.available.zone_ids[0]
  tags = {
    "Name" = "Private-Subnet-A"
  }
}

# Private Subnet B
resource "aws_subnet" "private_b" {
  vpc_id               = aws_vpc.sample_vpc.id
  ipv6_cidr_block      = cidrsubnet(aws_vpc.sample_vpc.ipv6_cidr_block, 8, 3)
  cidr_block           = cidrsubnet(aws_vpc.sample_vpc.cidr_block, 4, 13)
  availability_zone_id = data.aws_availability_zones.available.zone_ids[1]
  tags = {
    "Name" = "Private-Subnet-B"
  }
}
```

Run the `terraform plan` and apply the changes to assign the blocks to the subnets.

## Adding IPv6 Routing to the private subnets

This is where the limitations with the IPv4 NAT Gateway makes the changes for a private subnet add additional operational overhead to the work needing to be completed. In our sample code, we created two route tables for the private subnets, one for each availability zone, so that traffic within the subnet routes through the NAT Gateway for that availability zone.

This means, rather than like the public subnet, where we need to add just a single route in terraform. We have to add two and attach them to both route tables.

If you were creating an IPv6 only VPC, then you could reduce the work and have a single route table that works for both availability zones! However, we will look at this at a later date.

This time, we need to head to the `vpc_private_routing.tf` file (https://github.com/mystcb/ipv6-on-aws/blob/main/01-sample-vpc/vpc_private_routing.tf) again, and we need to add in the two new routes.

If you look at the file, you will see that for the IPv4 NAT Gateway, there is a specific parameter that you use to tell terraform to issue the right API command to AWS to add the route, which you can see here:

```HCL
# Route for Subnet A to access the NAT Gateway
resource "aws_route" "private2natgwa" {
  route_table_id         = aws_route_table.private_rt_a.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.sample_natgw_subnet_a.id
}
```

The `nat_gateway_id` will be specifically used to pointing to the ID of the NAT Gateway within the availability zone. Much like in the public subnet you would use `gateway_id` for the Internet Gateway, you can use the `egress_only_gateway_id` for the IPv6 traffic.

Therefore, we will need to add 2 new blocks to the terraform file, to add in the route `::/0` to point to the Egress-only Internet Gateway.

⚠️ **Attention:** Remember, for IPv6 routes, you will need to use the `destination_ipv6_cidr_block` as part of the route table resource ⚠️

```HCL
# Route for Subnet A to access the Egress-Only Internet Gateway
resource "aws_route" "private2ipv6egressigwa" {
  route_table_id              = aws_route_table.private_rt_a.id
  destination_ipv6_cidr_block = "::/0"
  egress_only_gateway_id      = aws_egress_only_internet_gateway.sample_ipv6_egress_igw.id
}

# Route for Subnet B to access the Egress-Only Internet Gateway
resource "aws_route" "private2ipv6egressigwb" {
  route_table_id              = aws_route_table.private_rt_b.id
  destination_ipv6_cidr_block = "::/0"
  egress_only_gateway_id      = aws_egress_only_internet_gateway.sample_ipv6_egress_igw.id
}
```

For one final time, run the `terraform plan` and you should see an output similar to this:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_route.private2ipv6egressigwa will be created
  + resource "aws_route" "private2ipv6egressigwa" {
      + destination_ipv6_cidr_block = "::/0"
      + egress_only_gateway_id      = "eigw-0123456789abcdefg"
      + id                          = (known after apply)
      + instance_id                 = (known after apply)
      + instance_owner_id           = (known after apply)
      + network_interface_id        = (known after apply)
      + origin                      = (known after apply)
      + route_table_id              = "rtb-0123456789abcdef1"
      + state                       = (known after apply)
    }

  # aws_route.private2ipv6egressigwb will be created
  + resource "aws_route" "private2ipv6egressigwb" {
      + destination_ipv6_cidr_block = "::/0"
      + egress_only_gateway_id      = "eigw-0123456789abcdefg"
      + id                          = (known after apply)
      + instance_id                 = (known after apply)
      + instance_owner_id           = (known after apply)
      + network_interface_id        = (known after apply)
      + origin                      = (known after apply)
      + route_table_id              = "rtb-0123456789abcdef2"
      + state                       = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

Remember, that we are adding two routes, as we have two route tables to make the change to. Apply the changes, and you should see the new route in the route tables:

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6PrivateRoutingEgressOnly.png" caption="New route pointing to the new Egress-Only Internet Gateway VPC resource" gallery="gallery" >}}

So, jumping on an EC2 instance in the private subnet (you can see on the command line that the instance is in the `192.168.12.0/24` subnet), you can see we have complete outbound access on IPv6!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6EC2PrivateConnected.png" caption="Private Subnet EC2 instance with connectivity through the Egress-Only Internet Gateway" gallery="gallery" >}}

## The final outcome

So you finally did it, you have enabled IPv6 on your VPC. If all went to plan, you should have something that looks like this:

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/02/IPv6EnabledVPC.png" caption="Sample VPC with IPv6 enabled" gallery="gallery" >}}

To make things a little more simpler, you can also check out the `02-vpc-with-ipv6` folder in the sample code (https://github.com/mystcb/ipv6-on-aws/tree/main/02-vpc-with-ipv6), which will produce the same output
as the diagram, and if you followed the changes!

## Summary and round up

Thank you for getting this far! There is a lot here, but hopefully I have been able to show you how to add IPv6 to an AWS VPC using Terraform. However, this is only the beginning. Throughout my time working with IPv6, there will always be different issues to trip you up, and there are far more features than just a simple VPC!

In a future post I hope to show you the following:

- Adding IPv6 to an existing EC2 instance inside a VPC without rebuilding it. A little harder than I expected with Terraform, but I did find a way around it!
- IPv6 only VPCs! Much cheaper to run that a normal IPv4 VPC, but at what "cost", adoption of IPv6 is still quite low, but there are a number of design patterns that mean an IPv6 VPC might work better for some situations.
- Going into IPv6 in more detail. This is only the very basic information on IPv6, and a lot of what I have mentioned, does come with caveats! I will hope to explain these in more details later on
- Bits? Why does one number mean something else, and why on earth do we count in bits? - Hope you have learnt binary for this one!

Thank you again, and feel free to send me any questions, or help me with any corrections to this post as well! Hopefully, I will see you in a future post!

P.S. The final cost of me running this for the creation of the post, was mainly the NAT Gateways! Everything else was free! It only cost me $1.50 total! Always make sure you run `terraform destroy` at the end!

P.P.S. Happy Birthday to me!

Further Reading:

- [_IPv6 on AWS_](https://aws.amazon.com/vpc/ipv6/)
- [_IPv6 - Wikipedia_](https://en.wikipedia.org/wiki/IPv6)
- [_Sample Code for this Post_](https://github.com/mystcb/ipv6-on-aws)
