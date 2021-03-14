---
title: Wishtrees and Serverless on AWS
author: Colin Barker
date: 2021-03-14T20:12:50.0000Z
description: Getting the Tokonatsu Virtual Matsuri Wishtree working on AWS
tags:
  - aws
  - tokonatsu
  - 2021
categories:
  - tokonatsu
  - aws
images:
  - src: https://static.colinbarker.me.uk/img/blog/2021/03/wishtree.jpg
    alt: The Tokonatsu Wish Tree
---

### What is it?

During the COVID-19 Pandemic, [Tokonatsu Festival](https://www.tokonatsu.org.uk)
needed to run a virtual online only event. One of the many ideas included the
creation of a [Virtual Portal](https://www.tokonatsu.org.uk/2020/) with an
online created [Matsuri](https://en.wikipedia.org/wiki/Japanese_festivals). This
included a virtual version of our Japanese Virtual [Wish Tree](https://en.wikipedia.org/wiki/Wish_tree).

{{< fancybox path="https://static.colinbarker.me.uk/img/blog/2021/03/" file="lennart-jonsson-0nc0_v-PRUg-unsplash.jpg" caption="Photo by Lennart Jönsson (@lenjons) on Unsplash" gallery="gallery" >}}

[Tanabata](https://en.wikipedia.org/wiki/Tanabata) which is also known as the
Star Festival, celebrates the meeting of the deities Orihime and Hikoboshi. A
popular custom of this festival involves writing wishes on strips of paper. In
modern times, these small bits of paper are hung on bamboo, or with Tokonatsu,
our [Wish Tree](https://en.wikipedia.org/wiki/Wish_tree) shrine.

This is where our [Serverless Framework](https://www.serverless.com/) solution
came from.

### Background

The original code for the [Tokonatsu Wish Tree](https://www.tokonatsu.org.uk/2020/wishtree/)
was written by the Co-Vice Chair, Adam Hay. This used:

- [NodeJS](https://nodejs.org/en/)
- [AWS API Gateway](https://aws.amazon.com/api-gateway/)
- [AWS CloudFront](https://aws.amazon.com/cloudfront/)
- [AWS Lambda](https://aws.amazon.com/lambda/)
- [AWS DynamoDB](https://aws.amazon.com/dynamodb/)

The code for which is available on the [Tokonatsu GitHub](https://github.com/TokonatsuFestival)
under the [WishTreeApplication](https://github.com/TokonatsuFestival/WishTreeApplication).

### Serverless Framework

- https://www.serverless.com/

An open-source framework that helps to easily build serverless applications on
services like AWS. You use a combination of [YAML](https://en.wikipedia.org/wiki/YAML)
and a [CLI](https://en.wikipedia.org/wiki/Command-line_interface) to control,
manage, maintain, and operate your application.

#### Serverless Configuration File

This controlled from the [serverless.yaml](https://github.com/TokonatsuFestival/WishTreeApplication/blob/main/api/serverless.yml)
file within the `api` folder, and here is a quick breakdown of the file

```yaml
service: toko-wishtree-api # Name of the serverless service
frameworkVersion: "2" # Framework version to use

provider: # Provider Block
  name: aws # Which cloud provider (AWS)
  runtime: nodejs12.x # Which Runtime (NodeJS12)
  lambdaHashingVersion: 20201221 # Used to remove depreciated options
  stage: ${opt:stage, 'dev'} # API Gateway Stage (defaults to dev)
  region: ${opt:region, 'eu-west-2'} # AWS Region (defaults to London)
  apiGateway: # Specific API Gateway config
    shouldStartNameWithService: true # Ensure the API GW service starts with this
# --snipped--
```

Here we setup the top level information for the serverless package. Setting up
values including the service name, which is useful for identifying your
deployment in AWS, and the framework version.

The `provider` section is where we start to define how the Serverless Framework
will interact with the destination, and which specific codebase you are running.
In this instance, we are using AWS with the NodeJS version of 12.x.

#### IAM Permissions

To ensure access to the [DynamoDB](https://aws.amazon.com/dynamodb/) table,
we can define permissions in a new role which can be assigned to the [Lambda](https://aws.amazon.com/lambda/)
functions. This will mean no specific IAM keys or credentials will be needed
within the code.]

```yaml
iam:
  role:
    statements:
      - Effect: "Allow"
        Action:
          - "dynamodb:GetItem"
          - "dynamodb:Scan"
          - "dynamodb:PutItem"
          - "dynamodb:UpdateItem"
        Resource:
          - arn:aws:dynamodb:*:*:table/${file(./config.${opt:stage, self:provider.stage, 'dev'}.json):WISH_TREE_TAGS_TABLE}
```

#### The Functions

Each API endpoint we need to define will need to have a function to go along
with it. In this case, we can see the `getTagsForWishTree` function, that is
used by the Javascript on the static site to collect a list of the current tags
that it has stored in the [DynamoDB](https://aws.amazon.com/dynamodb/) table.

```yaml
functions:
  getTagsForWishTree:
    name: wish-tree-${opt:stage, self:provider.stage, 'dev'}-get-tags
    handler: getTagsForWishTree.handler
    description: Function to get the Wishtree Tags from the DB
    environment:
      ENV: ${opt:stage, self:provider.stage, 'dev'}
      WISH_TREE_TAGS_TABLE: ${file(./config.${opt:stage, self:provider.stage, 'dev'}.json):WISH_TREE_TAGS_TABLE}
    events:
      - http:
          path: getTagsForWishTree
          method: get
          cors: true
```

This will generate two elements. The [Lambda](https://aws.amazon.com/lambda/)
function using the NodeJS code with the filename `getTagsForWishTree.js` and
will look for the `handler` function, as it's entry point. The [Lambda](https://aws.amazon.com/lambda/)
function then gets a set of environment variables called `ENV` and `WISH_TREE_TAGS_TABLE`
which the latter contains the configured [DynamoDB](https://aws.amazon.com/dynamodb/)
table name.

The `events` section defines what API Gateway should setup, in this instance the
config asks for a `path` value of `getTagsForWishTree` using the HTTP method of
`GET`, while also ensuring that the [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
headers are set. This ensures that browsers know which resources it should
permit to be loaded, and which ones it shouldn't.

#### The Resources

This is where we can get a little creative, as the Serverless Framework allows
us to include other non-Serverless Framework resource as part of our deployment,
beyond the usual Serverless technologies of [Lambda](https://aws.amazon.com/lambda/)
and [API Gateway](https://aws.amazon.com/api-gateway/).

```yaml
resources:
  Resources:
    tagsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${file(./config.${opt:stage, self:provider.stage, 'dev'}.json):WISH_TREE_TAGS_TABLE}
        AttributeDefinitions:
          - AttributeName: Description
            AttributeType: S
        KeySchema:
          - AttributeName: Description
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 5
          WriteCapacityUnits: 5
        SSESpecification:
          SSEEnabled: true
```

Here we can see the creation of our [DynamoDB](https://aws.amazon.com/dynamodb/)
table with [Server Side Encryption](https://docs.aws.amazon.com/dynamodb-encryption-client/latest/devguide/client-server-side.html)
being enabled to ensure that the wish tree tags are secured for our use only. A
small call out to the [AWS Well-Architected Framework's](https://aws.amazon.com/architecture/well-architected/)
Security Pillar!

#### What about AWS CloudFront?

Typically an [AWS API Gateway](https://aws.amazon.com/api-gateway/) uses it's
own naming system and stage information to generate the API Endpoint, which
takes the form of:

```
https://{restapi-id}.execute-api.{region}.amazonaws.com/{stageName}
```

This isn't very friendly! With the use of [Custom Domains](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html#edge-optimized-custom-domain-names)
we are able to choose our own user-friendly domain and map it to our API
endpoint, finishing off the whole package.

{{< fancybox path="https://static.colinbarker.me.uk/img/blog/2021/03/" file="custom-domain.png" caption="The Custom Domain settings on API Gateway" gallery="gallery" >}}

### Final Words

The full setup instructions on how to build your own Wishtree application are
available on the [WishTreeApplication](https://github.com/TokonatsuFestival/WishTreeApplication)
repo on GitHub.

- Lead photo by [Lennart Jönsson](https://unsplash.com/@lenjons?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/japan-wish?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
