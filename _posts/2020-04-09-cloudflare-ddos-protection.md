---
layout: post
title: "Implementing CloudFlare's DDoS protection"
subtitle: "Feat. Terraform"
date: 2020-04-09 00:30:00 -0400
background: '/img/posts/06.jpg'
---

### The Stack

I had set up a website on AWS S3, and managing it using Terraform. In this blog post, I will discuss how I use CloudFlare for prepare my website for the real world. The objectives are to:
- Get my S3 website hosted on a public-facing domain managed in CloudFlare
- Increase performance of my S3-hosted website
- Increase security of my S3-hosted website

The S3 bucket's Terraform set-up is:

```
resource "aws_s3_bucket" "website" {
  bucket        = local.bucket_name
  acl           = private
  region        = var.aws_region
  force_destroy = false

  website {
    index_document = "index.html"
    error_document = "error.html"
  }
}
```

## CloudFlare

CloudFlare acts as a content delivery network, by caching web content in lots of geographical locations, speeding up response times by serving content that has been cached physically near the end user.

CloudFlare also provides a firewall to monitor and filter your web traffic, based on CloudFlare-managed or self-managed rules. CloudFlare's managed firewall rules are great because they are continuously updated using web traffic experienced by all of their clients. For example, if a CloudFlare client experiences a DDoS attack, where multiple malicious machines overload a web address with traffic (with the intent of preventing the website from serving normal users), these malicious IP addresses are added to an IP blacklist, and their requests are blocked for all CloudFlare firewall users.

## The initial solution
I figured that creating a CNAME from my S3 URL to my CloudFlare domain would improve my website's security with CloudFlare's firewall.

## The problem

You may think that setting up an internet-facing URL (with a firewall) in CloudFlare would increase your S3-hosted website's security, because all requests sent to the public-facing URL must make it through the firewall. However, CloudFlare is accessing the bucket contents via the bucket URL (bucket.s3-website.region.amazonaws.com), and so can anybody else. If malicious users were to discover the S3 bucket's URL, they circumvent CloudFlare's firewall and send requests directly to S3, and since there would be no CDN, DDoS attacks would be even easier.

## The new solution

To stop the S3 bucket from being publicly accessible, we want our S3 bucket policy to only give our developers and CI/CD product write access to the bucket, and only give CloudFlare can read from the bucket. This requires the following bucket configuration:

### Public access
The `Block Public Access` settings should be enabled fully, as shown in the following Terraform module.

```
resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls        = true
  block_public_policy      = true
  ignore_public_acls       = true
  restrict_public_buckets  = true
}
```

### Bucket policy
The bucket policy will allow the `s3:GetObject` action for all requests coming from CloudFlare's IP addresses. We can limit successful requests using the bucket policy's `IpAddress` test, using CloudFlare's Terraform provider to give us a big list of CloudFlare IPs. 

```
provider "cloudflare" {
  version = "~> 2.0"
}

data "aws_iam_policy_document" "grant-cloudflare-ips" {
  statement {
    sid = "CloudflareIPsGetObject"

    actions = [
      "s3:GetObject"
    ]

    effect = "Allow"

    resources = [
      "arn:aws:s3:::${local.bucket_name}/*"
    ]

    condition {
      test     = "IpAddress"
      variable = "aws:SourceIp"
      values   = concat(data.cloudflare_ip_ranges.cloudflare.ipv4_cidr_blocks, data.cloudflare_ip_ranges.cloudflare.ipv6_cidr_blocks)
    }
  }
}
```

> Note: Our developers and CI pipeline grant appropriate permissions for the S3 bucket in the policies attached to their IAM groups. If you want all of your bucket policies kept in one place, then I'd suggest adding a `principals {...}` section to your policy document.

Finally, attach your policy to your bucket:
```
resource "aws_s3_bucket_policy" "static-bucket" {
  bucket = aws_s3_bucket.website.id
  policy = data.aws_iam_policy_document.grant-cloudflare-ips-and-devs.json
}
```

### CloudFlare CNAME

```
data "cloudflare_zones" "zones" {
  filter {
    name  = var.cloudflare_hosted_zone
  }
}

resource "cloudflare_record" "cname" {
  zone_id = data.cloudflare_zones.zones.zones[0].id
  name    = var.hostname  ## Set this to your desired subdomain
  type    = "CNAME"
  value   = "s3-website.region.amazonaws.com"  ## If your S3 bucket has a different name to your desired subdomain name, I'd suggest creating an alias to the bucket, and pointing your CNAME at that instead.
  ttl     = var.cloudflare_cname_ttl  ## This is up to you
  proxied = true
}
```

### Conclusion

After applying all of the changes in Terraform, I tried to access the S3 URL, and my request was denied (as expected), since only CloudFlare's IP addresses are whitelisted.

<img class="img-fluid" src="{{site.baseurl}}/assets/img/403_S3_cloudflare.png" alt="403 error message from S3">

However, accessing the CloudFlare-managed URL gave me access to the website right away.