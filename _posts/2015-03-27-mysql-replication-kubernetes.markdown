---
layout: post
title: "Simpe MySQL replication example on Kubernetes"
date: 2015-03-27 12:00:00 
categories: mysql,kubernetes,coreos,docker,vmware
---

I'm happy to show a simple example I developed for running simple [MySQL replication on Kubernetes](https://github.com/CaptTofu/mysql_replication_kubernetes.git).

The reason I've done this is that I'm in the process of preparing a [presentation for Percona Live](http://www.percona.com/live/mysql-conference-2015/sessions/running-galera-cluster-kubernetes) and decided to start out with basic replication. I figured if I get that working, getting Galera replication using Percona XtraDB Cluster will be even easier since SST makes it easier and less complicated when a node joins the cluster versus a slave connecting to a master and having to concern itself with binary log position and getting a snapshot that corresponds to that.


Using my blog post to easily build a Kubernetes cluster with VMware [```kubernetes_cluster_vmware```](https://github.com/CaptTofu/kubernetes_cluster_vmware) all one has to do to use these files is clone [the mysql_replication_kubernetes repo](https://github.com/CaptTofu/mysql_replication_kubernetes.git)

The README in the repo will explain how to run this simple proof-of-concept. 

## Example

On the master (where I usually create pods, services, replication controllers...) create the master pod and service:

<br />
```
core@master ~/mysql_project $ kubectl create -f mysql-master.json 
mysql-master
core@master ~/mysql_project $ kubectl create -f mysql-master-service.json 
mysql-master
```

Create the slave pod and service:



```
core@master ~/mysql_project $ kubectl create -f mysql-slave.json 
mysql-slave
core@master ~/mysql_project $ kubectl create -f mysql-slave-service.json 
mysql-slave
```


Verify things are running:


```
core@master ~/mysql_project $ kubectl get pods,services
POD                 IP                  CONTAINER(S)        IMAGE(S)                                  HOST                        LABELS              STATUS
mysql-master        10.244.19.6         mysql-master        capttofu/mysql_master_kubernetes:latest   192.168.1.19/192.168.1.19   name=mysql-master   Running
mysql-slave         10.244.88.3         mysql               capttofu/mysql_slave_kubernetes:latest    192.168.1.21/192.168.1.21   name=mysql-slave    Running
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes          component=apiserver,provider=kubernetes   <none>              10.100.0.2          443
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.100.0.1          80
mysql-master        name=mysql-master                         name=mysql-master   10.100.154.2        3306
mysql-slave         name=mysql-slave                          name=mysql-slave    10.100.194.208      3306
```


Connect to the master (see the master-service's IP). Notice here that the same container that the master is run using is used simply for the MySQL client:


```
core@node_01 ~ $ docker run -it capttofu/mysql_master_kubernetes mysql -u root -proot -h 10.100.154.2
Welcome to the MySQL monitor.  Commands end with ; or <snip>
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show slave hosts; 
+-----------+--------------+------+-----------+--------------------------------------+
| Server_id | Host         | Port | Master_id | Slave_UUID                           |
+-----------+--------------+------+-----------+--------------------------------------+
|     24698 | feec6e4827bc | 3306 |     29002 | dfcb0b0a-d4a6-11e4-b2db-02420af45803 |
+-----------+--------------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)
```

Connect to the slave:


```
core@node_01 ~ $ docker run -it capttofu/mysql_master_kubernetes mysql -u root -proot -h 10.100.194.208
<snip>

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.100.154.2
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 2803
               Relay_Log_File: mysqld-relay-bin.000003
                Relay_Log_Pos: 3015
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: mysql.%
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 2803
              Relay_Log_Space: 102663
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 29002
                  Master_UUID: d79ef03a-d4a6-11e4-b2db-02420af41306
             Master_Info_File: /var/lib/mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
1 row in set (0.01 sec)

```

Excellent! A master slave setup!




## Summary

This is a proof of concept. There is so much more to be done with this example pertaining to regular replication:

* How to add slaves (if replicas for slaves is increased) to the master after the binary log has incremented
* How to automatically perform a slave snapshot/backup and send that to the joining slave
* Volumes... 
* Intelligent connection pooling for an application using this setup

Kubernetes is a rapidly developing project and expect many things to change and projects to emerge that solve many of these issues-- I may not be aware of something that has already done what my POC here accomplishes. In any case, this should prove to be a useful example for those who are MySQL users who want to acquaint themselves with Kubernetes!
