---
title: AWS re:Invent 2020 - My thoughts
author: Colin Barker
date: 2020-12-14T10:12:50.0000Z
description: My thoughts behind the AWS re:Invent 2020 announcements
tags:
  - aws
  - reinvent
  - 2020
categories:
  - aws
images:
  - src: https://static.colinbarker.me.uk/img/blog/2020/12/reinvent-logo.png
    alt: AWS re:Invent Logo
---

Having been extremely lucky that [DevOpsGroup](https://www.devopsgroup.com) do
allow me the flexibility in my work to be able to attend previous [re:Invent](https://reinvent.awsevents.com/)
conferences however, this year was very different. COVID-19 and the worldwide
pandemic changed the way we work, interact, and how conferences should run.
[re:Invent](https://reinvent.awsevents.com/) was no different.

This year, AWS put on a **2 week** online conference, and still were able to
include all the bells and whistles that make [re:Invent](https://reinvent.awsevents.com/)
special!

For now, here are are my thoughts on some of the announcements made during the
event.

Sorry for the bad quality screenshots, unlike a real event where I would
remember my camera, I spent most of it trying to remember how to take
screenshots! (And somehow made them low quality!)

### Aurora Serverless v2

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2020/12/serverless-v2.png" caption="Screenshot of the Aurora Serverless v2 bullet points" gallery="gallery" >}}

Quite a significant change to the original [Aurora Serverless (now v1)](https://aws.amazon.com/rds/aurora/serverless/)
service, introducing additional features which opens the usage of this database
engine to even more creative ideas and solutions that might not have been
available to Solution Architects before.

Read Replicas was a sorely missed feature from the v1 service, which did make
the choice harder to sell to customers looking for a specific type of disaster
recovery solution, or being able to split their database traffic load in a very
specific way.

This is a preview at the moment, but will be keeping a close eye on this in the
near future.

- [AWS Blog Post](https://aws.amazon.com/about-aws/whats-new/2020/12/introducing-the-next-version-of-amazon-aurora-serverless-in-preview/)
- [Sign up for Early Access](https://pages.awscloud.com/AmazonAuroraServerlessv2Preview.html)
- [Aurora Serverless v2 Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-2.html)

### Bablefish for Aurora PostgreSQL

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2020/12/bablefish.png" caption="Screenshot of the AWS Bablefish for Aurora PostgreSQL announcement" gallery="gallery" >}}

Announced as an open source (Apache-2.0) project, [Bablefish for Aurora PostgreSQL](https://aws.amazon.com/rds/aurora/babelfish/)
is a welcome addition to any DBA, Engineer, and Solutions Architect's bag of
tricks! As with many projects, the challenge that always trips up those planning
on a migration into the cloud is that your build up tech debt of old database
technologies and setup does not quite fit nicely into your plan. This opens a
door which really needed to be open.

This specific version allows the migration easier over to using [Amazon Aurora](https://aws.amazon.com/rds/aurora/)
from Microsoft's SQL Server by "translating" the commands from applications
destined for SQL Server over to Amazon Aurora with very little code changes. A
nod to [The Hitchhiker's Guide to the Galaxy](https://en.wikipedia.org/wiki/The_Hitchhiker%27s_Guide_to_the_Galaxy)
by Douglas Adams, where so many other translation tools also get their name!

While it does understand a large number of the standard use cases for SQL Server,
care would need to be taken to ensure that you are able to translate everything.
Or at least, a way to migrate some of the more Microsoft proprietary elements of
SQL Server.

Once again, in preview, and will be keeping an eye out for more. Hopefully a
MySQL compatible one for assisting with the move into Aurora?

- [Bablefish for Aurora PostgreSQL Page](https://aws.amazon.com/rds/aurora/babelfish/)
- [Sign up for Early Access](https://pages.awscloud.com/Babelfish-for-Aurora-PostgreSQL.html)
- [Open source home page](https://babelfish-for-postgresql.github.io/babelfish-for-postgresql/)

### AWS CloudShell

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2020/12/aws-cloud-shell.png" caption="Screenshot of AWS CloudShell running" gallery="gallery" >}}

Another welcome addition to the AWS Console, [AWS CloudShell](https://aws.amazon.com/cloudshell/)
is just the missing item that allows operators of AWS to quickly get to a CLI
without worrying about what they need to have installed. Other cloud providers
had already included a function like this, and the addition of the CloudShell to
AWS was great to see.

Having a bit of a head start than the people who jumped in when it was
announced, I was able to quickly get a console up and running and have a look
around.

Like with other CloudShells, persistent storage of 1GB is useful to ensure you
have any non-secure settings readily available, is very handy and does save a
few headaches when logging back in for the first time. Pre-installed with the
[AWS CLI](https://aws.amazon.com/cli/), and running on an Amazon Linux 2 distro,
means that a number of other popular tools are available.

Pricing, **free**. Can't really argue with that!

If you can get on, I am sure you will understand that having this tool in your
back pocket for any event, will make your life a lot easier!

- [AWS CloudShell Page](https://aws.amazon.com/cloudshell/)
- [CloudShell Features](https://aws.amazon.com/cloudshell/features/)
- [CloudShell User Guide](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)

### Conclusion

As always, there was a lot to take in, and I only barely scratched the surface.
I'd recommend you look at some of AWS's Blogs to see all of the other
announcements, try and see if there is anything that might change the way you
think about solutions on AWS Cloud.

- [AWS re:Invent 2020 - Top Announcements](https://aws.amazon.com/blogs/aws/aws-reinvent-announcements-2020/)
- [Andy Jassy's Keynote](https://aws.amazon.com/blogs/aws/reinvent-2020-liveblog-andy-jassy-keynote/)
- [Verner Vogel's Keynote](https://aws.amazon.com/blogs/aws/reinvent-2020-liveblog-werner-vogels-keynote/)
