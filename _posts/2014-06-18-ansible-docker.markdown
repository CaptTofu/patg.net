---
layout: post
title: "Docker: Containers for the Masses -- Ansible and Docker"
date: 2014-06-18 12:00:00
categories: ansible,docker
---

Welcome again to the series "Docker: Containers for the Masses". Today's post will be the fourth in this series and will concern itself with management of [Docker][Docker] using [Ansible][Ansible]

For the reader just joining, the previous three posts in this series are:

- [Introduction][docker_intro_blog] -- Introduction to [Docker][Docker]
- [Installation][docker_install_blog] -- Installation of [Docker][Docker] 
- [Using Docker][docker_using_blog] -- Using [Docker][Docker]

With the basics of Docker having been presented, the following blog posts will cover more advanced usage-cases for Docker as well as complimentary and related projects.

As mentioned before, I have had the pleasure of working with both [Docker][Docker] and [Ansible][Ansible] and realized there were some features in [Ansible][Ansible], particularly with [Docker][Docker], that I would both like to familiarize myself with and take advantage of, as well as contribute new features to. One of those features created, the [docker_facts][ansible_docker_facts_module] [Ansible][Ansible] module, will be covered in a later post of this series.

This first post will familiarize the reader to what Ansible is, it's modular design, and introduce a simple [Ansible playbook][ansible_playbooks].


## Ansible

There is a good chance that readers of this blog post know what [Ansible][Ansible] is and they can skip ahead two sections.

For readers who don't know, [Ansible][Ansible] is a tool in the same category as Puppet, Chef, and SaltStack, Ansible is an automation tool, more specifically an orchestration engine. It is very simple and unlike so many other automation tools that use a server and pull model, it simply manages nodes via ssh using a push model, not requiring a server or clients acting on behalf of the server. It is modular and modules simply output JSON that Ansible engine acts upon.

Ansible is written in python and uses what it calls *playbooks*, written in YAML, to represent how it expects the state of a system to be and specific tasks, called *plays*, that occur to get a node to be in the specified state. Like SaltStack, it uses Jinja templates for files it needs to dynamically create.

Ansible uses an inventory file that contains a list of the nodes which can be grouped into different classifications depending on what purpose you want for each -- think set-theory. Furthermore you can also use multiple inventory files or even [dynamic inventory plugins][ansible_dynamic_inventory] that take into account elastic environments where it would be useful to be able to manage resources intelligently.

## Ansible Basics

To give the reader who is not familiar with Ansible an idea and context for the rest of this post, a simple playbook shows how Ansible is used to install Percona XtraDB cluster packages.

    - name: Install Percona XtraDB Cluster server
      apt: pkg={{ item }}
           state=present
      with_items:
        - percona-xtradb-cluster-server-5.6
        - python-mysqldb
        - xinetd
        - git

    - name: Configure Percona XtraDB
      template: src=etc/mysql/my.cnf.j2
                dest=/etc/mysql/my.cnf
                mode=0640 owner=mysql group=root
      notify: restart mysql
      when: bootstrap_check.stdout == "bootstrap"

The above example is contains a snippet from a playbook consisting of two plays, the ```name``` describing what the play does.

In the case of the playbook above, the first play being run results in the 4 packages listed in ```with_items``` being installed  using the ```apt``` [Ansible module][ansible_apt_module], the ```state``` of these to be ```present```. [Ansible][Ansible] modules are idempotent, hence once these modules are installed, subsequent plays will result in no action.

The next play uses ```template``` to specify a source template ```my.cnf.j2``` to be rendered as ```/etc/mysql/my.cnf```, set to the file attributes and ownership listed by ```mode```, and ```owner``` and ```group```, respectively. When this play is completed, there will be ```/etc/mysql/my.cnf``` file ```present``` and any variables in the jinja template ```my.cnf.j2``` being interpolated in the template rendering. As mentioned above, [Ansible][Ansible] modules are idempotent, the result being that once the template is rendered and file created, it won't be re-rendered unless the template changes.

Again, this is a snippet, and there are other great examples available including an example [Ansible playbook git repository][ansible_example_playbook_repository] complete with numerous playbooks as well as the [documentation][ansible_documentation] that one can read to become an expert Ansible user.


## Ansible and Docker

As I've mentioned in other blog posts on this site, my team, HP Advanced Technology Group, has taken an interest in a number of interesting and burgeoning projects and/or technologies, both internal and external to HP. Two of those projects are [Ansible][Ansible] and [Docker][Docker], and both go together well.

This and the next few blog posts will cover several Docker modules and plugins:

- The [```docker``` module][ansible_docker_module].  One of the things one naturally would want to be able to do is have a means to manage docker containers. By management, this would mean launch , provision, and delete containers.
- The [```docker_images``` module][ansible_docker_image_module]. This module is used for building images specifying a [Dockerfile][dockerfile].
- The [```docker_facts``` module][ansible_docker_facts_module]. This module has not been released and was developed by the author of this blog. It's used for surfacing information about [Docker][Docker] images and containers that can be used in playbooks to do a number of tasks
- The [docker dynamic inventory plugin][ansible_docker_dynamic_inventory]. This plugin allows a user to be able to manage the very containers launched by [Ansible][Ansible] using Ansible to obtain from [Docker][Docker] a dynamic list that has varying membership (hence "dynamic").

e for this task, as well as a plugin for building dynamic inventory based off of running containers which will be covered in a subsequent blog post of this series.

Additionally, in my research, I found there were some things that I needed that were missing that I would take a stab at implementing, namely, an Ansible module for Docker that provides "facts" specific to Docker. A fact, in Ansible-speak, is information gathered by Ansible from nodes it manages. These facts are surfaced in playbooks as a dictionary that can be used both in the playbook in question or the rendedering of a Jinja template as a file on the node being managed. For Docker, this means a specific dictionary containing information organized by node for a fleet of containers as well as images. This post will also cover this feature.

## The Ansible ```docker``` module

The first module to be discussed is the very module one would use for launching or deleting container, the Ansible docker module. This module is part of the default [Ansible][Ansible] installation and will only require the [Docker python client library][docker-py] to be installed.

As you recall in using [Docker][Docker] by hand using the [Docker CLI][docker_cli], you would use the Docker CLI to do these types of tasks. Ansible makes this even more simple.

This is a very simple module to use and essentially a play describes itself, which really is the whole idea behind ansible:

    - name: docker image control
      local_action:
        module: docker
        docker_url: "tcp://somehost:4243"
        image: "capttofu/percona_xtradb"
        name: "db"
        state: "present"
        publish_all_ports: yes

In the above example, this play would result in running locally (```local_action```) the launching of a container with the name of ```db``` with all ports specified in image (Dockerfile) to be published. Furthermore, the parameter ```docker_url``` specifies which Docker daemon to issue this to, from the local action. This is important to note, because otherwise it would assume localhost. If one is manageing Docker containers across multiple servers running the Docker daemon. Under the hood, the Ansible docker module is using the Ansible client python module that talks to the daemon on the current host specified in ```docker_url```.

This makes for an interesting topography: managing systems (VM or bare metal) that run Docker and in turn, hopefully manageing those very containers, all with Ansible.

A more practical example is one that can be used to run multiple containers

    - hosts: localhost
      tasks:
        - name: run docker containers
          local_action:
            module: docker
            docker_url: tcp://127.0.0.1:4243
            image: capttofu/apache
            name: container_{{ item }}
            state: present
          with_sequence: count=5

The example above is a simple playbook that has one task: run 5 docker containers. This playbook uses ```local_action``` to run docker locally against the [Docker][Docker] daemon running on port 4243 -- which could be any docker host but in this case is simplified to 127.0.0.1 -- to connect to to run these containers. The image used is ```capttofu/apache```, a simple container that runs Apache. The name of the container passed utilizes the the variable ```item``` which is whatever value is being iterated over using the ineger sequence ```with_sequence```, in this example 1 through 5.

Running this playbook results in the follow output from Ansible:

    $ ansible-playbook -i hosts sitex.yml

    PLAY [localhost] **************************************************************

    GATHERING FACTS ***************************************************************
    ok: [localhost]

    TASK: [run docker containers] *************************************************
    changed: [localhost] => (item=1)
    changed: [localhost] => (item=2)
    changed: [localhost] => (item=3)
    changed: [localhost] => (item=4)
    changed: [localhost] => (item=5)

    PLAY RECAP ********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0    failed=0

Upon Ansible running this playbook, there should be 5 containers running:

    $ docker ps
    CONTAINER ID  IMAGE                   COMMAND               CREATED        STATUS        PORTS                    NAMES
    cd0401b8027e  capttofu/apache:latest  /usr/local/sbin/entr  3 seconds ago  Up 3 seconds  22/tcp, 80/tcp, 443/tcp  container_5
    aacf1df11a19  capttofu/apache:latest  /usr/local/sbin/entr  3 seconds ago  Up 3 seconds  22/tcp, 80/tcp, 443/tcp  container_4
    29925f68b768  capttofu/apache:latest  /usr/local/sbin/entr  4 seconds ago  Up 3 seconds  22/tcp, 80/tcp, 443/tcp  container_3
    125f6c6bbfc0  capttofu/apache:latest  /usr/local/sbin/entr  4 seconds ago  Up 4 seconds  22/tcp, 80/tcp, 443/tcp  container_2
    82d714bf1fa3  capttofu/apache:latest  /usr/local/sbin/entr  4 seconds ago  Up 4 seconds  22/tcp, 80/tcp, 443/tcp  container_1

This is a very simple example of using the [Ansible Docker plugin][ansible_docker_plugin] to introduce the reader to how simple it is to use Ansible for provisioning [Docker][Docker] containers. A more thorough and interesting example will be demonstrated on a later post that will revisit some of the material covered in the [presentation][ansible_docker_presentation] the author recently gave at [AnsibleFest NYC 2014][AnsibleFest].

## Summary 

This blog post introduced the reader to Ansible, describing the various Ansible Docker modules and plugins,  and provided a demonstration of the basics of using Ansible with Docker with the [Ansible Docker module][ansible_docker_module]. Subsequent blog posts will cover the [Ansible docker_image module][ansible_docker_image_module], [docker_facts module][ansible_docker_facts_module] and the [Ansible docker dynamic inventory plugin][ansible_docker_dynamic_inventory].

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
[docker-py]: https://github.com/dotcloud/docker-py
[docker_intro_blog]: http://patg.net/containers,virtualization,docker/2014/06/05/docker-intro/
[docker_install_blog]: http://patg.net/containers,virtualization,docker/2014/06/09/docker-install/
[docker_using_blog]: http://patg.net/containers,virtualization,docker/2014/06/10/using-docker/
[ansible_playbooks]: http://docs.ansible.com/playbooks.html
[ansible_example_playbook_repository]: https://github.com/ansible/ansible-examples
[docker_cli]: https://docs.docker.com/reference/commandline/cli/
[AnsibleFest]: http://www.marketwired.com/press-release/speakers-from-twitter-google-twilio-edx-rackspace-headline-ansiblefest-nyc-2014-1902858.htm
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
