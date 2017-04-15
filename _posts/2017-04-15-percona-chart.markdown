---
layout: post
title: "Percona Chart now in Kubernetes Charts"
date: 2017-04-15 12:00:00 
categories: percona,mysql,kubernetes,helm
---



## [Percona Helm Chart](https://github.com/kubernetes/charts/tree/master/stable/percona)

I'm happy to see that my pull request was approved. Pretty simply, it was a Percona Chart. We needed the setup at HPE for our cluster and it was a good excuse to put together a chart, so there ya go!

## What is a Helm chart?

Well, for those that don't know, some prerequisite questions:

### What is [Helm](https://github.com/kubernets/helm)?

Helm is a package manager for Kubernetes, these packages called "[charts](https://github.com/kubernetes/charts)"

### What is [Kubernetes](https://github.com/kubernetes)?

Kubernetes is an open-sourced system for managing containerized applications across multiple hosts

### What are containers?

[Containers](https://www.redhat.com/en/containers) are concept of technology in an operating system that isolation such that you can package an application and ship it anywhere and have it run the same exact way.
