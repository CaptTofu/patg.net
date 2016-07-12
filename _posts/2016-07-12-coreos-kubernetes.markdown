---
layout: post
title: "Running Kubernetes with Fleet on CoreOS"
date: 2016-07-12 12:00:00 
categories: kubernetes,coreos
---

I'm please to announce publicly available Fleet Unit Files for launching a multi-node Kubernetes cluster on CoreOS, [kubernetes-cluster-fleet](https://github.com/CaptTofu/kubernetes-cluster-fleet)


## Background 

I have run Kubernetes in any number of ways, particularly using Ansible to set it up,  but was recently wanting to see if it was possible to run Kubernetes completely using Fleet unit files on CoreoS, and after a lot of trial and error and bash duct tape, indeed possible, hence my repository [https://github.com/CaptTofu/kubernetes-cluster-fleet](https://github.com/CaptTofu/kubernetes-cluster-fleet)

## Basic idea

* When one runs Kubernetes, the API server is run, and it has in etcd an entry for the IP address it's running on found in the entry ```/registry/services/endpoints/default/kubernetes```. The unit files for every other Kubernetes component use this entry to set the value of what the API server ought to be.
* There are also unit files that set up the environment and set the host/IP entries in ```/etc/hosts``` of every server to make it easier to deploy.

## Usage

Review the fine documentation on the project page for how to run this! It is pretty simple and straightforward.

[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
