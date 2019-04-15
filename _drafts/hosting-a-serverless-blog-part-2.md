---
title: "Hosting a Serverless Blog: Part 2"
categories: [posts]
tags: [terraform, aws, cloudfront, serverless]
excerpt: In this multi-part series, I go over how to host a Jekyll-based serverless blog and have a pipeline for updating it.
---

## Setting up for CodeBuild

We'll need to add a buildspec.yml file to the root of the repository. These are the steps that are run by CodeBuild to get the site ready to be deployed:

buildspec.yml:

``` yaml
version: 0.2

env:
  variables:
    LC_ALL: "C.UTF-8"
    LANG: "en_US.UTF-8"
    LANGUAGE: "en_US.UTF-8"
  parameter-store:
    SITE_BUCKET_URL: "codebuild-siteBucketUrl"
    DISTRIBUTION_ID: "codebuild-distributionId"

phases:
  install:
    commands:
      - bundle install
  build:
    commands:
      - bundle exec jekyll build
  post_build:
    commands:
      - aws s3 sync --delete _site/ s3://$SITE_BUCKET_URL/
      - aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths '/*'
```
