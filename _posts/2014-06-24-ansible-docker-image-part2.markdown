---
layout: post
title: "Docker: Containers for the Masses -- Yet another way to use Ansible to build Docker images"
date: 2014-06-24 12:00:00
categories: ansible,docker,mysql,galera,clustering
---

Good Monday morning! Over the weekend, there was a comment to my previous post covering [using Ansible to build Docker images][docker_ansible_images_blog] from [Michael DeHaan][michael_dehaan], CTO and creator of [Ansible][Ansible] (thank you!) reminding me of his work discussed in his blog post [Installing and Building Docker with Ansible][building_docker_with_ansible] that is definitely worth sharing and the method I first used to build Docker images and wrote [my first role][galera_role] that will be shared in this post.

For the reader just joining, the previous posts in this series "Docker: Containers for the Masses" are:

- [Introduction][docker_intro_blog] -- Introduction to [Docker][Docker]
- [Installation][docker_install_blog] -- Installation of [Docker][Docker]
- [Using Docker][docker_using_blog] -- Using [Docker][Docker]
- [Ansible and Docker][docker_ansible_blog] -- Using Ansible to manage [Docker][Docker]
- [Building Docker Images using Ansible][docker_ansible_images_blog] -- Using Ansible to build [Docker][Docker] images


<br />
## Building Docker Images with Ansible using a Dockerfile

[Michael's article][building_docker_with_ansible] details how to install Ansible, how to use [Paul Durivage's][paul_durivage] [angstwad.dcoker_ubuntu Ansible role][angstwad_repo], also features an important-to-know way of building Docker images whereby an image is build using a Dockerfile that specifies the installation of [Ansible][Ansible], checks out your playbook repository which it then runs with Ansible resulting in a built image with everything you would want on that image. This is different than in the previous post that details using Ansible to run the [Docker][Docker] image-building process and is yet another example on how using [Ansible][Ansible] and [Docker][Docker] together is flexible and the approach to both interchangeable and each method equally valid depending on what the user requires.

Additionally, I used this methodology when I first started using [Ansible][Ansible] and [Docker][Docker] and forked Michael's repository, adding a [Galera role][galera_role].

## The Playbook

In addition to showing yet another way to build [Docker][Docker] images, this post will also give the reader more insight into using [Ansible][Ansible] in general and show another example of what one can do with a [Dockerfile][Dockerfile].

This post will detail a playbook I wrote when I forked the [docker_dna repo][docker_dna_repo]. In my role with the HP ATG Group, I was tasked with researching Ansible and Docker and wanted to accomplish several things: Learn Docker and Ansible as well as see if my experience -- and a [Salt][saltstack] template and methodology for setting up a Galera cluster could be easily ported to [Ansible].


### Directory layout

The repo, when cloned, there are  the ```base```, ```rabbitmq```, ```zookeeper```, and ```galera``` subdirectories. The last one was added by myself when I used this repo to get familiar with this methodology for building [Docker][Docker] images. In that subdirectory

    $ ls -1
    dna.yml
    docker-dna_galera.yml
    Dockerfile
    group_vars
    host_vars
    README.md
    roles

<br />

### Top-level playbook

The ```dna.yml``` playbook sets some variables and includes ```docker-dns_galera.yml```:

    ---
    # file: dna.yml
    - include: docker-dna_galera.yml

<br />

### Tasks

The tasks that then are used for this role which are broken up into specific operations:

    $ ls -1 roles/docker-dna_galera/tasks/
    clustercheck.yml
    configure_galera.yml
    grants.yml
    install_galera.yml
    main.yml
    misc.yml
    repo.yml

<br />

### Specifying using the ```docker-dna_galera``` role
 
```docker-dna_galera.yml``` in turn uses the roles ```common``` and ```docker-dns_galera```

    ---
    # file: docker-dna_galera.yml
    - hosts: docker-dna_galera
      roles:
          - common
          - docker-dna_galera

<br />

### Role variables

By using the ```docker-dns_galera``` role, the role's variables are set in the file ```roles/docker-dna_galera/vars/main.yml``` which contains variables used by the the templates  ```roles/docker-dna_galera/templates/etc/mysql/my.cnf.j2``` and ```roles/docker-dna_galera/templates/usr/bin/clustercheck.j2```, as well as some of the role's tasks.

    ---
    # file: roles/docker-dna_galera/vars/main.yml
    # these values are default - change for security!
    galera:
        dbusers:
            xtrabackup:
                username: xtrabackup
                password: xtrabackup
            docker:
                username: docker
                password: docker
                host: 172.17.%
            clustercheck:
                username: clustercheck
                password: clustercheck

<br />

### Top-level playbook including tasks

```main.yml``` includes each task in the order it needs to be run:

    --
    # file: roles/docker-dna_percona/tasks/main.yml

    - include: misc.yml
    - include: repo.yml
    - include: install_galera.yml
    - include: grants.yml
    - include: configure_galera.yml
    - include: clustercheck.yml

<br />

### Misc playbook

The first task ```misc.yml``` installs vim or any other package other than the [Percona][percona] packages:

{% raw %}
```
---
# file: roles/docker-dna_percona/tasks/misc.yml

- name: Install things I like
  apt: pkg={{ item }}
       state=present
  with_items:
      - vim
```
{% endraw %}
<br />

### Set the repo

The ```repo.yml``` task simply sets up apt to use the [Percona][percona] apt repo:

    ---
    # file: roles/docker-dna_galera/tasks/repo.yml

    - name: Obtain Percona public key
      # apt_key: url=http://keys.gnupg.net/pks/lookup?op=get&search=0x1C4CBDCDCD2EFD2A
      apt_key: url=http://www.percona.com/downloads/RPM-GPG-KEY-percona
               state=present

    - name: Add Percona repository
      apt_repository: repo='deb http://repo.percona.com/apt precise main'
                      state=present

    - name: Add Percona source repository
      apt_repository: repo='deb-src http://repo.percona.com/apt precise main'
                      state=present

    - name: Update apt cache
      apt: update_cache=yes

<br />

### Install the database software

The ```install_galera.yml``` task installs [Percona XtraDB Cluster][percona_xtradb_cluster] as well as copying a startup script into /usr/local/bin. This is somewhat historic as upstart didn't work with older versions of [Docker][Docker]

{% raw %}

    ---

    - name: Install Percona XtraDB Cluster server
      apt: pkg={{ item }}
           state=present
      with_items:
          - percona-xtradb-cluster-server-5.6
          - python-mysqldb
          - xinetd
          - telnet

    - name: Copy the helper script
      copy: src=usr/local/bin/mysql_run.sh
            dest=/usr/local/bin/mysql_run.sh
            mode=0755
{% endraw %}

<br />

### Set the database grants

The ```grants.yml``` task sets the grants for the database that are needed to run a successful Galera cluster

{% raw %}
    ---
    # file: roles/docker-dna_percona/tasks/grants.yml

    - name: Add Docker database user
      mysql_user: user={{ galera['dbusers']['docker']['username'] }} host={{ galera['dbusers']['docker']['host'] }}  password={{ galera['dbusers']['docker']['password'] }} priv=*.*:"all privileges"

    - name: Add xtrabackup database user (for Galera SST)
      mysql_user: user={{ galera['dbusers']['xtrabackup']['username'] }} host="localhost" password={{ galera['dbusers']['xtrabackup']['password'] }} priv=*.*:"grant, reload, replication client"

    - name: Add clustercheck database user (for clustercheck/xinetd -> haproxy)
      mysql_user: user={{ galera['dbusers']['clustercheck']['username'] }} host="localhost" password={{ galera['dbusers']['clustercheck']['password'] }} priv=*.*:"grant, reload, replication client"
{% endraw %}

<br />

### Configure the database

```configure_galera.yml``` generates ```/etc/mysql/my.cnf``` and shuts down the ```mysqld``` process. Why shut it down? Because the container this is running on is only for building the image and just as when creating a snapshot, it makes more sense to not have a running database with open file-handles that an image is created from. 

    ---
    # file: roles/docker-dna_percona/tasks/configure_galera.yml

    - name: Configure Percona XtraDB Cluster server
      template: src=etc/mysql/my.cnf.j2
                dest=/etc/mysql/my.cnf

    - name: Stop MySQL
      action: service name=mysql state=stopped

<br />

### Set up the clustercheck script for [HAProxy][haproxy]

The last task, ```clustercheck.yml```, sets up the python script used by [HAProxy][haproxy] to determine which master to use. Why not the original xinetd-based clustercheck script? The author was never able to get the xinetd-based clustercheck script working with Docker.

    # file: roles/docker-dna_percona/tasks/clustercheck.yml

    - name: Copy clustercheck script
      copy: src=usr/local/bin/pyclustercheck
            dest=/usr/local/bin/pyclustercheck
            owner=root
            group=root
            mode=0700

<br />

### Template generation

The templates for the ```docker-dna_percona``` role are the ```my.cnf.j2``` jinja template which is generated as ```/etc/mysql/my.cnf``` and transliterates the variables set in the previously-mentioned variables file. This snippet shows the Galera-specific mysql options. The cluster address is set to bootstrap. Remember that this is an image that is being built. One would need to use [Ansible][Ansible] to configure this value to reflect node membership state of the cluster when the containers are run that use this image as well as set different passwords.

    wsrep_provider          = /usr/lib/libgalera_smm.so
    wsrep_slave_threads     = 4
    wsrep_sst_method        = xtrabackup
    wsrep_sst_auth          = {{ galera['dbusers']['xtrabackup']['username'] }}:{{ galera['dbusers']['xtrabackup']['password'] }}
    wsrep_cluster_name      = percona-cluster
    wsrep_cluster_address   = gcomm://
    wsrep_provider_options  = gcache.size=2G;

<br />
## Dockerfile goodness

Finally, the [Dockerfile][Dockerfile]! This is where all the work happens.

    # docker-dna/galera/Dockerfile
    #
    # VERSION    0.1.0
    #
    FROM capttofu/docker-dna_base
    MAINTAINER Patrick aka CaptTofu Galbraith , patg@patg.net

    # Update distribution
    RUN apt-get update \
          && apt-get upgrade -y \
          && apt-get clean

    # Add files
    ADD . ./DockerDNA

    # Install Percona XtraDB Cluster 
    RUN ( echo '[docker-dna_galera]' && \
          echo 'localhost' \
        ) > /etc/ansible/hosts \
          && ansible-playbook ./DockerDNA/dna.yml --connection=local \
          && apt-get clean

    # Expose MySQL/Galera
    EXPOSE 3306 4444 4567 4568 9200

    ENTRYPOINT ["/usr/local/bin/mysql_run.sh"]

The above [Dockerfile][Dockerfile] specifies using the ```capttofu/docker-dna_base``` image as a base. This image already has ansible and it's prerequisite libraries pre-installed and ready to use. The first event that is run in the Dockerfile is to update the apt system. Next, everything in the current repository is copied to a ```DockerDNA``` directory in the root directory of the temporary container.

## Building the image

Next, by running ```docker build .``` in the same directory, the image will be built, using the pre-installed ansible, run with a l;ocal connection, in this case.

    docker-dna-galera/galera$ docker build .
    Uploading context 49.15 kB
    Uploading context
    Step 0 : FROM capttofu/docker-dna_base
    Pulling repository capttofu/docker-dna_base
    1e47da3640f1: Download complete
    80cd3d2446e3: Download complete
    < ... snip -- numerous lines like the above and below ... >
    5607ff993e85: Download complete
    27e469823903: Download complete
     ---> 167dc428d943
    Step 1 : MAINTAINER Patrick aka CaptTofu Galbraith , patg@patg.net
     ---> Running in ceb1ec12aab7
     ---> c7916cff77cd
    Removing intermediate container ceb1ec12aab7
    Step 2 : RUN apt-get update      && apt-get upgrade -y      && apt-get clean
     ---> Running in d5b1189506ec
    Get:1 http://security.ubuntu.com precise-security Release.gpg [198 B]
    Get:2 http://ppa.launchpad.net precise Release.gpg [316 B]
    < ... snip ...> 
    Hit http://archive.ubuntu.com precise/main Translation-en
    Hit http://archive.ubuntu.com precise/universe Translation-en
    Get:20 http://archive.ubuntu.com precise-updates/main Translation-en [431 kB]
    Get:21 http://archive.ubuntu.com precise-updates/universe Translation-en [180 kB]
    Fetched 4734 kB in 20s (228 kB/s)
    Reading package lists...
    Reading package lists...
    Building dependency tree...
    Reading state information...
    The following packages have been kept back:
      ansible initscripts upstart
    The following packages will be upgraded:
      apt apt-utils base-files ca-certificates curl dpkg file gnupg gpgv ifupdown
      initramfs-tools initramfs-tools-bin iproute libapt-inst1.4 libapt-pkg4.12
      libc-bin libc6 libcurl3 libcurl3-gnutls libdrm-intel1 libdrm-nouveau1a
      libdrm-radeon1 libdrm2 libgnutls26 libmagic1 libssl1.0.0 libudev0
      libyaml-0-2 multiarch-support openssh-client openssl perl-base procps
      python-apt python-apt-common python-software-properties python2.7
      python2.7-minimal tzdata udev
    40 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
    Need to get 23.0 MB of archives.
    After this operation, 14.3 kB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu/ precise-updates/main base-files amd64 6.5ubuntu6.7 [61.0 kB]
    Get:2 http://archive.ubuntu.com/ubuntu/ precise-updates/main dpkg amd64 1.16.1.2ubuntu7.5 [1829 kB]
    < ... snip ... >
    Get:40 http://archive.ubuntu.com/ubuntu/ precise-updates/main python-software-properties all 0.82.7.7 [23.5 kB]
    debconf: unable to initialize frontend: Dialog
    debconf: (TERM is not set, so the dialog frontend is not usable.)
    debconf: falling back to frontend: Readline
    Installing new version of config file /etc/issue.net ...
    < ... snip - lots of apt output ...>
    ldconfig deferred processing now taking place
    Processing triggers for initramfs-tools ...
     ---> 0a8159128717
    Removing intermediate container d5b1189506ec
    Step 3 : ADD . ./DockerDNA
     ---> 93766d0fc5b2
    Removing intermediate container f4b216a8ddbf
    Step 4 : RUN ( echo '[docker-dna_galera]' &&      echo 'localhost'    ) > /etc/ansible/hosts      && ansible-playbook ./DockerDNA/dna.yml --connection=local      && apt-get clean
     ---> Running in e41cd0dba011

    PLAY [docker-dna_galera] ******************************************************

    GATHERING FACTS ***************************************************************
    ok: [localhost]

    TASK: [Install things I like] *************************************************
    changed: [localhost] => (item=vim)

    TASK: [Obtain Percona public key] *********************************************
    changed: [localhost]

    TASK: [Add Percona repository] ************************************************
    changed: [localhost]

    TASK: [Add Percona source repository] *****************************************
    changed: [localhost]

    TASK: [Update apt cache] ******************************************************
    ok: [localhost]

    TASK: [Install Percona XtraDB Cluster server] *********************************
    changed: [localhost] => (item=percona-xtradb-cluster-server-5.6,python-mysqldb,xinetd,telnet)

    TASK: [Copy the helper script] ************************************************
    changed: [localhost]

    TASK: [Add Docker database user] **********************************************
    changed: [localhost]

    TASK: [Add xtrabackup database user (for Galera SST)] *************************
    changed: [localhost]

    TASK: [Add clustercheck database user (for clustercheck/xinetd -> haproxy)] ***
    changed: [localhost]

    TASK: [Configure Percona XtraDB Cluster server] *******************************
    changed: [localhost]

    TASK: [Stop MySQL] ************************************************************
    ok: [localhost]

    TASK: [Copy clustercheck script] **********************************************
    changed: [localhost]

    TASK: [Copy clustercheck script] **********************************************
    changed: [localhost]

<br />

## Verifying image

When this has completed, the image, ```capttofu/docker-dna_base```, will be ready to use, in this case a container running [Percona XtraDB Cluster][percona_xtradb_cluster] that will need to be managed by Ansible in order to set up the galera cluster.

    $ docker images
    REPOSITORY                 TAG      IMAGE ID      CREATED       VIRTUAL SIZE
    <none>                     <none>   cba0a737d934  14 hours ago  850.7 MB
    ubuntu                     12.10    e314931015bd  13 days ago   172.2 MB
    ubuntu                     quantal  e314931015bd  13 days ago   172.2 MB
    ubuntu                     13.10    145762641db9  13 days ago   180.2 MB
    ubuntu                     saucy    145762641db9  13 days ago   180.2 MB
    ubuntu                     14.04    ad892dd21d60  13 days ago   275.5 MB
    ubuntu                     latest   ad892dd21d60  13 days ago   275.5 MB
    < ... snip ... >
    capttofu/docker-dna_base   latest   167dc428d943  4 months ago  350.6 MB
    capttofu/docker-dna_base   0.1.0    1e47da3640f1  4 months ago  798.7 MB
    capttofu/docker-dna_base   12.04.w1 5150446a5dd3  4 months ago  350.6 MB


<br />

## Summary

This blog post showed the reader yet another way to use [Docker][Docker] and [Ansible][Ansible] together to build Docker images by using a [Dockerfile][Dockerfile] to run Ansible to install packages and configure the temporary container that is being used to build the image. This provides yet another example of the flexibility of these two great applications and gives the user yet another method in their toolbox of solutions. Another side-benefit of this article was also learning how to install [Percona XtraDB Cluster][percona_xtradb_cluster] with [Ansible][Ansible].


[Docker]: http://docker.io
[Dockerfile]: http://docs.docker.com/reference/builder/
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
[docker_ansible_images_blog]: http://patg.net/ansible,docker/2014/06/20/ansible-docker-image/ 
[docker_image_hub]: https://registry.hub.docker.com/
[building_docker_with_ansible]: http://www.ansible.com/blog/2014/02/12/installing-and-building-docker-with-ansible
[haproxy]: http://www.haproxy.org/
[angstwad_repo]: https://github.com/angstwad/docker.ubuntu
[paul_durivage]: https://github.com/angstwad
[michael_dehaan]: http://michaeldehaan.net/
[docker_dna_repo]: https://github.com/wrale/docker-dna
[galera]: http://galeracluster.com/
[percona_xtradb_cluster]: http://www.percona.com/software/percona-xtradb-cluster
[percona]: http://www.percona.com
[galera_role]: https://github.com/CaptTofu/docker-dna/tree/master/galera
