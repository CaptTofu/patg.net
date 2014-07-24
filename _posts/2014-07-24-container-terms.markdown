---
layout: post
title: "Docker: Containers for the Masses -- Getting terms straight"
date: 2014-07-24 12:00:00
categories: docker
---

I was recently discussing container technologies with my team and was trying to explain the various container-related projects including LXC, libcontainer, Docker, [Kubernetes][kubernetes], [CoreOS][coreos], and how all these fit together.

I agreed that a blog post would be a good way to further clarify some terms. I wanted to continue my more in-depth example-laden blog posts, particularly about Ansible and Docker as well as the [Ansible][Ansible] dynamic inventory plugin for Docker, though wanted to get my thoughts out before I forget (I have young children)!

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

There are various types of implementations that provide virtual machines including: [KVM][kvm], [VMware][vmware], [Virtualbox][virtualbox], [Windows Virtual PC][virtualpc], and [QEMU][qemu].

In terms of containers, VMs can be incredibly useful for setting up something like Docker -- giving one the ability to run numerous VMs with containers running on each VM. VMs are also useful for composing blog posts and providing a test environment for Jekyll!

## Containers

A container is an operating system-level virtualization method that provides an self-contained execution environment, that look and feel like virtual machines, but run within the OS and us the OS itself to provide this functionality. Containers don't require installing and operating system. When you run a container, you run whatever program that you want to run in the container without the overhead having to run an entire operating system. The processes run in a container are only visible inside the container and isolated from the host OS and other containers and don't require any emulating to run.

Container functionality is made possible by Linux kernel features such a [cgroups][cgroups], namespaces, apparmor, networking interfaces, and firewalling rules - the idea being, use the host OS to create and provide an environment where disk, CPU, memory, and networking work as if on a host of its own, just as you have with a VM.

There are various mechanisms that make containers possible: chroot, [Docker][Docker], [LXC][lxc], [OpenVZ][openvz], Parallels, Solaris Containers, FreeBSD Jail. This blog post hones in on [Docker][Docker] and similar Linux containers.


## LXC

Until recently, [LXC][lxc] was the default execution environment for Docker. [LXC][lxc] stands for LinuX Containers. LXC combines [cgroups][cgroups], Linux Kernel namespaces, apparmor profiles, seccomp policies, and chroots to provide containers-- or what they refer to as "Chroot on steroids".

[LXC][lxc], [written in C][lxc_repo], provides a library, [liblxc][liblxc], language bindings, a set of tools for runnign containers.


## libcontainer

[libcontainer][libcontainer], written in the [Go language][golang], is the default execution environment for [Docker][Docker]. [libcontainer][libcontainer] provides essentially the same thing that [LXC][lxc] provides but in [Go][golang] with no external dependencies, as well as a goal to be underlying container technology agnostic.


## CoreOS

[CoreOS][coreos] is a stripped-down Linux OS based off of [Chrome OS][chromeos] that uses [Docker][Docker] containers to run multiple isolated Linux systems for resource partitioning. 

[CoreOS][coreos] provides [etcd][etcd], a key/value store written in [Go][golang] to provide both distributed configuration information as well as service discovery for the cluster, implemented using what [CoreOS][coreos] also provides called [Fleet][fleet], a cluster management daemon used to control systemd on each node.

This is certainly a topic this blog will revisit!


## Kubernetes

[Kubernetes][kubernetes] another topic the author has devled into and will have a separate post on in the future, is a project by Google ([Google Cloud Platform][google_cloud_platform] guys). It is an Open Source container cluster management project.

Google uses container technology thoughout Google to both scale out and provide security for a number of applications such as search and Gmail where they run up to 2 billion containers a week — 3300/sec! GCE -- Google Cloud Environment-- runs VMs inside of containers for resource isolation between VMs and non-VM workloads.

Google doesn't currently use Docker internally yet has written kernel features that make containerization possible cgroups (control groups) which are used to limit, account, and isolate resource usage (CPU, memory, disk I/O) of process groups. There is also the work Google did with [LMCTFY][lmctfy] (Let Me Contain That For You), which is the open-source version of Google's container stack (https://github.com/google/lmctfy) which the functionality is being moved into libcontainer, the current default [Go][golang]-based container driver for Docker.

Kubernetes is written in [Go][golang] and the idea is to build on top of Docker, which Google sees as a technology that will be a standard for containerization (hence the aforementioned work they are doing). It is a scheduler for containers organized into what it refers to as "pods" as well as providing communication for containers.

The primary purpose of pods is support of co-located and co-managed helper programs such as content management systems, logging, log management and backup, proxies, bridges, adaptors, etc.

Also, like CoreOS does, Kubernetes makes use of [etcd][etcd] which provides the persistent state of the master.

There is much more information about Kubernetes, and as already has been mentioned, there will be a future blog post specifically covering Kubernetes.


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
[docker_facts_blog]: http://patg.net/ansible,docker/2014/07/10/ansible-docker-facts/
[ansible_docker_facts_repo]: https://github.com/CaptTofu/ansible/tree/docker_facts
[ansible_repo]: https://github.com/ansible/ansible
[libcontainer]: https://github.com/docker/libcontainer
[golang]: http://golang.org/
[lxc]: http://linuxcontainers.org
[lxc_repo]: https://github.com/lxc/lxc
[liblxc]: https://github.com/lxc/lxc
[etcd]: https://github.com/coreos/etcd
[lmctfy]: https://github.com/google/lmctfy
[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
[coreos]: https://coreos.com/
[fleet]: https://github.com/coreos/fleet
[kvm]: http://www.linux-kvm.org/page/Main_Page
[qemu]: http://wiki.qemu.org/Main_Page
[vmware]: http://www.vmware.com/
[openvz]: http://openvz.org/Main_Page
[virtualbox]: https://www.virtualbox.org/
[virtualpc]: http://www.microsoft.com/en-us/download/details.aspx?id=3702
[cgroups]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch01.html
[google_cloud_platform]: http://googlecloudplatform.blogspot.com/
