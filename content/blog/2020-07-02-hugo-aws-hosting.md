---
title: What's behind this blog?
author: Colin Barker
date: 2020-07-02T10:30:00.0000Z
description: Learnings from trying to host this blog and the Tokonatsu Festival website, using CloudFlare, AWS S3 Static Hosting and Hugo
tags:
  - aws
  - s3
  - hosting
  - tokonatsu
  - cloudflare
categories:
  - aws
images:
  - src: https://static.colinbarker.me.uk/img/blog/2020/07/diagram.png
    alt: Logical Layout of Blog Setup
---

Seems pretty simple, a [CloudFlare](https://www.cloudflare.com) front end, with
an [AWS S3 bucket](https://aws.amazon.com/s3/) to hold the content for a static
website generated using [Hugo](https://gohugo.io/), but is it?

For me to get to where I am with two websites running in this way, it was a long
journey and a lot of experience gained along the way. Here I will go into the
details on why we are here today, and over time explain the full journey over
a number of posts.

### What are the key details?

#### CloudFlare

Not that I have a problem with [AWS CloudFront](https://aws.amazon.com/cloudfront/),
the choice of CloudFlare was a legacy one. Both the [Tokonatsu Festival](https://www.tokonats.org.uk)
and my personal website have had many different backing technologies over the
years. From Drupal to Wordpress, and even at one point a custom built PHP CMS
system that I wrote from the ground up back in the days of DDR:UK. All of these
have one thing in common; Terrible speeds under load!

CloudFlare at the time offered a service which my hosting provider at the time
did not have, and mainly due to time, I have not had a chance to change this in
anyway. It works perfectly well, and it would be a shame to break that!

#### AWS S3 Websites

A staple for any static based website. Having a large amount of processing power
behind a simple website doesn't make sense any more. Looking back at the days
before "The Cloud", I used a desktop computer, shared hosting on Plesk service,
IBM rack-mounts in physical data centres in Central London and even an
[ex-nuclear bunker](https://www.thebunker.net/). These all do have a place in
the modern world however, as time has progressed the use cases for these hosted
systems becomes narrower. Cost becomes a primary issue, and for someone running
a very simple blog, or a front face to a festival, an [AWS S3 bucket](https://aws.amazon.com/s3/)
can do the job just as well at a fraction of the cost.

#### Hugo

Why did I choose Hugo, a static website generator written in [Go](https://golang.org/)?
Probably as simple as the CloudFlare decision! Someone mentioned it to me and I
stuck with it! Primarily it was a decision made during the transition from
Drupal 6 for the [Tokonatsu Festival](https://www.tokonats.org.uk) to it's own
static version of the site.

At the time, [Tokonatsu Festival](https://www.tokonats.org.uk) used Drupal as
most of the [Anime Conventions](https://animecons.co.uk/) in the UK used a
similar ticking system which had been written in Drupal 5, which I was one of
a number of events that attempted to upgrade it to work with Drupal 6. The
festival moved away from the normal way of conventions in the style of running
and getting closer to other UK festivals in style. Thus we moved to a ticketing
system called [Pretix](https://pretix.eu/about/en/), and all of the processing
for the website was pushed away, leaving a very heavy CMS system for a bunch of
mostly static pages running on [AWS ElasticBeanstalk](https://aws.amazon.com/elasticbeanstalk/).

The transition to Hugo took some time to get right, and to modify the theme from
the Drupal site was the hardest part, but we go there in the end. Which bought
me to my own site! Why re-invent the wheel when you have everything working!

### What's next?

#### Automate everything!

My next goal will be to complete the automation of this blog, as it stands I am
borrowing the automation used for the [Tokonatsu Festival](https://www.tokonats.org.uk)
site which uses a self-hosted version of [GitLab](https://about.gitlab.com/).

#### Describe in-depth the different parts

To prevent this from becoming a massive blog, I plan to describe in more detail
the different parts of this, so expect details on:

- AWS S3 Bucket Security, what I am doing, and what you **should** be doing!
- CloudFlare, how to set that up to work with AWS S3 static website hosting
- Automation, how I automated the Tokonatsu website's deployment into AWS S3

### I was being serious

When I said I hosted on a desktop PC, it really wasn't a joke! My first ever
view on "hosting" and how to serve things on the internet was installing
Windows 95 (yes, you have permission to pull me apart on this one!) on a desktop
computer, and giving it a publicly routed IP address, and it survived!

I do have to call out [ExNet](https://www.exnet.com/index.html) and [Damon](http://d.hd.org/)
for helping me at the start. Looking after my desktop box inside your ISP! That
however, will be a story for another day!

{{< fancybox2 path="https://static.colinbarker.me.uk/img/blog/2020/07/faereal-server.jpg" caption="The original Faereal Server" gallery="gallery" >}}
