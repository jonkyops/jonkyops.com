---
title: Using Terraform and Lambda@Edge for Redirection
categories: [posts]
tags: [terraform, aws, lambda, cloudfront, serverless, oai]
excerpt: Using CloudFront to host your static site can come with some caveats. In this post, I go over one you might run into and how you can fix it using Terraform.
---

AWS S3 is great for hosting static sites; it scales automatically, you only pay for what you use, and no servers to worry about! Using CloudFront as your CDN to serve out your site from endpoints all over the world is even better. Add on top of that features like WAF, signed URLs, and Lambda@Edge, and you get something truly magical.

![image](/assets/images/pacha.jpg)

## The Problem

I'll get into the details more in a future post, but this blog was set up using a lot of that magic, along with [Hexo](https://hexo.io) (I've since moved to Jekyll, because of the bigger community, but this is still relevant for static sites in general), a blogging framework that generates a static site from pages you write in markdown.

It didn't take terribly long to get everything set up (once I finally decided what to use) over this past weekend, but I did run into a couple weird snags.

Let's assume that I'm deploying my site for the first time and it only contains the default Hello World post. I'll drop the static files on S3, set up my CloudFront distribution. I also want to make sure people are accessing the files through the CDN, so I'll lock down the bucket and give an OAI, then we should be good.

Once the site is deployed, I navigate to it and see what we would expect:

![image](/assets/images/home-page.jpg)

Perfect! Let's try opening that Hello World post:

![image](/assets/images/subpage-without-index.jpg)

Weird, it worked when serving the site locally. Looking at the url in the picture above, the path is really just following the folder structure of the folder that gets created when generating the site:

![image](/assets/images/hexo-index-files.jpg)

Adding the full path to the html file actually brings you to the page:

![image](/assets/images/subpage-with-index.jpg)

## The Solution

So why is this happening? It's explained in better detail in [the AWS article about it](https://aws.amazon.com/blogs/compute/implementing-default-directory-indexes-in-amazon-s3-backed-amazon-cloudfront-origins-using-lambdaedge/), but long story short:

S3 has two endpoints, HTTP and REST. Going to a normal site served from S3 is using the HTTP endpoint, which has the ability to redirect to the index.html page when going to a subfolder. When you lock the bucket down and use an [OAI](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) (Origin Access Identity), CloudFront needs to use the REST endpoint, which doesn't support that same redirection.

They go into further detail on how to fix the issue using Lambda@Edge to rewrite the URLs as the requests come in.

Let's put this fix in, onto the Terraform configs!

### Getting it Done with Terraform

We'll start by creating a file for the Lambda@Edge function they use in their article for handling the rewrites:

index.js:

{% gist 53e61471b9ae6b38f5df7e3d1b87af47 %}

Now the lambda itself.

lambda.tf:
{% gist c16383d1a971a48862143cf88de24e98 %}

And this is a very truncated version of the cloudfront resource, including the important parts for setting the OAI and Lambda@Edge function:

cloudfront.tf:
{% gist 96f4af7392e40d461c749bed1e02a39f cloudfront.tf %}

After a quick-ish `terraform apply` (CloudFront might take a few minutes), our Lambda@Edge function is ready to go.

Let's try it out:

![image](/assets/images/subpage-without-index-works.jpg)

Success!

### Conclusion

This was just a small fix for a static site, so I'm barely scratching the surface with what Lambda@Edge can do, especially around request manipulation. Being able to run code almost anywhere in the world with little setup is incredibly useful for performance and security.

Join me next time and I'll go over the steps I used to set up this site.

### Links

- <https://aws.amazon.com/blogs/compute/implementing-default-directory-indexes-in-amazon-s3-backed-amazon-cloudfront-origins-using-lambdaedge/>
- <http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html>
