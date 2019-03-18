---
title: Using Terraform and Lambda@Edge for Index Files
tags: terraform, aws, lambda, cloudfront, serverless
---

I ran into an interesting problem setting up this site over the weekend. The pipeline was set up, the infrastructure all seemed to be in place, and I was able to bring up the site just fine.
Things were looking good! I quickly noticed a problem after actually testing a few links on my site.

 I'll get into it more in a future post, but this blog is set up as a serverless site using AWS and Hexo.
[Hexo](https://hexo.io) is a blog framework that generates a static site from pages you write in markdown. When it generates the site

{% asset_img "hexo_index_files.png" image %}
