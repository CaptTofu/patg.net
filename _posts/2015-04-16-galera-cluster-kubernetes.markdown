---
layout: post
title: "Running Galera on Kubernetes"
date: 2015-04-21 12:00:00 
categories: galera,percona,kubernetes,coreos,docker,vmware
---


I recently gave a presentation at [Percona Live 2015](http://www.percona.com/live/mysql-conference-2015/) in Santa Clara, CA. In this presentaiton I originally wanted to simply show running MySQL replication, first asynchronous, and more importantly, a Galera cluster, and in so doing, demonstrate how useful [Kubernetes][Kubernetes] is. 

## Why?

The talk was a good chance to introduce the MySQL community-- developers, DBAs, sysadmins, and others to what Kubernetes is and what it means for MySQL

## A bit of learning
I thought at the time when I submitted my synopsis that the talk would be straightforward. About 2-3 months ago, I started working on the setup I would use for the demonstration. My goal was to use a stock CoreOS cluster with the necessary Kubernetes components installed and running as a cluster. 

The reality was that there was a bit more to it than that. Isn't that how everything that has to do with complex systems is? To make a long story short, I tried the Vagrant setup for CoreOS but using the cloud-init scripts in Kubernetes documentation but I could never get complete success running a [Kubernetes][kubernetes] cluster this way. Hence, the [blog post][kubernetes_blog_post] I recently published that covered my basic setup. 

Finally, using the process outlined in that [post], I had a Kubernetes cluster that consistently worked for the most part. Some gotchas were that upon launching the cluster, the cloud-init scripts had dependencies that required downloading various binaries required to run Kubernetes and set up networking. A slow network connection resulted in failure because of this particular timing-- something I plan to fix and contribute back to the community.

## Asynchronous Replication

With a working Kubernetes cluster, I decided it was time to first start with regular MySQL asyncronous replication since it might present a more simple proof of concept. The way to do this was essentially to modify the standard MySQL Docker container to have a master and slave variant. The higher abstraction of this is that there will be two pods - a master pod, and a slave pod. For the master pod, only one container will run. The slave pod could run one or more containers. 

The master container is built using a Dockerfile that specifies and entrypoint shell script. This is the basic pattern that the stock MySQL container uses, albeit only to set up essential MySQL settings, particularly the root user and password. This modifications to this entrypoint script sets up the replication user privileges (name, password, and host to allow). In order to do this, when the container is started, environment variables are passed from the pod configuration file supplying the mysql root password, replication username, and replication password. The host to allow connection from uses 10.x.x.x as that's the IP range that Kubernetes uses to assigns to pods. This range would cover any container in the slave pod(s) that would need to connect as a slave. With these environment variables, the entrypoint script builds up an SQL script that runs these priviledge modification is run with MySQL in insecure mode (for initialization) using ```mysqld --initialize-insecure=on```. Additionally, a script called "random.sh" runs to set the server-id value in my.cnf. Once the master pod is running, a master service is started called mysql_master which the great functionality of Kubernetes makes availble as environment variables ```MYSQL_MASTER_HOST``` and ```MYSQL_MASTER_PORT``` on any container launched there afterword, including the slave container.

From the entrypoint script:

```
echo "GRANT REPLICATION SLAVE, REPLICATION CLIENT on *.* TO '$MYSQL_REPLICATION_USER'@'10.100.%' IDENTIFIED BY '$MYSQL_REPLICATION_PASSWORD';" >> "$tempSqlFile"
```

The slave container is built similar to the master container with regard to the Dockerfile specifying an entrypoint script, except instead of setting up privileges, it sets up replication by running the ```CHANGE MASTER...``` in the sql script that is built up using the aforementioned environment variables both passed and available through Kubernetes ```MYSQL_MASTER_HOST``` which is the master the slave is set up to read from. 

From the entrypoint script:

```
if [ ! -z "$MYSQL_MASTER_SERVICE_HOST" ]; then
    echo "STOP SLAVE;" >> "$tempSqlFile"
    echo "CHANGE MASTER TO master_host='$MYSQL_MASTER_SERVICE_HOST', master_user='$MYSQL_REPLICATION_USER', master_password='$MYSQL_REPLICATION_PASSWORD';">> "$tempSqlFile"
    echo "START SLAVE;" >> "$tempSqlFile"
fi
```

This actually is quite straightforward and worked the first time I prototyped it. I first ran it as two separate containers, passing the environment variables explicitly - like the example below:

```
docker run -e MYSQL_ROOT_PASSWORD=c-kr1t capttofu/mysql_master_kubernetes
docker run -e MYSQL_MASTER_SERVICE_HOST=x.x.x.x -e MYSQL_ROOT_PASSWORD=c-kr1t capttofu/mysql_slave_kubernetes
```  

Once I verified this, it was a matter of creating master and slave pod files (to view follow links)

This proved the basic concept worked. That being, using an entrypoint script to set up the dabase in advance.

## Galera replication

For Galera replication, it seemed it might actually be more simple since when setting up Galera replication one need not concern themselves with binary log position nor how to get a snapshot of data-- that being handled by Galera (SST - single state transfer when joining). The difficulty was due to the fact that services can only have a single port and IP using the version of Kubernetes that I had to use for my demo. Galera replication requires 4 ports: 3306, 4444, 4567, and 4568. In newer versions of Kubernetes support multiple ports. The way I planned to get around this is that I took advantage of the read-only Kubernete API running on the host value found in the enviroment variable ```$KUBERNETES_RO_SERVICE_HOST``` on every container Kubernetes starts (in a pod). The Kubernetes client ```kubectl``` is included on the Docker image. The entrypoint script in turn runs ```kubectl``` and parses the output for every pod named "pxc_0", iterating from 1 to 3, in a loop, building up the string used for ```wsrep_cluster_address```. Of course, if the container is launched and the environment variable ```WSREP_CLUSTER_ADDRESS``` is set to ```gcomm://```, then that value is used, in this case the pod ```pxc_node1```, the "bootstrap" pod. 

Galera replication is pretty simple once you know which hosts will be part of the cluster. In this case, the pattern is to launch the ```pxc_node1``` pod as the bootstrap pod, then ```pxc_node2``` and ```pxc_node3```. When this is completed, there should be a cluster.


## Actual steps


First, set up a Kubernetes cluster per my [blog post][kubernetes_blog_post].

### Pre-reqs

Build the kubernetes client program:

```
$ git clone https://github.com/GoogleCloudPlatform/kubernetes 
$ cd kubernetes
kubernetes $ make
kubernetes $ sudo cp cmd/kubectl /usr/local/bin

```


### Clone the kubernetes mysql replication repository


```
$ git clone https://github.com/CaptTofu/mysql_replication_kubernetes.git
$ cd mysql_replication_kubernetes
mysql_replication_kubernetes $ git submodule init
mysql_replication_kubernetes $ git submodule update
```


### Create pxc_01 pod

```
mysql_replication_kubernetes $ cd galera_sync_replication
galera_sync_replication $ kubectl create -f pxc-node1.yaml 
pxc-node1
```


### Verify pod is running

```
galera_sync_replication $ kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)                                     HOST                            LABELS              STATUS              CREATED
pxc-node1           10.244.78.2         pxc-node1           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.131/172.16.230.131   name=pxc-node1      Pending 5 Seconds 

```

In the example above, the status is ```Pending```. Once the status is ```Running```, create the second pod

### Create pxc-node2 and pxc-node3 pod

Once pxc_node1 has a status of ```Running```, create pxc_node2 and pxc_node3:

```
galera_sync_replication $ kubectl create -f pxc-node2.yaml 
pxc-node2
galera_sync_replication $ kubectl create -f pxc-node3.yaml 
pxc-node3
```

### Create a service for pxc-node1

From before, recall that pxc-node1 is running on the kubernetes minion/node with an IP address of 172.16.230.131. Edit the configuration file for pxc_node1 service to make it possible to connect to the pxc_node1 pod using that address with ```publicIPs```. Edit pxc-node1-service.yaml:

```
---
  id: pxc-node1
  kind: Service
  apiVersion: v1beta1
  port: 3306
  containerPort: 3306
  selector:
    name: pxc-node1
  labels:
    name: pxc-node1
  publicIPs:
  - 172.16.230.131
```

Once this file is ready, create the service

```
galera_sync_replication $ kubectl create -f pxc-node3.yaml 
pxc-node3
```


### Verify everything is running

There should be all three pods running (status ```Running```) and a single pxc_node1 service:

```
galera_sync_replication $ kubectl get pods,services
POD                 IP                  CONTAINER(S)        IMAGE(S)                                     HOST                            LABELS              STATUS              CREATED
pxc-node1           10.244.78.2         pxc-node1           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.131/172.16.230.131   name=pxc-node1      Running             About an hour
pxc-node2           10.244.75.2         pxc-node2           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.139/172.16.230.139   name=pxc-node2      Running             About an hour
pxc-node3           10.244.11.2         pxc-node3           capttofu/percona_xtradb_cluster_5_6:latest   172.16.230.144/172.16.230.144   name=pxc-node3      Running             54 minutes
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes          component=apiserver,provider=kubernetes   <none>              10.100.0.2          443
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.100.0.1          80
pxc-node1           name=pxc-node1                            name=pxc-node1      10.100.43.123       3306
```

The output above shows that everything is up and running-- time to connect to the database!


### Access ```pxc-node1``` service

Services are created immediately, so the database can be immediately accessed

```
$ mysql -u root -p -h 172.16.230.131
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.6.22-72.0-56 Percona XtraDB Cluster (GPL), Release rel72.0, Revision 978, WSREP version 25.8, wsrep_25.8.r4150

Copyright (c) 2000, 2015, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show status like 'wsrep_inc%'
    -> ;
+--------------------------+----------------------------------------------------+
| Variable_name            | Value                                              |
+--------------------------+----------------------------------------------------+
| wsrep_incoming_addresses | 10.244.78.2:3306,10.244.11.2:3306,10.244.75.2:3306 |
+--------------------------+----------------------------------------------------+
1 row in set (0.01 sec)
```

This output shows that all three Galera nodes are up and running!

## Summary

With this proof of concept, there is much more to do. Most of all, it would be good to use replication controllers instead of simple pods to create the three galera single-container pods. That way, there is a means of ensuring that all pods will continue to run. It would also be good to demonstrate this proof-of-concept's value by launching an application that uses this Galera cluster. At least at this point, there is something very useful to start with! 

Special thanks to -- Kelsey Hightower, Tim Hockin, Daniel Smith and others in #google-containers for their patience and excellent help!


[kubernetes_blog_post]: http://patg.net/kubernetes,coreos,docker,vmware/2015/03/23/kubernetes-vmware-cluster/
[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
