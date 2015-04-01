---
layout: post
title: "Happiness is a working Galera cluster on Kubernetes!"
date: 2015-04-01 12:00:00 
categories: galera,percona,kubernetes,coreos,docker,vmware
---

Yes, after many hours, late nights, hair pulling, some private yelling:

```

core@master ~/mysql_replication_kubernetes/galera_sync_replication $ kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)                                     HOST                            LABELS              STATUS
pxc-node1           10.244.72.7         pxc-node1           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.136/172.16.230.136   <none>              Running
pxc-node2           10.244.15.2         pxc-node2           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.134/172.16.230.134   <none>              Running
pxc-node3           10.244.42.2         pxc-node3           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.135/172.16.230.135   <none>              Running

mysql> show status like 'wsrep_incoming_addresses';
+--------------------------+----------------------------------------------------+
| Variable_name            | Value                                              |
+--------------------------+----------------------------------------------------+
| wsrep_incoming_addresses | 10.244.72.7:3306,10.244.42.2:3306,10.244.15.2:3306 |
+--------------------------+----------------------------------------------------+

```

I will post details about this in an upcoming post!
