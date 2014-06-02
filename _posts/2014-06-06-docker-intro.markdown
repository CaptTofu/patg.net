---
layout: post
title: "Docker: Containers for the masses" 
date: 2014-06-05 12:00:00
categories: containers,virtualization,docker
---

# Docker: containers for the masses

For several months now, I've been meaning to post to this blog an entry about [Docker][Docker]. My primary tasks within the HP Advanced Technology Group over the last few months have been to research both [Docker][Docker] and [Ansible][Ansible]. This included me realizing I wanted to add a docker_facts module to Ansible as well as present a talk at AnsibleFest NYC in May. Since I have been wanting to write about my work, I have wanted to have something for readers to refer back to in order to give both context as well as some basic understanding about one of the core components of my work so as to avoid any revisitation of the basics and have a link to refer back to, hence this post!

Every time I begin to write about [Docker][Docker], there is so much on the topic to write about that the post really needs to be several posts. This will be the beginning of those posts, and an introduction to Docker.

## What are containers and how do they work?

The first thing in discussing [Docker][Docker] would be to explain its most basic component, the container.

A container is an operating-system level virtualization method that provides a completely isolated environment, what would appear to be its own system that runs on a single host. It gives the user the ability to have and environment to do whatever is needed, in particular develop and run applications having the necessary resources and environment.

There are numerous types of container mechanisms and technologies such as chroot, [OpenVZ][openvz], [Parallels][parallels], [FreeBSD Jail][freebsd_jail], Linux Containers ([LXC][lxc]), and [libcontainer][libcontainer], which is the default execution environment.

The basic idea of a container is to contain a process or set of processes such that a they run in a container and appear to have their own PID and are not visible outside of the container. This also means networking, users, disk, and other aspects that make an environment usable. This isolation is made possible on Linux using a number of kernel features: [kernel namespaces][kernel_features] (network, user, IPC, uts, PID, and mount), [cgroups][cgroups], Apparmor/SELinux profiles, and secomp policies.

## What is [Docker][Docker]?

As described on the website [Docker.io][Docker.io] run by [Docker.inc][Docker.inc] (formerly DotCloud), and corporate entity behind the Docker project, [Docker][Docker] is a "an easy, lightweight virtualized environent for portable applications". To elaborate, [Docker][Docker] is an application the extends containers (as opposed to virtual machines). Specifically, its value is that is makes it incredibly easy to manage containers and make it possible to deploy applications in manageable units that can run anywhere. The days of having to deal with the hassle of application dependencies across various Linux variants or even different types of virtual machines or hardware are now made trivial. With Docker, you can build your application in a container that runs on your laptop or on a virtual machine and be assured that it can also run in any environment. This truly is revolutionary in terms of software development, testing and deployment.

[Docker][Docker] consists of (1) a daemon that is run that is interacted through a RESTful API using the (2) [Docker][Docker] command line interface. This CLI also includes the ability to search and "pull" images from both (3) public and private [Docker][Docker] image repositories.



## What is the difference between containers and virtual machines?

One of the first questions that is often asked, especially considering the prevalent use of virtual machines (VM) is "what is the difference between virtual machines and containers?". The answer simply is that a virtual machine requires some sort of emulation layer or hypervisor complete with an OS installation for each VM whereas a container uses features in the Linux kernel (3.x) to provide an isolated virtual environment-- disk, memory, networking, etc., on the same OS, using the OS in question and not requiring an install of anything other than the files requires for your application.

If you think about this, consider your application in terms of what files on disk it requires as well as considering what processes are required when instantiated using a container, is far less less disk space and memory than is required to run on a virtual machines with all of its own resource prerequisites.

The next question that is commonly asked: which of the two is better? The answer is: neither. It really depends what you are trying to do -- each has its purpose. Sometimes, a virtual machine is needed to provide an environment that is satisfied with a full, complete OS installation and other associated functionalities while in other applications, in the case of [Docker][Docker], requirements are satisfied with an isolated environment for developing an application and developers wish to ship the application -- either to a testing group or even a customer-- as a single unit. Those are two such use cases among countless others for both virtual machines and containers that illustrate some of what each is suited for.

One other use case for a virtual machine: to run [Docker][Docker] on, particularly on non-Linux OSs!


-- insert image used in slides for VM vs. container --

## Why Docker?

Why would you want to use [Docker][Docker]? There are numerous reasons that I could cite an d some simple Google searches and reading would explain. My experiences in using it allowed me to draw my own conclusions:

### Docker is Fast!

I was very accustomed to thinking in terms of virtual machines -- concepts and terminology, coming from a background where on a daily basis, virual machines would solve so many of my development needs-- whether it was running VMware on a laptop with a development virtual machine for my various projects or working on various seplatform service applications that required automation and provisioning of numerous virtual machines on HP Cloud (Nova).

Particlarly with automation of virtual machines on the cloud, testing deployments using either Chef or SaltStack and the required waiting for a run to finish and to start seeing how the application as a whole could often take a long time and at times resemble watching paint dry.

When I first started investigating [Docker][Docker], I would provision numerous containers to get a feel for how it worked with SaltStack and Ansible. The thing that struck me the most was how fast launching containers is. This is because containers don't require an OS installation, hence not require having to boot an OS. A container consists of only the files and processes that an application requires, so launching a container takes as how long as it takes for your application to start.

### Docker is simple

Simply put, [Docker][Docker] is very simple to install, configure, and use. Their [installation page][docker_installation] covers installing [Docker][Docker] on any number of Linux variants as well as other OSs using virtual machines.

Once Docker is up and running, pulling images from the public [Docker][Docker] image repository is a snap which you can then quickly deploy containers from, make changes and create your own images, and another swath of icing on the cake: push up your images to share with others on the public [Docker][Docker] image repository.

The [Docker][Docker] command-line tool is simple to use, intuitive and offers plenty of help for the numerous sub-commands.

### Docker is a great tool for developers

Virtual machines have always been useful to me for doing local development work on a workstation or laptop. I usually end up setting up my virtual machine in a very particular way and hesitate to modify it or do something that my jeopardize how it's set up. If I have to create a new virtual machine, I have to go through the motions of installing the OS and setting up the virtual machine with the specific things I expect. I have found since using Docker, that my containers are much more "disposable" than the virtual machines I use. Plus, they take far fewer resources both in terms of disk and memory. I can run more of them and keep more images on disk than I would with a virtual machine. One thought might be "why not just use the cloud?". That's one possible solution to having to run virtual machines locally, but sometimes, I would prefer to not have to run things externally for a number of reasons. Docker solves the need to have a quick environment I can use to my liking and dispose of if I need to as well as commit along the way having any number of images representative of a point in time of my environment.

### Docker works well and has plugins for the varous automation tools

Because of [Docker][Docker]'s popularity, developers of automation tools have written plugins and functionality for [Docker][Docker]:

- [Ansible][Ansible] : [docker][docker_ansible] module, [docker_image][docker_image_ansible] module, [docker dynamic inventory plugin][docker_inventory_ansible]: http://docs.ansible.com/docker_image_module.html
- [SaltStack][SaltStack] : [salt.states.dockerio][salt_states_dockerio]
- [Puppet][SaltStack]: [gareth-docker][gareth-docker] module
- [Chef][Chef]: [Chef and Docker][chef_docker]


## Terminology: Containers and Images

Getting terminology correct is important in discussing technology. For me, I still have a slip of tongue and say "instance" when I mean "container" because my experience was working with virtual machines up until recently.

With [Docker][Docker], part of what makes it incredibly useful is the ability to easily create from running containers what are called [Docker][Docker] images. These are the read-only component, which can also be though of as templates for Docker containers, that when run, are a given container, the writeable component.

When using Docker, you start out with a base image which is your starting point in building derived images. An image you create from a running container, each time you commit, is comprised the "deltas", so to speak. An usage pattern would be that you might start out with a simple base image that contains a stock Linux environment. Then install a web server, along with your application. When the container is satisfactory representative of what comprises a running version of your application you would then commit that container as an image. The change that was committed, or what I previously mentioned as being thought of as a "delta", is given a unique 256-bit ID (64 hex characters). Subsequent commits are given their own IDs and Docker provides a way to view an image and see its history and have a hierarchical view of the image.

As previously stated, containers, since they only require the files specific to that applicaiton (code, libraries, etc) and are defined by specific processes versus an entire OS and its processes, require far less disk space and fewer resources, hence it's possible to have on disk many more images and run many more containers than VM technology would accomodate.

One can use the [Docker][Docker] CLI to search for any number of images to use as-is or to modify and build images from.

## Next installment: Installing and configuring Docker

The next blog post will cover installation and configuration of docker. Stay tuned, and do feel free to ask questions!
