---
title: GitHub Actions and OIDC Update for Terraform and AWS
author: Colin Barker
date: 2025-01-12T14:32:19.0000Z
description: A little while after creating my original blog post on how to setup GitHub Actions using AWS and OpenID, AWS created a list of trusted providers with their own library therefore negating the need for a Thumbprint. This post updates my previous blog post on the matter!
tags:
  - aws
  - github
  - security
  - iam
  - openid
categories:
  - aws
image: https://static.colinbarker.me.uk/img/blog/2023/02/roman-synkevych-wX2L8L-fGeA-unsplash.jpg
---

> Header photo by [Roman Synkevych 🇺🇦](https://unsplash.com/@synkevych?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/wX2L8L-fGeA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

> **NOTE**: This blog posts follows on from my original post back in 2023 where I first set this up. You can find the original blog post [GitHub Actions using AWS and OpenID](/blog/2023-02-28-github-actions-using-aws-and-openid) by using the link.

## Before we begin

Please make sure to read my original blog post, it has been updated with the new steps, but this post goes on to specifically call out the changes that allowed this to happen.

## What changed?

A few months beyond my original blog post, GitHub and AWS updated the way that they authenticate between each other, by removing the need to pin the certificate thumbprint as part of the OIDC authentication process. [GitHub Blog Post](https://github.blog/changelog/2023-07-13-github-actions-oidc-integration-with-aws-no-longer-requires-pinning-of-intermediate-tls-certificates/). AWS added GitHub to one if its many root certificate authorities (CAs), meaning GitHub can update their authentication certificate without the need for us to update the thumbprint that existed in the setup step of the OIDC setup. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html)

For a while the [AWS SDK](https://github.com/aws/aws-sdk-go/blob/main/CHANGELOG.md#release-v15120-2024-04-11) and the [Terraform AWS Provider](https://github.com/hashicorp/terraform-provider-aws/pull/37255) had not been updated, so still required a thumbprint to be added to the OIDC provider to allow Terraform to create the resource. Thankfully, in December, AWS updated their main Go SDK, which allowed the owners of the Terraform AWS Provider to make their change to make the Thumbprints Optional.

**NOTE** For existing implementations, this might be harder to work in, due to a quirk with the AWS API and the way Terraform works. There is a [bug issue](https://github.com/hashicorp/terraform-provider-aws/issues/40509) open for this. Essentially Terraform will not update the API if there is no default setting, clearing out the Thumbprint list with no default means Terraform will not update the API, so the thumbprints are not removed. As the default behaviour for AWS OIDC configurations that are trusted by AWS means that it ignores the thumbprint list, this should not cause any issues with existing OIDC setups. This is only an issue with Terraform, and not the AWS API.

## Creation of the OpenID Connect Provider

Setting up the Identity Provider (IdP) will need to be the updated. The [walkthrough](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) documentation, has also been updated:

- The provider URL - in the case of GitHub this is `https://token.actions.githubusercontent.com`
- The "Audience" - which scopes what can use this. Confusingly in Terraform this is also known as the `client_id_list`.
- You no longer need to generate the thumbprint

### Adding the resource in Terraform

Now that we have the only two bits we need, it is easy enough to amend / change your Terraform code to set up the [OpenID Connect Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider) in IAM. This code example below has been updated to use the only bits you need, removing the need for the old `thumbprint_list`.

```terraform
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  client_id_list = [
    "sts.amazonaws.com"
  ]
}
```

Everything else inside the original blog post still stands.

## What about other OpenID Connect Providers?

There are only a limited number of providers that AWS have on their root certification CA approved list. Therefore you will need to read the documentation on each provider to find out. If you are still using another OIDC provider with AWS, and need to use the thumbprint, then the usual steps still apply. Below is an updated version of the steps needed for a 3rd party OIDC provider.

You will need to ensure you have the following

- The provider URL - this is normally supplied by your third party, and should be a public address
- The "Audience" - which scopes what can use this. Once again in Terraform this is also known as the `client_id_list`.
- The Thumbprint of the endpoint - This one is the tricker one, as you will need to generate this yourself.

### Generating the thumbprint

For this example, you will need to ensure that you have downloaded the OIDC provider certificates, or have them to hand to generate the thumbprint. Ideally, if you can get the thumbprint from the provider, that would mean you can skip all the steps.

1. If you know the url that needs to be used, then you can use the following `openssl` command like before.

```
openssl s_client -servername tokens.endpoint.oidc.provider.com -showcerts -connect tokens.endpoint.oidc.provider.com:443
```

2. Grab the certificate shown in the output, you will see this starting with `-----BEGIN CERTIFICATE-----`, then place this content into a file. For this demo, I will use `openid.crt`.

3. Use the OpenSSL command again to generate the fingerprint from the file created above.

```
openssl x509 -in openid.crt -fingerprint -sha1 -noout
```

Which should output the fingerprint. Strip away all the extra parts, and the `:` between each of the pairs of hexadecimal characters, and you should end up with something like this:

```
6938fd4d98bab03faadb97b34396831e3780aea1
```

⚠️ **Note:** You will need to make the letters lowercase, as Terraform is case sensitive for the variable we need to put this in, but AWS is not case sensitive, so it can sent Terraform into a bit of a loop. ⚠️

### Adding the resource in Terraform

With all of this, you can now start to put together the Terraform elements. We can use the Terraform resource for setting up the [OpenID Connect Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider) in IAM. Below is a code example, using the information gathered from the documentation and the thumbprint generation, and all placed into a single resource object.

```terraform
resource "aws_iam_openid_connect_provider" "other_oidc_provider" {
  url = "https://tokens.endpoint.oidc.provider.com"
  client_id_list = [
    "sts.amazonaws.com"
  ]
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]
}
```

## Round up

While this is a shorter than normal blog post, this should hopefully show the differences between the two types now. Once we get an update on how the bug regarding existing OIDC setups work, I am sure I will post another update!

Any questions, please let me know!
