---
title: Five common mistakes that skew cloud migration TCO estimates
author: Colin Barker
date: 2023-04-19T12:50:19.0000Z
description: Large scale migrations are dependent on an accurate estimate of the total-cost-of-ownership (TCO) calculation that stacks up in your favour. However, this is largely more than a stab in the dark and can cause a migration to fail, or the costs to be out of control.
tags:
  - aws
  - migration
  - tco
  - estimation
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2023/04/christopher-bill-rrTRZdCu7No-unsplash.jpg
---

Header photo by [Christopher Bill](https://unsplash.com/@umbra_media?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/5-pieces-of-banknotes-on-yellow-and-white-textile-rrTRZdCu7No?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)

## What can be done?

Either scenario can result in decision makers and business managers getting the wrong impression of cloud computing, and potentially missing out on major business benefits. To avoid falling into this trap, a more considered approach to TCO calculation is needed. A detailed look at likely repercussions of migration can reveal both positive and negative outcomes, which enables more informed decisions.

As the AWS Practice Lead at Expert Thinking, I support and guide clients at every step of their cloud migration journey. Over the years, I’ve found that five key factors are often overlooked during an organisation’s TCO calculation phase. When they’re acknowledged, the result is a far more sophisticated estimation that goes beyond immediate transactional costs to build a longer-term picture.

## The five factors that hinder TCO calculations

### Like-for-like comparisons are rarely accurate

At a basic level, cloud migration TCO estimates look at the cost of hosting an instance in the original environment versus hosting the same instance in the cloud. AWS provides a great tool for this initial comparison with its Migration Evaluator.

However, things are never quite as simple as a like-for-like comparison might suggest. It should be treated as the first step, giving an initial indication of TCO ahead of further analysis and consideration of the wider context.

For instance, when migrating from a datacentre to the cloud, the depreciation of equipment value should be factored in. If the hardware CAPEX was expected to span five years, and cloud migration happens after two, that cost will remain on the balance sheet for another three years. What’s more, the cloud costs will include an allocation for management overhead and network bandwidth in addition to hosting.

On the face of it, these factors could make costs following cloud migration seem disproportionately high. But the story doesn’t end there.

### Shared responsibility is often misunderstood

A central aspect of AWS’ cloud offering is the shared responsibility model. It means AWS handles the operation, management and control of components from the operating system and virtualisation layer as well as the physical security of facilities.

So, while at first glance it may look as though monthly costs are higher with AWS, this doesn’t account for the reduced operational burden that shared responsibility brings. Relieving people from the ongoing management of underlying hardware unlocks new potential for staff productivity and business agility. Both of these elements play a critical role in deriving value from cloud computing, as per the AWS Cloud Value Framework.

For instance, time previously spent on day-to-day security and compliance matters can be diverted to application development or operational enhancements. This is a golden opportunity to reskill or upskill staff, boosting team morale. It ultimately leads to cumulative benefits as people make improvements and pass new learnings on to colleagues.

### The impact of past incidents and outages is forgotten

Like-for-like comparisons also ignore the fact that historic performance and reliability issues can be mitigated in the new environment through improved operational resilience.

Based on conversations we’ve had with hundreds of medium and large firms, it’s not uncommon for costs associated with a single outage to exceed £100k. When teams are too busy to properly identify or resolve underlying issues, they simply patch the platform up and the problems continue. It’s important to factor costs associated with platform instability into the TCO equation since many issues can be alleviated in the cloud environment.

The benefits of cloud-based business continuity can be harnessed from the outset of the migration process. For instance, AWS’ CloudEndure Disaster Recovery is an excellent tool that can be implemented across physical, virtual and cloud servers alike. We regularly use it on a short-term basis to protect customers’ existing environments during transformative migrations. It means that while time and energy is focused on building the new environment, the risk of suffering an outage on the old one is greatly reduced. CloudEndure can also be used as a cloud migration tool in situations where an automated lift-and-shift is appropriate.

### Software licences get overlooked

As with hardware, software licences need to be included in TCO estimates as not all software solutions are cloud friendly. Older licenses may be held on physical CPUs or per cores, and others are simply not portable to the cloud. AWS offers ‘Bring Your Own Licences’ (BYOL) options for Windows Server Licences via EC2 instances. However, there may be additional licences that cannot be used in the new environment, potentially resulting in cost duplication.

Taking steps to understand the impact of cloud migration on software licence costs at an early stage is critical. Conducting an AWS Migration Readiness Assessment is an effective way to achieve this. The assessment identifies systems that are reliant on software and triggers questions related to licence Ts&Cs. From here, decision makers can make an informed judgement surrounding total costs of hardware, software and licences in the original environment compared to an AWS PAYG model.

### Existing contractual obligations go unnoticed

Multiyear contracts for services such as rack rental are another factor that’s often forgotten in like-for-like comparison. Decommissioning racks before the end of a contract can result in penalty charges that should be included in the TCO calculation. It’s useful to prepare a burndown chart of costs for existing contracted services to provide clarity on the matter.

For organisations that prefer to work with a long-term contract fee structure, AWS can offer multi-year deals that may offset some of the costs associated with early termination penalties. Options include Amazon EC2 Reserved Instances and Savings Plans, both of which are up to 72% cheaper than on-demand instance pricing.
Take a broad perspective

## In Summary

TCO calculations need to look beyond the initial transaction. They should consider the bigger picture of costs involved with migrating to the cloud versus remaining in the current environment. Even then, the true value of cloud computing extends beyond potential cost savings. It unlocks new ways of working that boost staff productivity, operational resilience and business agility. The combined benefits of these improvements can have a transformative impact on business culture and performance. Cloud migration is an opportunity to reposition any business as dynamic, customer-focused and value-driven, rather than a slow-moving, cost-led entity.
