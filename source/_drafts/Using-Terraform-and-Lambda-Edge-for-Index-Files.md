---
title: Using Terraform and Lambda@Edge for Index Files
tags: 
  - terraform
  - aws
  - lambda
  - cloudfront
  - serverless
  - oai
---

AWS S3 is great for hosting static sites; it scales automatically, you only pay for what you use, and no servers to worry about! Using CloudFront as your CDN to serve out your site from endpoints all over the world is even better. Add on top of that features like WAF, signed URLs, and Lambda@Edge, and you get something truly magical.

{% asset_img "pacha.jpg" image %}

I'll get into the details more in a future post, but this blog is set up using a lot of that magic, along with [Hexo](https://hexo.io), a blogging framework that generates a static site from pages you write in markdown.

As I'm going to try to set things up to be way more secure than they need to be, I ran into interesting problem setting this site up over the past weekend.

Let's assume that I'm deploying my site for the first time and it only contains the default Hello World post. I'll drop the static files on S3, set up my CloudFront distribution. I also want to make sure people are accessing the files through the CDN, so I'll lock down the bucket and give an OAIthat we should be good after that.

Navigating to the site shows what we would expect:

{% asset_img "home_page.png" image %}  

Perfect! Let's try opening that Hello World post:

{% asset_img "subpage_without_index.png" image %}  

Weird, it worked when serving the site locally. Looking at the url in the picture above, the path is actually just following the folder structure of the folder that gets created when generating the site:

{% asset_img "hexo_index_files.png" image %}

Adding the full path to the html file actually brings you to the page:

{% asset_img "subpage_with_index.png" image %}
  
So why is this happening? It's explained in better detail in [the AWS article about it](https://aws.amazon.com/blogs/compute/implementing-default-directory-indexes-in-amazon-s3-backed-amazon-cloudfront-origins-using-lambdaedge/), but here's the long story short:

S3 has two endpoints, HTTP and REST. Going to a normal site served from S3 is using the HTTP endpoint, which will redirect to the index.html page when going to a subfolder. When you lock the bucket down and use an [OAI](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) (Origin Access Identity), CloudFront needs to use the REST endpoint, which doesn't support that same redirection.

They go into further detail on how to fix the issue using Lambda@Edge to rewrite the URLs as the requests come in.

Onto the Terraform part of the post!

Let's start by creating a file for the Lambda@Edge function they use in their article:

index.js:

{% gist 53e61471b9ae6b38f5df7e3d1b87af47 index.js %}

Now the lambda itself.

lambda.tf:
{% gist c16383d1a971a48862143cf88de24e98 lambda.tf %}

And this is a very truncated version of cloudfront, including the important parts for setting the OAI and Lambda@Edge function:

cloudfront.tf:
{% gist 96f4af7392e40d461c749bed1e02a39f cloudfront.tf %}
