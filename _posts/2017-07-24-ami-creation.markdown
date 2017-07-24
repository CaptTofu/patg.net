---
layout: post
title: "AMI creation - what is your process?"
date: 2017-07-24 12:00:00 
categories: AWS, AMI, images, instances 
---



## What is your process for creating AMIs? 

I've had a discussion with a colleague about the process of creating AMIs, or more specifically, dealing with instances that need some sort of state captured. AWS offers you the ability to create an image off of an instance. We were pondering in our discussion how that can be possible with the situation of how can AWS ensure that the state of what is being made into an image is what you expect it to be? Open file handles, things in memory needing to be flushed to disk. Is this feature of EC2 something that should even be used for an authoritative state you expect your image to be?

In our case, it was an instance that was on a host that had problems and needed to be moved off and the underlying host become unresponsive (according to AWS) and the instance couldn't even be shut down.

The basic way to go about it would be to build a new instance, provision it the way you want it (Ansible, Chef, manual configuration) and once complete, stop the instance, and make an image from that. 

What is your process? And those of you intimate with the details of EC2, how fool-proof is the 'create an image' functionality?
