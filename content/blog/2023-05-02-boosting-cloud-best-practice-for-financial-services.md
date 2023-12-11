---
title: Boosting cloud best practice for financial services organisations
author: Colin Barker
date: 2023-05-02T18:11:45.0000Z
description: Data security is a key issue for financial services leaders as they consider strategies to accelerate or escalate cloud adoption. We find they share a hesitancy to store and manage data using a relatively new, evolving technology. Risk-aversion is perfectly natural in heavily regulated industries, but this blog considers how cloud best practice can alleviate concerns.
tags:
  - aws
  - migration
  - tco
  - estimation
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2023/05/alicja-ziajowska-AOjmfr3ofSY-unsplash.jpg
---

Header photo by [Alicja Ziajowska](https://unsplash.com/@alicja_photos?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/a-building-with-columns-and-a-flag-AOjmfr3ofSY?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)

## Cloud security fears

According to a study by cloud security specialist Barracuda, many IT security professionals are reluctant to host highly sensitive data in the cloud. This is especially true when it comes to customer information (53%) and internal financial data (55%). Respondents also said their cloud security efforts are hampered by a cybersecurity skill shortage (47%) and lack of visibility (42%). More than half (56%) are not confident that their cloud set-up is compliant.

It’s likely that these concerns are felt even more acutely in financial services organisations. Yet the opportunities and benefits of cloud adoption are too great to be ignored.

To help address this dichotomy, AWS published a Financial Services Industry Lens for its Well-Architected Framework. The document shows how to go beyond general best practice to satisfy stringent financial services industry demands. It focuses on “how to design, deploy, and architect financial services industry workloads that promote the resiliency, security, and operational performance in line with risk and control objectives…including those to meet the regulatory and compliance requirements of supervisory authorities.”

It’s critical that financial services workloads hosted on AWS are aligned with the document’s design principles.

## Well-Architected financial services

IT security risks come from within the organisation as well as outside it. Employee error or negligence can pose a significant threat, not to mention the damage that can be inflicted by malicious insiders. Whether data is stored in the cloud or on-premise, technical solutions must go hand-in-hand with operational measures.

For financial services organisations, least-privileged access or a ‘zero trust’ philosophy is a good starting point. AWS also advocates four principles to underpin the design of cloud-based architectures for financial services workloads:

- Documented operational planning
- Automated infrastructure and application deployment
- Security by design
- Automated governance

Automating infrastructure, application deployment and governance is perhaps daunting for organisations that are new to the cloud. However, it enables security to advance to a higher level than can ever be achieved on-premise. By minimising human involvement, it significantly reduces risk of error and improves consistency. It also allows quicker execution and scaling of security, compliance and governance activities.

Financial services organisations undergoing largescale migration would be well advised to refactor workloads for the new environment. This provides a valuable opportunity to introduce automation alongside security by design approaches. The additional upfront investment will go a long way towards addressing security concerns in the cloud.

As well as outlining general principles for the good design of financial services workloads, AWS illustrates six common scenarios that influence design and architecture. These include financial data, regulatory reporting, AI and machine learning, grid computing, open banking and user engagement. The list is not meant to be exhaustive, but an additional scenario that we regularly encounter is that related to network connectivity.

Read on to find out how we handled a financial data scenario for a customer that specialises in employee pay processes.

## Financial data in cloud-based workloads

According to AWS guidance, any financial data architecture should exhibit three common characteristics:

- Strict requirements around user entitlements and data redistribution.
- Low latency requirements that change depending on how the market data is used (for example, trade decision vs. post trade analytics) and which can vary from seconds to sub-millisecond.
- Reliable network connectivity for market data providers and exchanges

But how are these upheld during mandatory audits, especially when it comes to user entitlement and data redistribution? This was the challenge facing one of our customers as it prepared for largescale cloud migration. An internal firewall appliance solution needed to meet the auditing requirements of regulatory bodies without compromising security standards.

Our solution involved the CloudGuard platform from cybersecurity specialist Check Point, enabling traffic to be scanned securely in a way that met regulatory stipulations. We also suggested modernising the security set-up to make the important transition from a ‘pet’ to a ‘cattle’ mindset. This paved the way for a more automated approach firmly aligned with security by design principles as well as allowing the customer to build out scalable groups behind the scenes. Our approach is rooted in the assumption that failure is inevitable, looking to minimise the damage that occurs when it happens. In this way, it supports both the ‘reliability’ and ‘security’ pillars of Well-Architected.

## A high level overview

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2023/05/networking-setup.png" caption="Diagram of an example FSI's Customer Network" gallery="gallery" >}}

As the below diagram shows, we used AWS Gateway Load Balancers (GWLB) on the third layer of the networking Open Systems Interconnection (OSI) model, dynamically routing traffic to firewall appliances. As with any Application Load Balancer (Layer 7) or the Network Load Balancer (Layer 4), GWLB can step down to the network layer, using the GENEVE protocol, removing the need for ‘static’ or ‘pet’ instances to run the firewall appliance.
financial services networking set up

GWLB understands and can control the physical path that data takes without affecting the Transport Layer (4) which primarily deals with transmission protocols. This enables the use of autoscaling to dynamically scale firewall appliances in response to changes in traffic requirements. It’s similar to how an Application Load Balancer understands and translates application traffic, forwarding it to the correct endpoint. And it’s cheaper than running the appliances all the time.

As with all AWS’ Elastic Load Balancers, you can configure AWS Private Link to set up endpoint services and endpoints within subnets. We’ll come back to this later.

The above design also deploys the AWS Transit Gateway (TGW) service. This ‘level up’ on the original virtual private cloud (VPC) peering option allows you to configure deeper routing of traffic between VPCs. It’s highly available, secure, and ensures traffic never needs to leave the security of your AWS network. Exposure from the public internet is reduced as there are no internet-facing endpoints. And it’s also possible to include completed routing that pushes traffic to a centralised VPC, in this case ‘Egress’. Traffic bound for the internet is routed through the TGW service, and then through the Check Point appliance. This in turn uses NAT Gateways to talk to the internet.

## Industry-specific requirements

Using these services together, you can configure Internet Gateways assigned to VPCs to allow modification of routes for external traffic. It’s also possible to associate a standard VPC route table to the edge using an ‘edge association’. By pointing the routes for any external IP to the previously created TGW VPC endpoints, this traffic can be:

- encapsulated and sent through the Check Point appliances,
- filtered, scanned, checked, and firewalled,
- returned to the service that exists in the Public Subnet,
- sent to the final destination.

In our diagram, the Application Load Balancer in the Public Subnet balances traffic to EC2 instances in an autoscaling group.

The final service is the AWS Resource Access Manager (RAM), which is part of AWS Organizations. Centralised network administration is a key compliance requirement for several financial services certifications or programmes. However, configuring a VPC for access from the internet requires an Internet Gateway, which poses a security risk. Deploying the VPC and Internet Gateway into a central network account provides full control and visibility for relevant people. However, building resources in this VPC would normally require additional work to give appropriate levels of access. AWS RAM solves this by sharing resources with other accounts.

In the above example, two of the four subnets required for the set-up to work have been shared. So, the account with the shared subnets will be able to build resources and have full control of the process. However, it will not be able to bypass routing options set in place by the networking team, specifically the Gateway Load Balancer endpoint. This reduces risk and assures security/networking teams and auditors that nobody working in the accounts can bypass Check Point appliances.

## Enhanced security outcomes

The above solution ensures all traffic, specifically confidential information, is internally routed and doesn’t leave the secure network. Any traffic ingress and egress passes through the Check Point appliances, securing the data within the AWS network.

Network engineers, security teams and compliance auditors are given a centralised view and greater visibility of the deployed AWS network. The set-up permits any specific custom data monitoring elements using the Check Point appliances. With additional custom routing on TGW, you can also force traffic between VPCs to pass through the Check Point appliances. This adds a further level of network segregation, which can reduce the risk of data exfiltration between accounts and environments.

An additional benefit is that the Check Point appliance service works like any on-premises version. This reduces the need to re-skill teams to use a different technology stack and facilitates integration with existing management services. It also means the same solution can work with other networking appliances that support the GENEVE protocol, extending the range of services that align with the organisation’s existing skillsets.
