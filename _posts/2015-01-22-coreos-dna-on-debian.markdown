---
layout: post
title: "CoreOS DNA on Debian"
date: 2015-01-27 12:00:00 
categories: docker,coreos,fleet,etcd,containers
---


# CoreOS DNA on Debian

For a month now, I've been meaning to post about the great work my team, the Advanced Technology Group at HP, did with regard to [Docker][Docker] and a proof-of-concept that we embarked on. Our project was named "Inserting CoreOS DNA on Debian for Creating Docker Clusters".

The goal was to create a proof of concept (POC) to meld the best parts and be usable for experimentation. We examined the criteria and choices for creating the clustered [Docker][Docker] POC and demonstrated its use.  

## Why Debian?

HP [Helion][helion] runs on Debian and uses HLinux and we wanted to see how feasible this was as well as introduce this interesting combination of components to others.

## What is CoreOS?

The strategies and architectures that influence CoreOS allow companies like Google, Facebook and Twitter to run their services at scale with high resilience.

[CoreOS][coreos] is a minimalist Linux system based on Chrome OS and uses 40% less RAM on boot than the average Linux installations. It has been rearchitected to provide features to run modern infrastructure stacks by facilitating the building of customized platforms depending on the requirements of a given invidual or organization. 

As mentioned on [CoreOS's website][coreos], its development and architecture is intended to allow companies like Google, Facebook, and Twitter to run their services at scale and with high resilience. 

CoreOS includes the necessary components to run Docker clustered across machines. The components are:

* [systemd][systemd] - standard with newer viersions of Linux. This is a system of daemons, libraries, and utilities that replace upstart and init scripts. Systemd uses the concept of unit files that define what process to run, how to run it, and ensures that there are no conflicts with the process. The version of Debian we built our POC on included systemd by default.

* [etcd][etcd] - this is a simple, highly available, durable, and distributed (using Raft) key/value store, written in Go, that allows simple values pertaining to a given machine or subsystem that other components can utilize to work in conjunction with. Its API is simple HTTP/JSON.

* [Fleet][fleet] - this is a rudimentary scheduler for Docker, designed to be very low-level and to be a foundation for higher order orchestration. Fleet is written in Go, fleet runs on a single machine and uses etcd to discover information about services and containers on other machines. Fleet is the interface that uses systemd across machines to control the state of docker containers, in essence providing a distributed systemd functionality.


## Implementation

Using HP Cloud, we built a VM image with the aforementioned components, using Ansible on a stock Debian setup. Using that image, we launched a given number (in our case 5) VMs to demonstrate the usefulness of fleet with a sample [ELKstack][elkstack] application. The ELKStack sample app was a modified version of Marcel DeGraaf's excellent [blog post][marcel_blog] and code to demonstrate our proof of concept worked in a practical way, showing how fleet intelligently scheduled docker containers across machines in the cluster. 

## The Sample Application

The sample application consisted of the following containers and numbers when run:

* Kibana + Elasticsearch (1) - this container was built to have both Kibana for visualization as well as the data storage and search functionality provided by Elasticsearch. This container receives events to be stored from the central logstash agent. 
* Logstash (1) - A container for centralized logging to be forwarded to the Elasticsearch container 
* Sinatra sample app (5) - Very simple Sinatra/Ruby web application combined with a logstash-forwarder (using the Lumberjack protocol over SSL). The application also produces logging events that are written to logs in the format Lumberjack expects which are also being watched by the logstash-forwarder to be sent to the central Logstash agent. 
* Nginx (1) - Simple Nginx configuration to proxy to the Sinatra web application containers 
* Test script container (1) - Simple container that automatically runs a script that accesses the Nginx front end that in turn produces log events that in turn result in Kibana showing traffic against the web application.

### Systemd magic

Each of these containers is specified within a systemd service file that fleet utilizes when interacting with systemd on each node of the cluster. Systemd provides a means to ensure that a service (corresponds to container) has a distinct name and that there cannot be conflicts. Also systemd provides a means to run certain things before and after a service is started. Some of these are:

* ```ExecStartPre``` - this runs before the container is launched. For all the containers in this example, the container is pre-pulled from a Docker repository
* ```ExecStart``` - runs the actual container
* ```ExecStartPost``` - sets each container's IP address as ```/container/host``` - ie. ```/elasticsearch/host```
* ```ExecStop``` - Kills the container
* ```ExecStopPost``` - Remove the IP entry from etcd

This is an extremely useful functionality systemd provides that makes applications like this possible!

### Launch order


For all of these containers except the Kibana/Elasticsearch container, they each need to have the ability once running to know the IP address of the other containers. Logstash needs to know the IP of Kibana/Elasticsearch in order for forward events to, the Sinatra web app containers need to know the IP address of the Logstash container for each logstash-agent to forward events to the Logstash central logging agent, the Nginx container needs to know which Kibana web application containers there are to proxy to. The tesp script container needs to know what the IP is of Nginx.

This discovery capability is provided by the very thing that makes Fleet able to do its job: etcd

These containers are luanched using the ```fleetctl`` utility (once the service file is loaded) in the given order listed above. When each container is launched, the service file for each specifies a step to be run immediately after the service is launch which in the case of these services, the IP address of each is stored in etcd to be in turn used by the next container launched. 

For instance, when the Elasticsearch container is launched, it registers its IP address as ```elasticsearch/host``` in etcd. Each of these containers is "baked" with another great utility that is luanched with a simple ```boot.sh``` script: [confd][confd]

### Confd another ingredient of magic

```confd``` on each container is configured to generated the necessary configuration files using a confd template that interpolates the variable provided from etcd.

This can be explained better this way:

* The Elasticsearch/Kibana container launches, stores its IP address in etcd as ```/elasticsearch/host```

* The Logstash container launches, stores its IP address as ```/logstash/host``` and confd is launched, generated logstash.conf from a template using the Elasticsearch/Kibana IP entry as well as generating an SSL cert and private key and storing that in etcd

* The Sinatra containers are launched, each having different ports, storing their IP addresses as ```/sinatra/port/host``` and then generates the logstash forwarder config file as well as the SSL cert and private key for the forwarder, from etcd.

* The Nginx container is launched and confd generates the nginx config file using the entries in ```etcd``` for each sinatra container to know which to proxy to.

* The test script container is launched and the test script is templated to be generated to know what the IP of the nginx container is to run against

The one thing this sample app for our POC made clear to me: the very thing that makes the docker clustering work can in turn provide a great means of providing flexible discovery for a given application. The one thing that comes in mind for me that I've had to make work with numerous tricks through the years using Chef, Saltstack, and Ansible is Galera clustering.

## Summary

There is so much about the proof-of-concept that we worked on that I want to write about and will have subsequent posts about a number of things such as other Docker clustering and scheduling projects, the Ansible work that was done, the work we did to make this work on HP Cloud considering private and public IPV4 addresses, as well as more details on how to use ```confd``` which is a tool that I really had fun aquainting myself with.

Stay tuned!


[Docker]: http://docker.io
[Docker.inc]: http://docker.com
[docker_signup]: https://www.docker.io/account/signup/
[docker_installation]: http://docs.docker.io/installation/#installation
[kernel_features]: http://www.kbartocha.com/tag/linux-kernel-namespaces/
[cgroups]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch01.html
[libcontainer]: http://blog.docker.com/2014/03/docker-0-9-introducing-execution-drivers-and-libcontainer/
[lxc]: https://linuxcontainers.org/
[parallels]: http://www.parallels.com/
[openvz]: http://openvz.org/Main_Page
[freebsd_jail]: http://www.freebsd.org/doc/handbook/jails.html
[dockerfile]: http://docs.docker.io/reference/builder/
[docker_ansible]: http://docs.ansible.com/docker_module.html
[docker_image_ansible]: http://docs.ansible.com/docker_image_module.html
[docker_inventory_ansible]: https://github.com/ansible/ansible/blob/devel/plugins/inventory/docker.yml
[salt_states_dockerio]: http://docs.saltstack.com/en/latest/ref/states/all/salt.states.dockerio.html
[gareth-docker]: https://forge.puppetlabs.com/garethr/docker
[chef_docker]: http://www.getchef.com/blog/2014/04/23/chef-docker-automating-container-workflows/
[Ansible]: http://www.ansible.com/home
[SaltStack]: http://www.saltstack.com/
[Puppet]: http://puppetlabs.com/puppet/puppet-enterprise?gclid=CKvX14_85b4CFSgQ7AodhlMAwQ
[Chef]: http://www.getchef.com/chef/
[Solum]: https://wiki.openstack.org/wiki/Solum
[ansible_galaxy]: https://galaxy.ansible.com
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
[nova_containers_openstack]: http://blog.docker.io/2013/06/openstack-docker-manage-linux-containers-with-nova/
[dockenstack]: https://index.docker.io/u/ewindisch/dockenstack/
[openstack_docker]: https://wiki.openstack.org/wiki/Docker
[openshift]: https://www.openshift.com/?sc_cid=70160000000UJArAAO&gclid=COfd-Oz-5b4CFcHm7AodS1gA7Q
[freebsd_jail]: http://www.freebsd.org/doc/handbook/jails.html 
[docker_image_registry]: https://registry.hub.docker.com/
[helion]: http://www8.hp.com/us/en/cloud/helion-overview.html
[coreos]: http://coreos.com
[systemd]: http://www.freedesktop.org/wiki/Software/systemd/
[etcd]: https://github.com/coreos/etcd
[fleet]: https://github.com/coreos/fleet
[confd]: https://github.com/kelseyhightower/confd
[golang]: http://golang.org
[elkstack]: http://www.elasticsearch.org/webinars/elk-stack-devops-environment/ 
