# Colin Barker's Personal Website

Repo for the personal website of Colin Barker, using Hugo as the CMS base

## Deployment Requirements

* Hugo v0.72.0+ - installed via your favourite package manager (macOS = `brew install hugo`)

## How to use this site

### Run a local server

1) Within the root folder run:

`hugo server -D`

3) Go to your browser and head to:

`http://localhost:1313`

### Build the static content directory

1) Within the root folder run: 

`hugo --baseURL https://<insertUrl>`

2) The `public` folder will contain the generated content

3) (Optional) Sync to a static location (like S3)