---
layout: post
title: "Docker: Containers for the masses -- Installation"
date: 2014-06-09 12:00:00
categories: containers,virtualization,docker
---

Welcome back! This is the second of a series of blog posts on [Docker][Docker]. The [previous post][docker_intro] introduced the reader to [Docker][Docker]: the concept of containers and the technology that makes them possible. This current post will cover installation and configuration of [Docker][Docker] and issues that the author of this post has found in using Docker that are worth sharing.


## Installing and configuring Docker

The [installation][docker_install] directions on the [Docker website][Docker] are excellent and provide instructions for the various Linux distributions as well as running on virtual machines on non-Linux OSs.

In addition to their instructions, I needed containers to use 0.0.0.0 when binding container ports so it will be possible to connect to a container running on one cartridge from an external server (external to the [Docker][Docker] host). The following was added to the end of the file /etc/defaults/docker:

    DOCKER_OPTS="--ip=0.0.0.0"

With this option, it is possible to run containers and have any exposed ports on the container to be bound to ports on the host. There will be more elaboration on this in a later post in this series.


## Running the Docker Daemon: UNIX domain socket or TCP socket

By default, [Docker][Docker] will runs so that it binds to a UNIX domain socket versus a TCP socket on 127.0.0.1. There is a good reason for this: [Docker][Docker] runs as root. There are risks of cross-site-scripting attacks using a TCP socket if you are not on a completely trusted network or VPN (or both).

I recently investigated and ended up demonstrating using [Ansible][Ansible] [modules (see presentation)][ansible_docker_presentation] to manage containers run across 45 cartridges on a Moonshot server. Since the network was behind a VPN, completely trusted, and locked down (even using an internal apt repo) the [Docker][Docker] daemon was set to run as a service on port 4243 so that the [Ansible][Ansible] modules, run via playbooks on a single host, could connect and communicate with these 45 [Docker][Docker] daemons running on each cartridge. This also makes it possible to connect to Docker locally as a non-privileged user. 

The option for using a TCP socket setting in /etc/defaults/docker:

    DOCKER_OPTS="--host=tcp://0.0.0.0:4243”

I will again stress that this was a need for my specific setup from a single host to an HP Moonshot system with 45 cartridges on an internal network that ould only be accessed through a VPN with no external access. ONLY use a setup like this if you have a secure setup. Also make sure that access to port you use (for instance, 4243 in the example above) is limited to only the host that needs to access it. In my case, it was the host I was running [Ansible][Ansible] playbooks on.


## Sign up for an account

So you can share your images as well as follow this blog post, it is recommended that you sign up for an Docker account. Follow the '[sign up][docker_signup]' link next to 'login' in the upper right hand corner of the page. This will be a name you want to use to tag your images with and the string value that when searched on, will display your images for someone interested.


## About using sudo to run the [Docker CLI][docker_cli]

The documentation on the [Docker][Docker] site has docker commands run via sudo. This is a best practice and assumed setup for security purposes. As mentioned before, [Docker][Docker] runs as root, so it's important to make sure that only trusted users can run [Docker][Docker]. In the case of most developers running [Docker][Docker] locally either on a laptop of VM, since there would only be one user to concern themselves with, could set up their system to allow a given user the ability to not have to preface every docker command with sudo. The following will set this up:

    $ sudo groupadd docker
    $ sudo gpasswd -a myusername docker # myusername == user in question
    $ sudo service docker restart

Note: The above steps are for Ubuntu. Mileage may vary for other Linux variants.


## Summary

This the second blog post in the series on Docker, and discussed Docker installation and configuration. Specifics that the author has found in working with Docker particular to configuration issues were provided, including: how to run the Docker daemon, running Docker bound to a UNIX socket versus a TCP socket, and the use of sudo for running the [Docker CLI][docker_cli]. 

The next post will cover actual usage of [Docker][Docker].




[Docker.inc]: http://docker.com
[Docker]: http://docker.io
[docker_cli]: http://docs.docker.com/reference/commandline/cli/
[docker_intro]: http://patg.net/containers,virtualization,docker/2014/06/05/docker-intro/
[docker_install]: https://docs.docker.com/installation/#installation
[docker_signup]: https://www.docker.io/account/signup/
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
[garety-docker]: https://forge.puppetlabs.com/garethr/docker
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
