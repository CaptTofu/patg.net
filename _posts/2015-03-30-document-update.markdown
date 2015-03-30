---
layout: post
title: "Updates to CoreOS and Kubernetes cluster automation projects"
date: 2015-03-30 12:00:00 
categories: kubernetes,coreos,docker,vmware
---

Just a quick post to let people know I've updated both my simple projects that [launch CoreOS cluster ```coreos_cluster_vmware```](https://github.com/CaptTofu/coreos_cluster_vmware) and [a Kubernetes cluster ```kubernetes_cluster_vmware```](https://github.com/CaptTofu/kubernetes_cluster_vmware)

The changes are simply:

* Using NAT for networking of virtual machines
* Removed annoying prompts for "did you copy this..." 
* Made Kubernetes minions headless
* Removed any use of ```sudo``` to run ```vmrun```, as it is not needed

Have fun with these!
