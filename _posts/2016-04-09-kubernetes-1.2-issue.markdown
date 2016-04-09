---
layout: post
title: "Pre-conference preparation breakage"
date: 2016-04-09 12:00:00 
categories: kubernetes,ubuntu,cgroups,docker
---
# Pre-conference preparation breakage

Just a quick post to help others who get stuck on the problem I solved today that could save a lot of time!

## Introduction

Of course it's Saturday (what else is there to do?!), and there is a little over a week before I present on [running a Galera Cluster MySQL on Kubernetes](https://www.percona.com/live/data-performance-conference-2016/sessions/running-galera-cluster-kubernetes) and while trying to record the demo, everything was apparently broken. Here's the tale of woe in a nutshell.

## Initial problem

So, when trying to run my [kubernetes cluster][kubernetes_blog_post], when I ran 

    patg@kubernetes-master001:~$ ./kubectl exec pxc-node2-hjjrw -i -t -- mysql -u root -p -h pxc-cluster
    Content-Type specified (plain/text) must be 'application/json'

Normally, this would bring me into an active MySQL client session.

## Diagnosis and side-effect of cure

A bit of searching, seems it's something that Docker changed in the client API and requires a new [kubernetes][kubernetes] version. The version I have is 1.0.6. I try 1.1.1. Still, no go. So I try the latest 1.2.

The build script is useful:

    ~/kubernetes/cluster/ubuntu$ export KUBE_VERSION=1.2.2
    ~/kubernetes/cluster/ubuntu$ ./build.sh 

The binaries are copied over to the minion VMs and restarted (kube-proxy and kubelet). They are not registering with the master, so the journal shows:

    Apr 09 11:03:52 kubernetes-minion003 kubelet[13645]: I0409 11:03:52.213704   13645 kubelet.go:2365] skipping pod synchronization - [Failed to start ContainerManager system validation failed - Following Cgroup subsystem not mounted: [memory]]

## Cure to side-effects of original cure

Not as bad as statin drug side-effects, but something that had to be fixed. The cause is that Kubernetes does a check for whether the memory cgroup is mounted. Looking at the source (Use the source!) [line 128 of container manager in Kubernetes](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/container_manager_linux.go#L128).

    if expectedCgroups.Len() > 0 {
        return f, fmt.Errorf("%s - Following Cgroup subsystem not mounted: %v", localErr, expectedCgroups.List())
    }

More searching, and found [this page](http://unix.stackexchange.com/questions/157361/lxc-start-no-cgroup-mounted-on-the-system) with a similar issue on lxc.

The fix for cgroups is:

1. Add to ```/etc/fstab```

        cgroup /sys/fs/cgroup cgroup defaults,blkio,net_cls,freezer,devices,cpuacct,cpu,cpuset,memory,clone_children 0 0

2. Add to ```/etc/default/grub```:

        GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory,namespace"

3. Run ```update-grub```
4. Reboot

And this fixes the problem! 

```
mysql-galera patg$ kubectl exec -it pxc-node3-iu89r -- mysql -u root -p -h pxc-cluster
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.6.24-72.2-56-log Percona XtraDB Cluster (GPL), Release rel72.2, Revision 43abf03, WSREP version 25.11, wsrep_25.11

Copyright (c) 2009-2015 Percona LLC and/or its affiliates
Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
1 row in set (0.00 sec)
```

Phew! I was seriously fit to be tied!

[kubernetes_blog_post]: http://patg.net/kubernetes,coreos,docker,vmware/2015/03/23/kubernetes-vmware-cluster/
[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
