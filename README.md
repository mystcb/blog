# Colin Barker's Personal Website

This repo contains my personal blog website, using Hugo to generate static
content for deployment into AWS S3.

## Contents

- [Tooling Requirements](#tooling-requirements)
- [Local Development](#local-development)
  - [Running the local server](#running-the-local-server)
  - [Building the static content](#building-the-static-content)

# Tooling Requirements

The following tools are required for testing and editing the site locally, note
that in reality, the only tool you actually need for website development is the
Hugo CLI however, other tooling is listed for those wanting to test other parts
of the site.

| Tool                                   | Version | Notes                                                                                  |
| -------------------------------------- | ------- | -------------------------------------------------------------------------------------- |
| [Hugo](https://gohugo.io/)             | v0.81.0 | Static Website Generator, your favourite package manager (macOS = `brew install hugo`) |
| [AWS-CLI](https://aws.amazon.com/cli/) | v2      | The AWS CLI to upload the static content to S3 (testing only)                          |
| [Act](https://github.com/nektos/act)   | v0.2.20 | Local GitHub actions tester (only if you want to test GitHub actions)                  |

# Local Development

When developing my blog, it is best to use Hugo to ensure that you are looking
at a working version of the site before you upload it to the public. Below is
how you can enable access to the site locally for testing.

## Running the local server

1. Within the root folder run:

`hugo server -D`

3. Go to your browser and head to:

`http://localhost:1313`

## Building the static content

This is not really needed, but good for testing that it exports properly to the
public folder, that is used to sync to AWS.

1. Within the root folder run:

`hugo --baseURL https://<insertUrl>`

2. The `public` folder will contain the generated content

3. (Optional) Sync to a static location (like S3)
