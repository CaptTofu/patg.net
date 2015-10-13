---
layout: post
title: "Galera on Kubernetes Example now merged"
date: 2015-10-13 12:00:00 
categories: galera,percona,kubernetes,coreos,docker
---


I'm quite pleased to announce that my pull request for Kubernetes which provides a 3-node Galera cluster for Kubernetes was merged last week. I gave a presentation at [Percona Live 2015](http://www.percona.com/live/mysql-conference-2015/) in Santa Clara, CA back in April about this an had since modified it to use newer Kubernete's functionality, particular now that Kubernetes services allow more than one port and Kubernetes requires several ports. Also, Skydns, which provides the ability to specify the name (vs. IP address) of each node's service when building up the ```wsrep_cluster_address``` string as well as providing a cluster-wide service.

Tim St. Clair (Google) was quite through and very helpful in his review and the code and docuementation ended up being very complete. You can find it on Kubernetes' source code repo:

[https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-galera](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-galera)  

The README contains everything you need to know to get your cluster up and running. 

I have plans including an HA Proxy server to this cluster as well as other examples including asynchronous replication.

[kubernetes_blog_post]: http://patg.net/kubernetes,coreos,docker,vmware/2015/03/23/kubernetes-vmware-cluster/
[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
