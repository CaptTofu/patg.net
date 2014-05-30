---
layout: post
title: "Docker and Ansible: Container management made easy on a Moonshot"
date: 2014-06-02 12:00:00
categories: ansible,docker
---

# Managing a [Docker][Docker] fleet using [Ansible][Ansible]

Recently, I had the pleasure of working with both [Docker][Docker] and [Ansible][Ansible] and realized there were some features in [Ansible][Ansible], particularly with [Docker][Docker], that I would both like to familiarize myself with and take advantage of, as well as contribute new features to.


## What is [Docker][Docker]?

Most people reading this blog post would know, especially considering the growing popularity of the [Docker][Docker] project, but for the sake of completeness, [Docker][Docker], per the project itself, is a "an easy, lightweight virtualized environent for portable applications". To elaborate, [Docker][Docker] is an application the uses containers (as opposed to virtual machines). Specifically, its value is that is makes it incredibly easy to manage containers and make it possible to deploy applications in manageable units that can run anywhere. The days of having to deal with the hassle of application dependencies across various Linux variants or even different types of virtual machines or hardware are now made trivial. With Docker, you can build your application in a container that runs on your laptop or on a virtual machine and be assured that it can also run in any environment. This truley is revolutionary in terms of software development, testing and deployment.

[Docker][Docker] consists of (1) a daemon that is run that is interacted through a RESTful API using the (2) [Docker][Docker] command line interface. This CLI also includes the ability to search and "pull" images from both (3) [Docker][Docker] image repositories.

## What is the difference between containers and virtual machines?

The difference between virtual machines and containers is that a virtual machine requires some sort of emulation layer or hypervisor and would require a complete OS installation whereas a container uses features in the Linux kernel (3.x) to provide an isolated virtual environment-- disk, memory, networking, etc., on the same OS.

If you think about this, if you are only considering your application and the files it requires, and only the processes it runs when instantiated as a container, requires far less disk space and memory than traditional virtual machines.

The other thing that might come to mind for you is the question I often find asked: which of the two is better? Neither. It really depends what you are trying to do and each has its purpose. Sometimes, a virtual machine is needed to have a full, complete OS and associated functionalies while other times, in the case of [Docker][Docker], you want to have an isolated environment for developing an application and wish to ship it as a single unit. Those are two use cases for both that illustrate some of what each is suited for.

-- insert image used in slides for VM vs. container --


## Containers and Images

With [Docker][Docker], you have what are called docker images. These are the read-only component, the templates, that when run, are a container, the writeable component. These images contain the "deltas", so to speak, of what comparises a particular container. You might start out with a simple base image that contains a stock Linux environment. You install a web server, perhaps your application as well, then you commit that container as an image. The changes, or what I like to think of as a "delta", are what are each given a 256-bit ID (64 hex characters). These do not take up much space as you can imagine. They do refer to the base image ultimately, and any subsequent images that the current image was derived from.

One can use the [Docker][Docker] CLI to search for any number of images to use as-is or to modify and build images from.

## Ansible

Another thing assumed in this blog post is that most readers should know what Ansible is. In the case not, In the same category as Puppet, Chef, and SaltStack, Ansible is an automation tool, more specifically an orchestration engine. It is very simple and unlike so many other automation tools that use a server and pull model, it simply manages nodes via ssh using a push model, not requiring a server or clients acting on behalf of the server. It is modular and modules simply output JSON that Ansible engine acts upon. Ansible is written in python and uses what it calls "playbooks", written in YAML, to represent how it expects the state of a system to be and specific tasks, called "plays", that occur to get a node to be in the specified state. Like SaltStack, it uses Jinja templates for files it needs to dynamically create.

Ansible uses an inventory file that contains a list of the nodes which can be grouped into different classifications depending on what purpose you want for each -- think set-theory. Furthermore you can also use multiple inventory files or even dynamic inventory plugins that take into account elastic environments where it would be useful to be able to manage resources intelligently.

## Ansible and Docker, on a Moonshot

As I've mentioned in other blog posts on this site, my team, HP Advanced Technology Group, has taken an interest in a number of interesting and burgeoning projects and/or technologies, both internal and external to HP. Two of those projects are [Ansible][Ansible] and [Docker][Docker], and both go together well.

Additionally, we have some excellent technology within HP to take advantage of: the HP Moonshot. The Moonshot System is a revolutionary low-power, state-of-the art chassis that holds server cartridges designed for specific workloads (“system on a chip”). This blog post will detail how Ansible and Docker were used to run and manage numerous [Docker][Docker] containers across 45 cartridges.

One of the things one naturally would want to be able to do is have a means to manage docker containers. By management, this would mean launch , provision, and delete containers, as well as creating images from running containers. Luckily for Ansible, there was an module already available for this task, as well as a plugin for building dynamic inventory based off of running containers, something this blog post will feature.

Additionally, in my research, I found there were some things that I needed that were missing that I would take a stab at implementing, namely, an Ansible module for Docker that provides "facts" specific to Docker. A fact, in Ansible-speak, is information gathered by Ansible from nodes it manages. These facts are surfaced in playbooks as a dictionary that can be used both in the playbook in question or the rendedering of a Jinja template as a file on the node being managed. For Docker, this means a specific dictionary containing information organized by node for a fleet of containers as well as images. This post will also cover this feature.

## The docker Ansible module

The first module to be discussed is the very module one would use for launching or deleting container, the Ansible docker module. 

Normally, outside of ansible, you would use the Docker CLI to do these types of tasks. Ansible makes this even more simple.

This is a very simple module to use and essentially a play describes itself, which really is the whole idea behind ansible:

- name: docker image control
  local_action:
    module: docker
    docker_url: "tcp://somehost:4243"
    image: "capttofu/percona_xtradb"
    name: "db"
    state: "present"
    publish_all_ports: yes

In the above example, this play would result in running locally (local_action) the launching of a container with the name of "db" with all ports specified in image (Dockerfile) to be published. Furthermore, the parameter docker_url specifies which Docker daemon to issue this to, from the local action. This is important to note, because otherwise it would assume localhost. If one is manageing Docker containers across multiple servers running the Docker daemon. Under the hood, the Ansible docker module is using the Ansible client python module that talks to the daemon on the current host specified in docker_url.

This makes for an interesting topography: managing systems (VM or bare metal) that run Docker and in turn, hopefully manageing those very containers, all with Ansible.

## The docker_images module

The other thing that users of [Docker][Docker] want to do is to create images from running containers after modifying containers to their specifications. 



