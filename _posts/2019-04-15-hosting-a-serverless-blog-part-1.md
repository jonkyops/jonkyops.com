---
title: "Hosting a Serverless Blog: Part 1"
categories: [posts]
tags: [terraform, aws, cloudfront, serverless]
excerpt: In this multi-part series, I go over how to host a Jekyll-based serverless blog and have a pipeline for updating it.
---

In my last post, I mentioned a couple of the benefits of having a serverless web app, cost and scalability being some of the major ones.
In this post, I go over how you can set up your Jekyll blog to be hosted "serverless-ly", as well as setting up a pipeline to deploy changes (i.e. new posts).

## Overview

This site will be set up using a combination of Jekyll for the blogging platform, CodePipeline for build/deployment, and terraform for managing the infrastructure.

I'm also going to assume you have a few basic prerequisites set up:

- An AWS account, along with credentials
- A public GitHub repository containing your Jekyll site
- A Github OAuth token
- A domain registered for your blog

One thing to note is that I am setting this up with the major pieces broken down into two terraform submodules, pipeline and site, to keep it a little more organized.

## Setting up the infrastructure

Let's start by laying down the pieces that will actually host the site, which will be under "/modules/serverless_site". What we're going for is using CloudFront to serve out our serverless site, so we'll need:

- Buckets for logging and hosting the site
- The CloudFront distribution
- A Lambda@Edge function to handle index.html files in subfolders (explained in [my last post](https://jonkyops.com/posts/using-terraform-and-lambda-edge-for-redirection/))
- Domain/Cert for SSL
- DNS to point the domain to our CloudFront distribution
- SSM params with the IDs of the bucket/CloudFront distribution for the build pipeline to use

Since the site portion of the configurations are going to be in their own module, start working in /modules/serverless_site.

Create our variables file:

/modules/serverless_site/variables.tf:
{% gist d703c67b34303fee9b7779167d284d51 variables.tf %}

Obviously you'll want to change that out for your own domain name.

We're going to make an origin access identity for CloudFront to access the bucket hosting the website, that way we can lock the bucket down and make sure everyone is accessing it through the CDN.

/modules/serverless_site/cloudfront.tf:
{% gist d703c67b34303fee9b7779167d284d51 cloudfront_1.tf %}

Now we need to set up the domain in Route53, along with creating a certificate for SSL, which includes creating the certificate validation record in Route53 as well.

/modules/serverless_site/domain.tf:
{% gist d703c67b34303fee9b7779167d284d51 domain.tf %}

Next, the buckets for logging and hosting the site. You'll probably notice I'm using the 'bucket_prefix' property instead of 'bucket', this lets you always get a unique bucket name. The bucket name doesn't matter too much, since you would normally have a R53 record pointing to it and only CloudFront will be accessing it anyway.

/modules/serverless_site/s3.tf:
{% gist d703c67b34303fee9b7779167d284d51 s3.tf %}

Policy for the bucket, which should only allow read from the cloudfront origin access identity we made earlier.

/modules/serverless_site/iam.tf:
{% gist d703c67b34303fee9b7779167d284d51 iam.tf %}

The lambda function for handling the html files in subfolders. Here's the javascript, which will be packaged up when we run terraform deploy.

/modules/serverless_site/index.js:
{% gist 53e61471b9ae6b38f5df7e3d1b87af47 index.js %}

The configuration for packaging and creating the lambda function:

/modules/serverless_site/lambda.tf:
{% gist 53e61471b9ae6b38f5df7e3d1b87af47 lambda.tf %}

Now we're ready to create the CloudFront distribution. This will point back to our site bucket as the origin, using the origin access identity we created before. Update the cloudfront.tf file:

/modules/serverless_site/cloudfront.tf:
{% gist d703c67b34303fee9b7779167d284d51 cloudfront_2.tf %}

As part of our build for the serverless site, we'll need two parameters to be available in SSM; One for the S3 bucket to upload the static files to and one for CloudFront distribution ID to invalidate cached files.

/modules/serverless_site/ssm.tf:
{% gist d703c67b34303fee9b7779167d284d51 ssm.tf %}

And last, some outputs.

/modules/serverless_site/outputs.tf:
{% gist d703c67b34303fee9b7779167d284d51 outputs.tf %}

Now the site module is done, so let's go back to the root of the project (would be / where we were working in /modules/serverless_site before). Let's create the variables file and include the variable for the site module.

/variables.tf:
{% gist 5f8017202e1fe425a081edc4b2e86786 variables.tf %}

Again, changing the domain.

This may not be the best way to organize the config files, but I like to keep my main.tf to just providers, locals, and state storage.

/main.tf:
{% gist 5f8017202e1fe425a081edc4b2e86786 main.tf %}

And finally, call on the module we just created.

/site.tf:
{% gist 5f8017202e1fe425a081edc4b2e86786 site.tf %}

Now we can init and apply:

```shell
terraform init --backend=mybackendfile
terraform apply
```

Then we have everything we need to host the site (not quite build, that's coming in the next post) up and running.

## Conclusion

Bit of a longer post this time, but now we have the infrastructure in place to host our serverless blog. If you want to try it out now, you can build your site locally, then manually copy the S3 files to the site bucket.

Check out part 2 of this series (when it comes out) to learn how to set up an automated pipeline for publishing your posts.
