---
title: AWS DNS, Route53, CNAME records, and how it is resolved
author: Colin Barker
date: 2024-12-10T21:32:00.0000Z
description: During my time at a previous customer, we had an issue with a customer DNS records not quite resolving as expected. Lets have a look into the different DNS resolver points, and why this was an issue.
tags:
  - aws
  - networking
  - dns
  - cnames
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2024/12/brittany-colette-GFLMi4c8XMg-unsplash.jpg
---

> Header photo by [Brittany Colette](https://unsplash.com/@brittanycolette?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/persons-holding-book-GFLMi4c8XMg?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)

## What was the issue?

While I would love to take all of the credit for this, my old squad working with this customer had to deal with this issue for a very long time, till we got our head together and figured out what was going on! [Jake Morgan](https://www.linkedin.com/in/jakeelliotmorgan/) put most of this together into a more visual and documented sense for the customer so I am hoping to use what I remember from back then to put this together.

The issue we had, a customer using CNAMEs to point generic host names to key services within their network, were having major issues resolving the names once they had migrated to AWS. For this I will have to explain the scenario in a little more detail, the domain used in this example is one of my own - and something you should be able to test yourself with your own account if you so wish!

Ultimately during a migration, we needed to move a service from On-Premise to AWS, in doing so - it's IP would change, but we only had one top level record for running this service. Switching the IP was, a little harder than expected, so here we go into a bit more detail as to what the domain was and what it entailed.

### The domain

For this example, we are going to be using the `acmeltd.co.uk` domain. One of my personal domains that I use for random testing and development, bought when I had to use a domain for Active Directory, but over the years has become a little underused! For this to work correctly, the authoritative domain records can be found at:

- `ns1.faereal.net`
- `ns2.faereal.net`

Here the root domain sits, and where most of the "original" configuration will come from. Here we will have a top level entry of `service1.acmeltd.co.uk` to represent a service hosted somewhere in our environment.

### The delegation

What our customer originally had setup, was not quite best practice, but this is why we had come into migrate them into AWS! However, this can show how the issue can occur.

Here we have to "pretend" that we have a DNS server on premise, in this example we will be using `ns3.internal.faereal.net` - this entry doesn't exist, so it will always fail, but for our customer - this was pointing to a DNS server on-premise with a local non-internet routable IP.

The delegation we will use will be `region.prod.acmeltd.co.uk` - a regional production zone that will be initially hosted on-premise.

### The service

Here is where we can go back to our service above. Internally, the service can be referenced by the DNS record `service1.region.prod.acmeltd.co.uk` of which we can pretend that this an `A` record that points to `192.168.100.10`. This works fine on-premise when looking up. The next part of our example, the top level service will be a `CNAME` record, pointing to a record specifically hosted on our internal DNS server. `service1.acmeltd.co.uk` will be a CNAME record, pointing to `service1.region.prod.acmeltd.co.uk`. This means anyone looking up `service1.acmeltd.co.uk` will be pointed to the internal DNS server, where the record will be resolved to `192.168.100.5`.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/example-private-dns-zone.png" caption="The Route53 Private DNS zone used in this example" gallery="gallery" >}}

### The migration

Usually, the easiest option here would be to use a [Route53 Outbound Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html) however, in this instance - it doesn't work as expected. So for the moment, we can say that this is in place, but we can ignore it for the moment.

To try and get around the issue, we tried to setup a [Route53 Private Hosted Zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html) to match the zone that is currently held on premise. `region.prod.acmeltd.co.uk`, in an attempt to localise the zone inside the VPC, in the hopes that this would resolve the query issues. Within this, we included a specific `A` record that points to a local IP inside AWS `10.10.10.10` as an example.

### The expected DNS resolution pathway

So with all in place, we were expecting the following:

**Within AWS**

`service1.acmeltd.co.uk` -> `CNAME` -> `service1.region.prod.acmeltd.co.uk` -> Private Hosted Zone -> `10.10.10.10`

**Within On-Prem**

`service1.acmeltd.co.uk` -> `CNAME` -> `service1.region.prod.acmeltd.co.uk` -> On-Premise DNS server -> `192.168.100.10`

### The outcome

```
[ec2-user@ip-10-10-10-12 ~]$ host service1.acmeltd.co.uk
Host service1.acmeltd.co.uk not found: 2(SERVFAIL)
```

Well, that didn't work at all did it.

## Troubleshooting the DNS resolvers

This is where we got stuck originally, DNS wouldn't resolve, and we needed to get this working to ensure the migration worked. So we stepped through each resolver till we could see where the issue was.

### From an EC2 instance

For our testing, we are going to be using a simple EC2 instance, this way we can check along the way. So lets look at where it resolves it's DNS.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/DNSandCnamesBlogPost-EC2-And-VPC.jpg" caption="The EC2 instance we are testing with inside a VPC, with the associated Route53 Private DNS zone" gallery="gallery" >}}

All VPC's have their own DNS resolver build into it, specifically its on the second IP within each subnet, so if you had a subnet of `10.10.10.0/24` the DNS resolver would be at `10.10.10.2`. For more information [see the AWS documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-DSN-queries-to-vpc.html). Using the `dig` command we can see this in action, including the server it was looking at.

```
[ec2-user@ip-10-10-10-12 ~]$ dig CNAME service1.acmeltd.co.uk

; <<>> DiG 9.18.28 <<>> service1.acmeltd.co.uk
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24752
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;service1.acmeltd.co.uk.                IN      CNAME

;; ANSWER SECTION:
service1.acmeltd.co.uk. 60      IN      CNAME   service1.region.prod.acmeltd.co.uk.

;; Query time: 0 msec
;; SERVER: 10.10.10.2#53(10.10.10.2) (UDP)
;; WHEN: Mon Dec 09 13:30:59 UTC 2024
;; MSG SIZE  rcvd: 86
```

As you can see, the dig looking up the CNAME record did pull back the CNAME record, but it hasn't then continued the resolution onto getting the `A` record from anywhere. Very confusing, as the VPC resolver also has the private hosted zone `region.prod.acmeltd.co.uk` associated to it, so logically it should have picked it up. What you see, is why this isn't the case.

### AWS's External DNS resolver

From the VPC resolver, it is only logical that the lookup of `service1.acmeltd.co.uk` would then head out to the two authoritative public DNS servers. To be able to do this, AWS themselves will have a resolver to connect out to the public internet for you. This is why on a private subnet, with a VPC resolver enabled, it is possible to resolve public DNS records without any access to the internet.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/DNSandCnamesBlogPost-AuthoritiveDNS.jpg" caption="Example of where the AWS External Resolver and the Authoritative DNS resolver sit in relation to a VPC" gallery="gallery" >}}

As the AWS External Resolver isn't authoritative for the `acmeltd.co.uk` domain, it would pass the query onwards to its DNS servers, in this case, the `ns1.faereal.net` and `ns2.faereal.net` servers mentioned before.

### The Authoritative DNS Resolver

Now that the query has been received by the authoritative DNS resolver, it finally can get the `CNAME` record, and this is what we saw from the server - it responded with the `CNAME` of `service1.region.prod.acmeltd.co.uk`. Which is then sent back to the AWS External Resolver, which then tries to look up that domain, and we hit an issue.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/DNSandCnamesBlogPost-ResolverIssue.jpg" caption="Diagram showing the flow of what happens when the External Resolver tries to resolve the service1.acmeltd.co.uk hostname" gallery="gallery" >}}

The AWS External Resolver already knows that the `acmeltd.co.uk` has the `region.prod.acmeltd.co.uk` record which is an `NS` record pointing to the private internal DNS server `ns2.internal.faereal.net` - this being an internal IP, means it can't continue with the resolution, and will report back a `SERVFAIL`, and no record is resolved. The AWS External DNS resolver doesn't have access to the VPC private networks, so it wouldn't be able to resolve to the on-premise DNS servers. It's the "knowing" part that causes the issue, as it won't push the answer for the `CNAME` record back down the chain.

### As one picture

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/DNSandCnamesBlogPost-FullFlow.jpg" caption="The full end to end flow of the DNS lookup" gallery="gallery" >}}

As you can see, even with the private zone, the and even a specific Route53 outbound resolver in your VPC, this setup doesn't work. How did we resolve this.

## The resolution

Of everything we tried, the only one that worked for this customer, was using a completely separate domain. Let's see how this changes the setup.

### The new domain

For our example, we will be using a new domain `brkr.io`, but also creating a new root level record called `service2.acmeltd.co.uk` that we will `CNAME` - this is just so if you wanted to follow along and see this for yourself, the lookups will work for you!

With this new domain, we can get the original `CNAME` record answer to be pushed back into the AWS VPC Resolver instead, and it can then do the final step of the look up for us. So quickly, this is how we set this up:

- A new root level `CNAME` record has been setup `service2.acmeltd.co.uk` - In the real world, we changed the original `service1.acmeltd.co.uk` to point to the new domain record
- The `CNAME` pointed to `service2.region.prod.brkr.io`
- On-premise a new DNS zone was setup for `region.prod.brkr.io` to resolve services to local IP's (`192.168.100.10`) within their on-premise setup
- A new Route53 Private Hosted Zone called `region.prod.brkr.io` was created with a record for `service2.region.prod.brkr.io` to point to `10.10.10.10`

### The output

```
ec2-user@ip-10-10-10-12 ~]$ dig service2.acmeltd.co.uk

; <<>> DiG 9.18.28 <<>> service2.acmeltd.co.uk
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20464
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;service2.acmeltd.co.uk.                IN      A

;; ANSWER SECTION:
service2.acmeltd.co.uk. 54      IN      CNAME   service2.region.prod.brkr.io.
service2.region.prod.brkr.io. 300 IN    A       10.10.10.10

;; Query time: 139 msec
;; SERVER: 10.10.10.2#53(10.10.10.2) (UDP)
;; WHEN: Mon Dec 09 14:02:29 UTC 2024
;; MSG SIZE  rcvd: 109
```

It worked!

## Why did it work?

For this, we will need to update our original diagram to show the flow, but the main reason is - the `brkr.io` domain in our example, wasn't authoritative to the original DNS servers, so it needed to go back to "the start" and continue the resolution chain again. This allowed it to use the Route53 Private Hosted Zone for its lookup, but this would also work with a Route53 Outbound Resolver as well.

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/12/DNSandCnamesBlogPost-Updated Flow.jpg" caption="The final working flow, and it resolving to the correct address" gallery="gallery" >}}

## Summary

DNS can be very easy, it can also be a compete nightmare to work out where everything is! For us, it was this mysterious "AWS External Resolver" which, once we had put it on paper, made complete sense as to why it was the issue - however not knowing it was there was part of the problem. Always check your DNS resolution chains to see where and more specifically how a resolver is getting an answer.
