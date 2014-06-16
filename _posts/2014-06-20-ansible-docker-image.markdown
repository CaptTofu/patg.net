---
layout: post
title: "Docker: Containers for the Masses -- Using Ansible to build Docker images"
date: 2014-06-20 12:00:00
categories: ansible,docker
---

Happy summer solstice! For the longest day of the year, welcome to a short blog post on using Ansible to automate the building of Docker images. This post is part of the blog series of posts, "Docker: Containers for the Masses".

For the reader just joining, the previous posts in this series are:

- [Introduction][docker_intro_blog] -- Introduction to [Docker][Docker]
- [Installation][docker_install_blog] -- Installation of [Docker][Docker]
- [Using Docker][docker_using_blog] -- Using [Docker][Docker]
- [Ansible and Docker][docker_ansible_blog] -- Using Ansible to manage [Docker][Docker]

Now that the basics of using Ansible to manage Docker have been introduced, particularly the Ansible [```docker```][ansible_docker_module] module, the next module to cover is the [```docker_image``` module][ansible_docker_image_module]. The [```docker_image``` module][ansible_docker_image_module] is an [Ansible][Ansible] module that uses the [Docker python client][docker-py] to connect to the [Docker][Docker] daemon (via a RESTful API) to build an images. This module, like any other module, can be creatively used in any playbook to build a [Docker][Docker] image that can be then run as a container.

## Building Docker Images with Ansible

Refering to the post in this blog series on [using Docker][docker_using_blog], examples showing how to build [Docker][Docker] images were given as well as covering the ```Dockerfile```. As the reader could see, there was a manual process to do this. A way of automating this is to use the Ansible [```docker_image``` module][ansible_docker_image_module].

Using this module is painfully simply. Well, not painful, but certainly simple!

A simple playbook:

    - hosts: local
      tasks:
        - name: build Apache image
          local_action:
              module: docker_image
              path: ../docker-image-source/ssh/
              name: CaptTofu/apache
              state: present

<br />
In the above example, as the YAML makes obvious, specifies using a ```local_action``` (since this is not something you typically run across multiple hosts), the name of the ```docker_image``` module, the ```path``` to where the docker file is, the ```name``` of the image and even more obvious the ```state```, in this case ```present``` which means "yes, please build this!". 

When the playbook is run, the following output would be expected:

    $ ansible-playbook -i hosts images.yml

    PLAY [local] ******************************************************************

    GATHERING FACTS ***************************************************************
    ok: [localhost]

    TASK: [build Apache image] ****************************************************
    changed: [localhost]

    PLAY RECAP ********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0    failed=0

<br />
And the build can be verified:

    $ docker images
    REPOSITORY                TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    CaptTofu/apache           latest              3be6decea409        9 minutes ago       297.5 MB

<br />
## A more interesting example

The above example was obviously very simple. A more interesting example can be found in the author's git repository at [https://github.com/CaptTofu/ansible-docker-presentation][ansible_docker_presentation_repo]. This repository was used for recent attendence at [AnsibleFest NYC 2014][AnsibleFest] to provide a practical example of using [Ansible][Ansible] to do a number of things including building four different [Docker][Docker] images and then running those images as [Docker][Docker] containers across 45 [HP Moonshot][moonshot] [cartridges][moonshot_cartridge].

The [repository][ansible_docker_presentation_repo] contains several roles including an ```image_builder``` role that has the following tasks:

    ---
    - name: check or build ssh image
      docker_image: path="{{ images_source_path }}/ssh/" name="{{ repository_name }}/ssh" state=present
    - name: check or build apache image
      docker_image: path="{{ images_source_path }}/apache/" name="{{ repository_name }}/apache" state=present
    - name: check or build haproxy image
      docker_image: path="{{ images_source_path }}/haproxy/" name="{{ repository_name }}/haproxy" state=present
    - name: check or build percona XtraDB Cluster image
      docker_image: path="{{ images_source_path }}/pxc/" name="{{ repository_name }}/pxc" state=present

<br />
The variables seen in this playbook task are set in the ```vars/main.yml``` file:

    ---
    images_source_path: /home/username/ansible_work/docker-image-source
    repository_name: capttofu

<br />
NOTE: one can overwrite these variables using ```-e EXTRA_VARS``` or long form ``--extra-vars=EXTRA_VARS```.

The location of the Docker image source files is also available in a repo, . As is shown, [https://github.com:CaptTofu/docker-image-source][docker_image_source]. The image source repo is checked out one directory up relative to where the [playbook repository][ansible_docker_presentation_repo] is located.

The playbook that is run that utilizes this role is very simple:

    ---
    - hosts: localhost
      roles:
      - image_builder

<br />
And when run:

    $ ansible-playbook -i hosts images.yml

    PLAY [localhost] **************************************************************

    GATHERING FACTS ***************************************************************
    ok: [localhost]

    TASK: [image_builder | check or build ssh image] ******************************
    changed: [localhost]

    TASK: [image_builder | check or build apache image] ***************************
    changed: [localhost]

    TASK: [image_builder | check or build haproxy image] **************************
    changed: [localhost]

    TASK: [image_builder | check or build percona XtraDB Cluster image] ***********
    changed: [localhost]

    PLAY RECAP ********************************************************************
    localhost                  : ok=5    changed=4    unreachable=0    failed=0

<br />
One still will need to manually push built images to a private or the [public Docker image registry][docker_image_hub]. The previous blog posts cover how to do this.

## Summary

This blog post demonstrated to the reader how to use Ansible to automate the building of Docker images using the [Ansible][Ansible] [```docker_image``` module][ansible_docker_image_module]. Starting with a simple example, and moving on to one more complex, the reader can gain a sense of how this can be used in an overall Ansible-based system to automate ensuring images that need to be run as containers will be available and if not, built for use. 

Stay tuned for the next blog post on how to use the [```docker_facts```][ansible_docker_facts_module] for gathering information and (other tricks) from a running fleet of containers.


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
[ansible_docker_presentation_repo]: https://github.com/CaptTofu/ansible-docker-presentation
[moonshot]: http://h17007.www1.hp.com/us/en/enterprise/servers/products/moonshot/index.aspx#.U0gU2PldXJh?jumpid=ps_r163&k_clickid=AMS|112|61913|169782eb-9c7a-df89-4afe-0000588f5dc0
[moonshot_cartridge]: http://www8.hp.com/us/en/products/proliant-servers/product-detail.html?oid=5375897#!tab=features
[docker_image_source]: https://github.com/CaptTofu/docker-image-source
[docker_ansible_blog]: http://patg.net/ansible,docker/2014/06/18/ansible-docker/
[docker_image_hub]: https://registry.hub.docker.com/
