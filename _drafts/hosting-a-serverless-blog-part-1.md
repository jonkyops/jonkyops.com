---
title: "Hosting a Serverless Blog: Part 1"
categories: [posts]
tags: [terraform, aws, cloudfront, serverless]
excerpt: In this multi-part series, I go over how to host a Jekyll-based serverless blog and have a pipeline for updating it.
---

In my last post, I mentioned a couple of the benefits of having a serverless web app, cost and scalability being some of the major ones.
In this post, I go over how you can set up your Jekyll blog to be hosted "serverless-ly", as well as setting up a pipeline to deploy changes (i.e. new posts).

## Overview

This site will be set up using a combination of Jekyll for the blogging platform, codepipeline for build/deployment, and terraform for managing the infrastructure.

I'm also going to assume you have a few basic prerequisites set up:

- An AWS account, along with credentials
- A public GitHub repository containing your Jekyll site
- A Github OAuth token
- A domain registered for your blog

One thing to note is that I am setting this up with the major pieces broken down into two modules, pipeline and site, in case I want to break them out later.

## Setting up the infrastructure

Let's start by laying down the pieces that will actually host the site, which will be under "modules/serverless_site". What we're going for is using CloudFront to serve out our serverless site, so we'll need:

- Buckets for logging and hosting the site
- The CloudFront distribution
- A Lambda@Edge function to handle index.html files in subfolders (explained in [my last post](https://jonkyops.com/posts/using-terraform-and-lambda-edge-for-redirection/))
- Domain/Cert for SSL
- DNS to point the domain to our CloudFront distribution
- SSM params with the IDs of the bucket/CF distribution for the build pipeline to use

First, create our variables file:

variables.tf:

``` hcl
variable "site_domain" {
  description = "site url"
  default     = "jonkyops.com"
}
```

Obviously you'll want to change that out for your own domain name.

We're going to make an origin access identity for CloudFront to access the bucket hosting the website, that way we can lock the bucket down and make sure everyone is accessing it through the CDN.

cloudfront.tf:

``` hcl
resource "aws_cloudfront_origin_access_identity" "origin_access_identity" {
  comment = "cloudfront origin access identity"
}
```

Now we need to set up the domain in Route53, along with creating a certificate for SSL, which includes creating the certificate validation record in Route53 as well:

domain.tf:

```hcl
data "aws_route53_zone" "domain" {
  name         = "${var.site_domain}"
  private_zone = false
}

# This creates the certificate, along with the dns records for validation
resource "aws_acm_certificate" "cert" {
  domain_name       = "${var.site_domain}"
  validation_method = "DNS"
}

resource "aws_route53_record" "cert_validation" {
  name    = "${aws_acm_certificate.cert.domain_validation_options.0.resource_record_name}"
  type    = "${aws_acm_certificate.cert.domain_validation_options.0.resource_record_type}"
  zone_id = "${data.aws_route53_zone.domain.id}"
  records = ["${aws_acm_certificate.cert.domain_validation_options.0.resource_record_value}"]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "cert" {
  certificate_arn         = "${aws_acm_certificate.cert.arn}"
  validation_record_fqdns = ["${aws_route53_record.cert_validation.fqdn}"]
}
```

Our buckets for logging and hosting the site:

s3.tf

``` hcl
resource "aws_s3_bucket" "logs" {
  bucket_prefix = "${var.site_domain}-site-logs"
  acl           = "log-delivery-write"
  force_destroy = true

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket" "site" {
  bucket_prefix = "${var.site_domain}"
  force_destroy = true

  logging {
    target_bucket = "${aws_s3_bucket.logs.bucket}"
    target_prefix = "${var.site_domain}/"
  }

  website {
    index_document = "index.html"
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```

Policy for the bucket, which should only allow read from the cloudfront origin access identity we made earlier:

iam.tf:

``` hcl
# policy for the site to only allow access from the cloudfront origin
data "aws_iam_policy_document" "s3_policy" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.site.arn}/*"]

    principals {
      type        = "AWS"
      identifiers = ["${aws_cloudfront_origin_access_identity.origin_access_identity.iam_arn}"]
    }
  }

  statement {
    actions   = ["s3:ListBucket"]
    resources = ["${aws_s3_bucket.site.arn}"]

    principals {
      type        = "AWS"
      identifiers = ["${aws_cloudfront_origin_access_identity.origin_access_identity.iam_arn}"]
    }
  }
}

resource "aws_s3_bucket_policy" "site" {
  bucket = "${aws_s3_bucket.site.id}"
  policy = "${data.aws_iam_policy_document.s3_policy.json}"
}
```

The lambda function for handling the html files in subfolders. Here's the javascript, which will be packaged up when we run terraform deploy:

index.js

``` javascript
'use strict';
exports.handler = (event, context, callback) => {

    // Extract the request from the CloudFront event that is sent to Lambda@Edge 
    var request = event.Records[0].cf.request;

    // Extract the URI from the request
    var olduri = request.uri;

    // Match any '/' that occurs at the end of a URI. Replace it with a default index
    var newuri = olduri.replace(/\/$/, '\/index.html');

    // Log the URI as received by CloudFront and the new URI to be used to fetch from origin
    console.log("Old URI: " + olduri);
    console.log("New URI: " + newuri);

    // Replace the received URI with the URI that includes the index page
    request.uri = newuri;

    // Return to CloudFront
    return callback(null, request);
};
```

The configuration for packaging and creating the lambda function:

``` hcl
# zip up the function to send to lambda
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/index.js"
  output_path = "${path.module}/function.zip"
}

resource "aws_lambda_function" "edge_lambda" {
  function_name = "oai-url-rewrite"
  filename      = "function.zip"
  handler       = "index.handler"
  runtime       = "nodejs8.10"
  publish       = "true"
  role          = "${aws_iam_role.lambda_edge.arn}"

  depends_on = ["data.archive_file.lambda_zip"]
}

# lambda @edge role/policy
data "aws_iam_policy_document" "lambda_edge_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type = "Service"

      identifiers = [
        "lambda.amazonaws.com",
        "edgelambda.amazonaws.com",
      ]
    }
  }
}

resource "aws_iam_role" "lambda_edge" {
  name               = "lambda-edge-role"
  assume_role_policy = "${data.aws_iam_policy_document.lambda_edge_assume_role_policy.json}"
}
```

Now we're ready to create the CloudFront distribution. This will point back to our site bucket as the origin, using the origin access identity we created before. Add this to the cloudfront.tf file:

cloudfront.tf:

```hcl
resource "aws_cloudfront_distribution" "distribution" {
  enabled         = true
  price_class     = "PriceClass_100"
  aliases         = ["${var.site_domain}"]
  is_ipv6_enabled = true

  origin {
    origin_id   = "origin-bucket-${aws_s3_bucket.site.id}"
    domain_name = "${aws_s3_bucket.site.bucket_regional_domain_name}"

    s3_origin_config {
      origin_access_identity = "${aws_cloudfront_origin_access_identity.origin_access_identity.cloudfront_access_identity_path}"
    }
  }

  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "origin-bucket-${aws_s3_bucket.site.id}"

    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = "${aws_lambda_function.edge_lambda.qualified_arn}"
      include_body = false
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = "${aws_acm_certificate.cert.arn}"
    minimum_protocol_version = "TLSv1.1_2016"
    ssl_support_method       = "sni-only"
  }
}

# DNS record pointing to the cloudfront distribution
resource "aws_route53_record" "site" {
  zone_id = "${data.aws_route53_zone.domain.zone_id}"
  name    = "${var.site_domain}"
  type    = "A"

  alias {
    name                   = "${aws_cloudfront_distribution.distribution.domain_name}"
    zone_id                = "${aws_cloudfront_distribution.distribution.hosted_zone_id}"
    evaluate_target_health = false
  }
}
```

As part of our build for the serverless site, we'll need two parameters to be available in SSM; One for the S3 bucket to upload the static files to and one for CloudFront distribution ID to invalidate cached files.

ssm.tf:

```hcl
resource "aws_ssm_parameter" "site_bucket_url" {
  name      = "codebuild-siteBucketUrl"
  type      = "String"
  value     = "${aws_s3_bucket.site.id}"
  overwrite = true
}

resource "aws_ssm_parameter" "distribution_id" {
  name      = "codebuild-distributionId"
  type      = "String"
  value     = "${aws_cloudfront_distribution.distribution.id}"
  overwrite = true
}
```

And last, some outputs:

outputs.tf

```hcl
output "site_bucket_arn" {
  value = "${aws_s3_bucket.site.arn}"
}

output "site_bucket_id" {
  value = "${aws_s3_bucket.site.id}"
}

output "logs_bucket_arn" {
  value = "${aws_s3_bucket.logs.arn}"
}

output "logs_bucket_id" {
  value = "${aws_s3_bucket.logs.id}"
}
```



Notice the values under "parameter-store". These are SSM parameters that we'll be setting up a little later

how to get it done with terraform:

- variables
- ssm param for github oauth token
- s3 bucket for artifacts
- iam
- codebuild
- codepipeline
- outputs

you'll probably notice there's no deploy script built in. going into the project, havent' built cloudfront yet

create buildspec.yml (don't include deploy)

