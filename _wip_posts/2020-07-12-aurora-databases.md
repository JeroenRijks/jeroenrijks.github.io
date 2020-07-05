---
layout: post
title: "Amazon Aurora databases"
subtitle: "And why I like them"
date: 2020-07-05 00:30:00 -0400
background: '/img/posts/10.jpg'
---

## Database engines
Does your ruby app care? Why does choice matter for your rails app? What are popular? What influences choice?

MySql or PG compatible Aurora, but under thge hood its different


#### Who cares if compatible?
The other aforementioned factors are much more significant. How is Aurora more cloud-friendly than others?

## Separate storage and compute power

## Read number scaling

## Read and write size scaling
## Disaster recovery cost-effectiveness

## Notes on performance
- learn about second indices


## Notes on implementation
- Read and write connection management in Rails
- Fixed endpoint creation
- Cut DB connections on switching
- Terraform management is chaotic for instances - learn about ASGs
- Version limitations
- No reserved instances if youre scaling
- ASGs vs lambdas
