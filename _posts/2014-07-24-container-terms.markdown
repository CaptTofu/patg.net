---
layout: post
title: "Docker: Containers for the Masses -- Getting more terms straight 
date: 2014-07-24 12:00:00
categories: docker
---

I was recently discussing container technologies with my team and was trying to explain the various container-related projects including LXC, libcontainer, Docker, Kubernetes, CoreOS, and how all these fit together.

I agreed that a blog post is a good way to clarify things. I wanted to continue my blog posts, particularly about Ansible and Docker as well as the [Ansible][Ansible] dynamic inventory plugin for Docker, though wanted to get my thoughts out before I forget (I have young children)!

This blog post is the latest in the series:

- [Introduction][docker_intro_blog]
- [Installation][docker_install_blog]
- [Using Docker][docker_using_blog]
- [Ansible and Docker][docker_ansible_blog]
- [Building Docker Images with Ansible][docker_images_ansible_blog]
- [Building Docker Images with Ansible (yet another way)][docker_images_ansible_blog2]
- [Anisible Docker Facts module][docker_facts_blog]

<br />

## Virtual Machines

The first term is one that anyone reading this article already knows, but I find it useful to revisit it when discussing containers. I myself still fall back into thinking about virtual machine concepts when discussing containers which is a normal way to grok a new concept, though containers really are different and using accurate terminology is ultimately a requirement to understanding something correctly.

A virtual machine, or for the sake of discussion, a VM, is emulation software that provides the ability to run programs as if you are running them on real hardware, constrained to that machine, that environment.

This usually means you install an OS on a virtual machine and on top of that OS, programs of your choice to make your VM complete. You can start and stop a VM, create an image based off of the VM at a given state and launch those images to have more copies of a given machine and environment.

There are various types of implementations that provide virtual machines including: KVM, VMware, Virtualbox, Windows Virtual PC, and QEMU.

In terms of containers, VMs can be incredibly useful for setting up something lik e Docker -- giving one the ability to run numerous VMs with containers running on each VM. VMs are also useful for composing blog posts and providing a test environment for Jekyll!

## Containers

A container is an operating system-level virtualization method that provides an self-contained execution environment, that look and feel like virtual machines, but run within the OS and us the OS itself to provide this functionality. Containers don't require installing and operating system. When you run a container, you run whatever program that you want to run in the container without the overhead having to run an entire operating system. The processes run in a container are only visible inside the container and isolated from the host OS and other containers and don't require any emulating to run.

Container functionality is made possible by Linux kernel features such a cgroups, namespaces, apparmor, networking interfaces, and firewalling rules - the idea being, use the host OS to create and provide an environment where disk, CPU, memory, and networking work as if on a host of its own, just as you have with a VM.

There are various mechanisms that make containers possible: chroot, Docker, LXC, OpenVZ, Parallels, Solaris Containers, FreeBSD Jail. This blog post hones in on [Docker][Docker] and similar Linux containers.


## LXC

Until recently, [LXC][lxc] was the default execution environment for Docker. [LXC][lxc] stands for LinuX Containers. LXC combines [cgroups][cgroups], Linux Kernel namespaces, apparmor profiles, seccomp policies, and chroots to provide containers-- or what they refer to as "Chroot on steroids". 

[LXC][lxc], [written in C][lxc_repo], provides a library, [liblxc][liblxc], language bindings, a set of tools for runnign containers. 


## Libcontainer

[Libcontainer][libcontainer], written in the [Go language][go_lang], is the default execution environment for [Docker][Docker]. [libcontainer][libcontainer] provides essentially the same thing that [LXC][lxc] provides but in [Go][go_lang] with no external dependencies, as well as a goal to be underlying container technology agnostic.


## CoreOS

[CoreOS][coreos] is a stripped-down Linux OS based off of [Chrome OS][chromeos] that uses [Docker][Docker] containers to run multiple isolated Linux systems for resource partitioning. 

[CoreOS][coreos] provides [etcd][etcd], a key/value store written in [Go][golang] to provide both distributed configuration information as well as service discovery for the cluster, implemented using what [CoreOS][coreos] also provides called [Fleet][fleet], a cluster management daemon used to control systemd on each node.


## Kubernetes

[Kubernetes][kubernetes]

[Docker]: http://docker.io
[Ansible]: http://www.ansible.com/home
[ansible_dynamic_inventory]: http://docs.ansible.com/intro_dynamic_inventory.html#dynamic-inventory
[ansible_example_repository]: https://github.com/ansible/ansible-examples
[ansible_documentation]: http://docs.ansible.com/
[ansible_apt_module]: http://docs.ansible.com/apt_module.html
[ansible_docker_module]: http://docs.ansible.com/docker_module.html
[ansible_docker_image_module]: http://docs.ansible.com/docker_image_module.html 
[ansible_docker_facts_module]: https://github.com/CaptTofu/ansible/tree/docker_facts
[ansible_docker_dynamic_inventory]: https://github.com/ansible/ansible/blob/devel/plugins/inventory/docker.py
[docker_py]: https://github.com/dotcloud/docker-py
[docker_intro_blog]: http://patg.net/containers,virtualization,docker/2014/06/05/docker-intro/
[docker_install_blog]: http://patg.net/containers,virtualization,docker/2014/06/09/docker-install/
[docker_using_blog]: http://patg.net/containers,virtualization,docker/2014/06/10/using-docker/
[ansible_playbooks]: http://docs.ansible.com/playbooks.html
[ansible_example_playbook_repository]: https://github.com/ansible/ansible-examples
[docker_cli]: https://docs.docker.com/reference/commandline/cli/
[AnsibleFest]: http://www.marketwired.com/press-release/speakers-from-twitter-google-twilio-edx-rackspace-headline-ansiblefest-nyc-2014-1902858.htm
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
[ansible_docker_presentation_repo]: https://github.com/CaptTofu/ansible-docker-presentation
[moonshot]: http://h17007.www1.hp.com/us/en/enterprise/servers/products/moonshot/index.aspx#.U0gU2PldXJh?jumpid=ps_r163&k_clickid=AMS|112|61913|169782eb-9c7a-df89-4afe-0000588f5dc0
[moonshot_cartridge]: http://www8.hp.com/us/en/products/proliant-servers/product-detail.html?oid=5375897#!tab=features
[docker_image_source]: https://github.com/CaptTofu/docker-image-source
[docker_ansible_blog]: http://patg.net/ansible,docker/2014/06/18/ansible-docker/
[docker_images_ansible_blog]: http://patg.net/ansible,docker/2014/06/20/ansible-docker-image/
[docker_images_ansible_blog2]: http://patg.net/ansible,docker,mysql,galera,clustering/2014/06/24/ansible-docker-image-part2/
[docker_image_hub]: https://registry.hub.docker.com/
[ansible_facts]: http://docs.ansible.com/playbooks_variables.html#information-discovered-from-systems-facts
[ansible_docker_facts_repo]: https://github.com/CaptTofu/ansible/tree/docker_facts
[ansible_repo]: https://github.com/ansible/ansible
[libcontainer]: https://github.com/docker/libcontainer
[go_lang]: http://golang.org/
[lxc_repo]: git://github.com/lxc/lxc
