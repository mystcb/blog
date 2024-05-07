---
title: Cloud Landing Zones on AWS
author: Colin Barker
date: 2024-02-25T10:30:00.0000Z
description: The battle ranging on over what a "Landing Zone" should look like seems to be a common occurrence across the globe, but what is a Landing Zone? Here I will share my opinions on the concept known as "The Landing Zone" and how it applies to AWS.
tags:
  - aws
  - landing zone
  - concepts
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2024/02/pascal-meier-UYiesSO4FiM-unsplash.jpg
---

> Header photo by [Pascal Meier](https://unsplash.com/@zhpix?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/airport-runway?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Does one size fit all?

This is a very good question, and my opinion and answer will always be no. I have always been weary of companies that offer a "full stack Landing Zone" which is ready "as is". Some companies probably do have a good amount of Inner Source material that can be used to build a Landing Zone up quickly, but not a lot of companies do. A fair few of the big name players have "their way or the highway" version of the Landing Zone to deploy.

It's like asking everyone on the planet being asked to wear a size 10 shoe (UK size of course!). For me it would be slightly too big, and one of my friends it would be a perfect fit, but for others, I am sure there will be a fair few jokes! However, that is the point of having multiple types of shoes, sizes, and uses, they are there so people can choose what works for them.

Then there is the header photo, chosen because for me airports are a pretty good example of what a Landing Zone is. An airport must have some very specific elements to it to be able to accept the plane for landing. One key one is a runway of some description. However, for larger international airports, you would need a level of security as well. Then again, when you go to a larger airport you will see shops, duty free, taxi lanes, car parks, you name it - but are they required? No, but it does make things a lot easier!

This type of modular approach is how we should look at Landing Zones, and this is where we move from the Landing Zone being a static design, and into a concept.

## Landing Zone - The Concept

Let's dive into the concept here, and for me the Landing Zone is key to any implementation on any cloud provider. It provides the foundation for using cloud services, and ensures that from the start the Landing Zone meets the pillars of the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) - Operationally Excellent, Security, Reliable, Performance Efficient, Cost Optimised, and Sustainable).

### Mandatory Elements

These elements of a Landing Zone are pretty much the core, and foundation, of what you intend to deploy on AWS. Without these elements, the Landing Zone will likely not follow the Well-Architected Framework, and probably not work at all! Don't let the list fool you either, it does seem pretty short, but sometimes its the smallest implementations that can cause the most hassle!

#### AWS Account Structure

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-account-structure.png" caption="Typical Large Scale AWS Account Structure Example" gallery="gallery" >}}

This has to be the first step in the creation of the Landing Zone - the Account Structure. Ensuring a safe, secure, and best practice driven structure, you will always have to consider multiple accounts with AWS. My personal opinion and highest recommendation is that even if you are a small customer, that only needs a single account, still have a minimum of 2.

In the world of AWS, the top level account is known as the "Management Account", and while it is just a "normal account" that resources can be spun up in, it is not the best practice. Keeping this "Management Account" clear for just managing AWS will always be [best practice](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html). Within this account, you should set up your [AWS Organisation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html), and your Identity Management. From here, you have the option to delegate responsibility for these items to member accounts within your organisation. With the top of the tree in place, we can continue.

#### Authentication

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-iic-with-entraid.png" caption="Example of using an external IdP (EntraID) to log into AWS" gallery="gallery" >}}

With your account structure defined, you will need a way to be able to authenticate to AWS and the member accounts. This is where a secure and centralised Identity Provider (IdP) comes into play. There are many examples of ways this can be done, and with the example above, we are using [AWS IAM Identity Centre (originally called AWS SSO)](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html). One of the major benfits of using AWS IAM Identity Centre (IIC) to authenticate, is that you can also use both external based IdP, such as [Okta and Entra ID](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-change.html), but also it has its own database that can be used to [manage identities](https://docs.aws.amazon.com/singlesignon/latest/userguide/identities.html) on as well.

Now you should be able to give access to your AWS platform in a [centralised identity provider](https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_identities_identity_provider.html) as per the Well-Architected Best Practices.

#### Security Configuration

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-confirmance-packs.png" caption="Example of a Conformance Pack Template used to audit your platform" gallery="gallery" >}}

I would normally have this front and centre, and at the top of the list of mandatory elements as security is NEVER an option. However, in this case you do actually need something to apply the security to! Therefore, the last mandatory element of the Landing Zone Concept, is Security, and your Security Configuration.

It shouldn't come as a surprise to anyone that no matter what you are doing online, security must be your number one priority. The issues with data breaches is well known, and can affect millions, not just you. Within AWS there are several tools which can help "assess, audit, and evaluate" your estate, such as [AWS Config](https://aws.amazon.com/config/), as seen in the example above. Without becoming just a very long list of [AWS Security Products](https://aws.amazon.com/products/security/) which AWS do very well at listing, I want to keep this close to the concept for this blog.

> ⚠️ Remember: This isn't just about security of access, this is about security of the whole platform, its usage, availability, and logging of all the actions.

There are many tools that can be used to ensure that the security of your platform is as good as it can be, compliant as it needs to be, and safe as it should be, and making sure you are alerted to these and take action quickly all play into the concept of security.

{{< quote author="Werner Vogels" source="allthingsdistributed.com" url="https://www.allthingsdistributed.com/2016/03/10-lessons-from-10-years-of-aws.html">}}
everything will eventually fail over time
{{< /quote >}}

Took me a while to find that quote! I have seen it written as "Everything fails, all the time", which is still a fantastic quote - but I couldn't see where Werner had actually said that directly! (Personally I thought it was at a Re:Invent Video, but all my copies show others pasting the text on top of the screen with him on it!). Either way, I found a 2016 article that had him say something very close! However, think about it - while that applies to a lot of things, its quite clear, EVERYTHING will eventually fail, and you can say the same for security too. The Cat and Mouse game of keeping up with bad actors is never going to stop. At some point your security will fail, IF you do not maintain it, and keep on top of it, and update it.

Once again, Security is mandatory. Not just for a Landing Zone, but pretty much generally!

### Optional Elements

Only 3 mandatory elements? Doesn't seem right... but for me, that is really it. The Landing Zone is whatever you need it to be to work with your product, application, usage, the list goes on. Some of you may be thinking why something like "Networking" isn't on the list. In a way, the key bits you need to worry about if you deploy a network is, (the security), but do you actually need it?

#### Networking and VPCs

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-serverless-non-network-example.png" caption="Serverless Application, using Lambda and DynamoDB with no network" gallery="gallery" >}}

So, here is an example - say you have a Serverless Application, you are using [AWS API Gateway](https://aws.amazon.com/api-gateway/) with [AWS Lambda](https://aws.amazon.com/lambda/) then using an [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) backend. Simple enough, handy if you want to build an application that can return something specific for you, or log something, up to you, however there is no [VPC network](https://aws.amazon.com/vpc/) involved. All of this is happening using AWS's own Managed Networking, and API calls. The API Gateway Endpoint is available to you, and your application runs.

You should always remember the rule mentioned above though - Security is front and centre at all times. Even without any specific networking deployed, you are using a network of some sort, and you must continue to secure it. In this case, the network deployment is optional, but you still have to consider the risk when working with services in this way.

Still, you could add more networking, if you need it!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-cross-account-networking.png" caption="Go complex with your networking!" gallery="gallery" >}}

#### AWS Control Tower

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-control-tower.png" caption="A completed AWS Control Tower configuration" gallery="gallery" >}}

Probably my most controversial optional on the list, and this comes back to being a concept as well. AWS Control Tower is a tool that can be used to deploy AWS's own Opinionated Landing Zone, with that it comes with some very specific set of security configurations, authentication options, and account design. All the key required elements for your platform, by why state this is optional.

This goes back to the start of this post, where a size 10 shoe on someone that it doesn't fit on, wouldn't work. Using the right tool for the right scenario will be your path to success. Using AWS Control Tower therefore is optional, if the service works for you, then use it. For many users of Control Tower, it will work as they expect it to, and cover a lot of the main areas and requirements for your platform. However, so long as your platform contains the key required elements to make this work, then you still have a valid landing zone.

## Your AWS Platform - As The Concept

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2024/02/aws-full-landing-zone.png" caption="The Landing Zone with services examples" gallery="gallery" >}}

Going back to the start, the original AWS Account design that was shown, is a valid Landing Zone concept being deployed to any customer. In the example above, we have started to fill out keys services that will be used across the platform.

- _AWS Account Structure_ - Multiple accounts used for multiple types of services
- _Authentication_ - Using AWS IAM Identity Centre to act as a single point of entry into the platform
- _Security_ - Using AWS Security Hub, A Logging account separated to ensure security across the platform

The additional optional elements are also present

- _AWS Control Tower_ - A central tenant of building a save, secure, and Well-Architected Landing Zone
- _Networking_ - Seen with a separate Transit account.

But note, that your Landing Zone will still work, without those optional elements!

## Conclusion

The example shown here, might not work for you, it might be too complex, or it could be not as segmented as you require for compliance reasons, or an internal security policy. However, you can still use the concept to build your own AWS Platform. Concepts build the framework for your deployment. Frameworks allow you to use your skills, knowledge, and expertise to decide the right tool for the right job, and if not someone should be able to support you in that decision. However, the tool of choice, for it to be any good, must follow the concepts in this blog.
