---
layout: post
title: "Terraform setup repository for AWS"
date: 2016-09-21 12:00:00 
categories: terraform,aws 
---

Just a quick post for a project I put together using Terraform to [build out ready-to-use networking on aws](https://github.com/CaptTofu/terraform-aws-setup)

## What it does in a nutshell

This project basically builds out a VPC with 4 subnets (one public, three private) that is all ready to use to build out your AWS environment using [Terraform](https://www.terraform.io/).

It's really simple to use. You just need to edit the `variables.tf` to suit your AWS account credentials as well as whatever else you want to adjust. 

Sanity check with `terraform plan` and then launch with `terraform apply`. 

That's it!

Add more hosts, run whatever you need. All the networking (routing, gateways, security groups, etc) is ready to use.
